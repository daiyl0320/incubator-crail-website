---
layout: default
title: Documentation 
---

The Crail I/O stack consists of a set of components. Typically only a subset of the components are required for a particular use case (e.g., Spark, Hadoop, Hive, etc.) or hardware setup (e.g., RDMA, TCP, Flash, etc.). Here is a list of the components together with their GitHub repository. 

* <a href="{{ site.base }}/community/">Crail Store</a>: The backbone for all I/O operations across distributed storage resource. Includes both the RDMA/DRAM and the NVMf/Flash storage tier.
* [Crail-Blkdev](https://github.com/zrlio/crail-blkdev): A Crail storage tier for shared volume storage.
* [Crail-Netty](https://github.com/zrlio/crail-netty): A Crail TCP/DRAM storage tier built on top of Netty.
* [Crail-Spark-IO](https://github.com/zrlio/crail-spark-io): A module including Crail-based Shuffle and Broadcast plugins for Spark.
* [Crail-Spark-TeraSort](https://github.com/zrlio/crail-terasort): Currently only the sorting benchmark is available.

We currently do not provide binary releases. This page describes how to build the Crail I/O stack from source, and how to configure and deploy it. 

<h2 id="crail">Building Crail Store</h2>

Building the source requires [Apache Maven](http://maven.apache.org/) and Java version 8 or higher.
To build Crail execute the following steps:

1. Obtain a copy of <a href="{{ site.base }}/community/">Crail Store</a>
2. Make sure your local maven repo contains [DiSNI](https://github.com/zrlio/disni), if not build DiSNI from Github
3. Make sure your local maven repo contains [DaRPC](https://github.com/zrlio/darpc), if not build DaRPC from Github
4. Run: mvn -DskipTests install
5. Copy tarball to the cluster and unpack it using tar xvfz crail-1.0-bin.tar.gz

Note: later, when deploying Crail, make sure libdisni.so is part of your LD_LIBRARY_PATH. The easiest way to make it work is to copy libdisni.so into crail-1.0/lib 

### Configuration

To configure Crail use crail-site.conf.template as a basis and modify it to match your environment. 

    cd crail-1.0/conf
    mv crail-site.conf.template crail-site.conf
  
There are a general file system properties and specific properties for the different storage tiers. A typical configuration for the general file system section may look as follows:

    crail.namenode.address                crail://namenode:9060
    crail.storage.types                   org.apache.crail.storage.rdma.RdmaStorageTier
    crail.cachepath                       /memory/cache
    crail.cachelimit                      12884901888
    crail.blocksize                       1048576
    crail.buffersize                      1048576

In this configuration the namenode is configured to run using port 9060 on host 'namenode', which must be a valid host in the cluster. We further configure a single storage tier, in this case the RDMA-based DRAM tier. Cachepath points to a directory that is used by the file system to allocate memory for the client cache. Up to cachelimit size, all the memory that is used by Crail will be allocated via mmap from this location. Ideally, the directory specified in cachepath points to a hugetlbfs mountpoint. Aside from the general properties, each storage tier needs to be configured separately.

#### RDMA/DRAM Storage Tier

For the RDMA/DRAM tier we need to specify the interface that should be used by the storage nodes.

    crail.storage.rdma.interface          eth0
  
The datapath property specifies a path from which the storage nodes will allocate blocks of memory via mmap. Again, that path best points to a hugetlbfs mountpoint.

    crail.storage.rdma.datapath           /memory/data

You want to specify how much DRAM each datanode should donate into the file system pool using the `storagelimit` property. DRAM is allocated in chunks of `allocationsize`, which needs to be a multiple of `crail.blocksize`.

    crail.storage.rdma.allocationsize     1073741824
    crail.storage.rdma.storagelimit       75161927680

Crail supports optimized local operations via memcpy (instead of RDMA) in case a given file operation is backed by a local storage node. The indexpath specifies where Crail will store the necessary metadata that make these optimizations possible. Important: the indexpath must NOT point to a hugetlbfs mountpoint because index files will be updated which not possible in hugetlbfs.

    crail.storage.rdma.localmap           true
    crail.storage.rdma.indexpath          /index
    
#### NVMf/Flash Storage Tier

Crail is a multi-tiered storage system. Additinoal tiers can be enabled by adding them to the configuration as follows.

    crail.storage.types                  org.apache.crail.storage.rdma.RdmaStorageTier,org.apache.crail.storage.nvmf.NvmfStorageTier


For the NVMf storage tier we need to configure the server IP that is used when listening for new connections. We also need to configure the PCI address of the flash device we want to use, as well as the huge page mount point to be used for allocating memory. 

    crail.storage.nvmf.bindip		10.40.0.XX
    crail.storage.nvmf.pcieaddr		0000:11:00.0
    crail.storage.nvmf.hugedir		/dev/hugepages
    crail.storage.nvmf.socketmem		512,512

### Deployment

For all deployments, make sure you define CRAIL_HOME on each machine to point to the top level Crail directory.

#### Starting Crail manually

The simplest way to run Crail is to start it manually on just a handful nodes. You will need to start the Crail namenode, plus at least one datanode. To start the namenode execute the following command on the host that is configured to be the namenode:

    cd crail-1.0/
    ./bin/crail namenode

To start a datanode run the following command on a host in the cluster (ideally this is a different physical machine than the one running the namenode):

    ./bin/crail datanode

Now you should have a small deployment up with just one datanode. In this case the datanode is of type RDMA/DRAM, which is the default datnode. If you want to start a different storage tier you can do so by passing a specific datanode class as follows:

    ./bin/crail datanode -t org.apache.crail.storage.nvmf.NvmfStorageTier

This would start the shared storage datanode. Note that configuration in crail-site.conf needs to have the specific properties set of this type of datanode, in order for this to work. 

#### Larger deployments

To run larger deployments start Crail using 

    ./bin/start-crail.sh

Similarly, Crail can be stopped by using 

    ./bin/stop-crail.sh

For this to work include the list of machines to start datanodes in conf/slaves. You can start multiple datanode of different types on the same host as follows:

    host02-ib
    host02-ib -t org.apache.crail.storage.nvmf.NvmfStorageTier
    host03-ib

In this example, we are configuring a Crail cluster with 2 physical hosts but 3 datanodes and two different storage tiers.

### Crail Shell

Crail provides an contains an HDFS adaptor, thus, you can interact with Crail using the HDFS shell:

    ./bin/crail fs

Crail, however, does not implement the full HDFS shell functionality. The basic commands to copy file to/from Crail, or to move and delete files, will work.

    ./bin/crail fs -mkdir /test
    ./bin/crail fs -ls /
    ./bin/crail fs -copyFromLocal <path-to-local-file> /test
    ./bin/crail fs -cat /test/<file-name>

For the Crail shell to work properly, the HDFS configuration in crail-1.0/conf/core-site.xml needs to be configured accordingly:

    <configuration>
      <property>
       <name>fs.crail.impl</name>
       <value>org.apache.crail.hdfs.CrailHadoopFileSystem</value>
      </property>
      <property>
        <name>fs.defaultFS</name>
        <value>crail://namenode:9060</value>
      </property>
      <property>
        <name>fs.AbstractFileSystem.crail.impl</name>
        <value>org.apache.crail.hdfs.CrailHDFS</value>
      </property>
     </configuration>

Note that the Crail HDFS interface currently cannot provide the full performance of Crail due to limitations of the HDFS API. In particular, the HDFS `FSDataOutputStream` API only support heap-based `byte[]` arrays which requires a data copy. Moreover, HDFS operations are synchronous preventing efficient pipelining of operations. Instead, applications that seek the best performance should use the Crail interface directly, as shown next.

### Programming against Crail

The best way to program against Crail is to use Maven. Make sure you have the Crail dependency specified in your application pom.xml file:

    <dependency>
      <groupId>org.apache.crail</groupId>
      <artifactId>crail-client</artifactId>
      <version>1.0</version>
    </dependency>

Then, create a Crail file system instance as follows:

    CrailConfiguration conf = new CrailConfiguration();
    CrailFS fs = CrailFS.newInstance(conf);

Make sure the crail-1.0/conf directory is part of the classpath. 

The simplest way to create a file in Crail is as follows:

    CrailFile file = fs.create(filename, CrailNodeType.DATAFILE, CrailStorageClass.DEFAULT, CrailLocationClass.DEFAULT).get().syncDir();

Aside from the actual filename, the 'create()' call takes as input the storage and location classes which are preferences for the storage tier and physical location that this file should be created in. Crail tries to satisfy these preferences later when the file is written. In the example we do not request any particular storage or location affinity.

The 'create()' call is non-blocking, calling 'get()' on the returning future object awaits the completion of the call. At that time, the file has been created, but its directory entry may not be visible. Therefore, the file may not yet show up in a file enumeration of the given parent directory. Calling 'syncDir()' waits to for the directory entry to be completed. Both the 'get()' and the 'syncDir()' operation can be deffered to a later time at which they may become non-blocking operations. 

Once the file is created, a file stream can be obtained for writing:

    CrailBufferedOutputStream outstream = file.getBufferedOutputStream(1024);	

Here, we create a buffered stream so that we can pass heap byte arrays as well. We could also create a non-buffered stream using

    CrailOutputStream outstream = file.getDirectOutputStream(1024);

In both cases, we pass a write hint (1024 in the example) that indicates to Crail how much data we are intending to write. This allows Crail to optimize metadatanode lookups. Crail never prefetches data, but it may fetch the metadata of the very next operation concurrently with the current data operation if the write hint allows to do so. 

Once the stream has been obtained, there exist various ways to write a file. The code snippet below shows the use of the asynchronous interface:

    ByteBuffer dataBuf = fs.allocateBuffer();
    Future<DataResult> future = outputStream.write(dataBuf);
    ...
    future.get();

Reading files works very similar to writing. There exist various examples in org.apache.crail.tools.CrailBenchmark.

### Storage Tiers

Crail ships with the RDMA/DRAM storage tier. Currently there are two additional storage tiers available in separate repos:

* [Crail-Blkdev](https://github.com/zrlio/crail-blkdev)  is a storage tier integrating shared volume block devices such as disaggregated flash. 
* [Crail-Netty](https://github.com/zrlio/crail-netty) is a DRAM storage tier for Crail that uses TCP, you can use it to run Crail on non-RDMA hardware. Follow the instructions in these repos to build, deploy and use these storage tiers in your Crail environmnet. 

### Benchmarks

Crail provides a set of benchmark tools to measure the performance. Type

    ./bin/crail iobench

to get an overview of the available benchmarks. For instance, to benchmark the sequential write performance, type

    ./bin/crail iobench -t write -s 1048576 -k 102400 -f /tmp.dat

This will create a file of size 100G, written sequentially in a sequence of 1MB operations. 

To read a file sequentially, type

    ./bin/crail iobench -t read -s 1048576 -k 102400 -f /tmp.dat

This command issues 102400 read operations of 1MB each.

The tool also contains benchmarks to read files randomly, or to measure the performance of opening files, etc.

<h2 id="spark">Building Crail Spark Modules</h2>

Building the source requires [Apache Maven](http://maven.apache.org/) and Java version 8 or higher.
To build Crail execute the following steps:

1. Obtain a copy of [Crail-Spark-IO](https://github.com/zrlio/crail-spark-io) from Github
2. Make sure your local maven repo contains crail store jars, if not build Crail from the <a href="{{ site.base }}/community/">source</a> 
4. Run: mvn -DskipTests install
5. Add crail-spark-1.0.jar as well as its Crail dependencies to the Spark extra class path, both for the driver and the executors

```
spark.driver.extraClassPath     $CRAIL_HOME/jars/*:<path>/crail-spark.jar:.
spark.executor.extraClassPath   $CRAIL_HOME/jars/*:<path>/crail-spark.jar:.
```

### Configuration

To configure the crail shuffle plugin included in spark-io add the following line to spark-defaults.conf
```
spark.shuffle.manager		org.apache.spark.shuffle.crail.CrailShuffleManager
```
Since spark version 2.0.0, broadcast is no longer an exchangeable plugin, unfortunately. To use the crail broadcast plugin in Spark it has to be manually added to Spark's BroadcastManager.scala.

### Running

For the Crail shuffler to perform best, applications are encouraged to provide an implementation of the `CrailShuffleSerializer` interface, as well as an implementation of the `CrailShuffleSorter` interface. Defining its own custom serializer and sorter for the shuffle phase not only allows the application to serialize and sort faster, but allows applications to directly leverage the functionality provided by the Crail input/output streams such as zero-copy or asynchronous operations. Custom serializer and sorter can be specified in spark-defaults.xml. For instance, [crail-terasort](https://github.com/zrlio/crail-terasort) defines the shuffle serializer and sorter as follows:

```
spark.crail.shuffle.sorter     com.ibm.crail.terasort.sorter.CrailShuffleNativeRadixSorter
spark.crail.shuffle.serializer com.ibm.crail.terasort.serializer.F22Serializer
```
