---
layout: post
title: "Crail Storage Performance -- Part I: DRAM (Draft)"
author: Patrick Stuedi
category: blog
comments: true
---

<div style="text-align: justify"> 
<p>
This is the first of three blog posts illustrating Crail's raw storage performance. In part I we cover Crail's DRAM storage tier, part II will be about Crail's NVMe flash storage tier, and part III will be about Crail's metadata performance. 
</p>
</div>

### Hardware Configuration

The specific cluster configuration used for the experiments in this blog:

* Cluster
  * 8 node OpenPower cluster
* Node configuration
  * CPU: 2x OpenPOWER Power8 10-core @2.9Ghz
  * DRAM: 512GB DDR4
  * Network: 1x100Gbit/s Ethernet Mellanox ConnectX-4 EN (RoCE)
             1x100Gbit/s Infiniband Mellanox ConnectX-4 EN
* Software
  * RedHat 7.2 with Linux kernel version 4.10.13
  * Crail 1.0, internal version 2842

### Anatomy of a Crail Data Operation

<div style="text-align: justify"> 
<p>
Data operations in Crail -- such as the reading or writing of files -- are internally composed of metadata operations and actual data transfers. Let's look at a simple Crail application that opens a file and reads the file sequentially:
</p>
</div>
```
CrailConfiguration conf = new CrailConfiguration();
CrailFS fs = CrailFS.newInstance(conf);
CrailFile file = fs.lookup(filename).get().asFile();
CrailInputStream stream = file.getDirectInputStream();
while(stream.available() > 0){
    Future<Buffer> future = stream.read(buf);
    //Do something useful
    ...
    //Await completion of operation
    future.get();
}
```    
<div style="text-align: justify"> 
<p>
One challenge with file read/write operations is to avoid blocking in case block metadata information is missing. Crail caches block metadata at the client, but caching is ineffective for both random reads and write-once read-once data. To avoid blocking for sequential read/write operations, Crail interleaves metadata operations and actual data transfers.
</p>
</div>
<br>
<div style="text-align:center"><img src ="http://crail.io/img/blog/crail-memory/anatomy.png" width="400"></div>
<br>
<div style="text-align: justify"> 
<p>
Each read operation always triggers the lookup of block metadata for the next block immediately after issuing the RDMA read operation for the current block. Note that the asynchronous and non-blocking nature of RDMA allows both operations to be executed in the process context of the application, without context switching or any additional background threads. The figure also illustrates the efficiency of Crail for small operations. During the last operation, with only a few bytes left to be read, the byte-granular nature of Crail's block access protocol makes sure that only the relevant bytes are transmitted over the network, as opposed to transmitting the entire block. This basic read/write logic is common to all storage tiers in Crail. In the remainder of this post, we specificially look at the performance of Crail's DRAM storage tier.
</p>
</div>

### Sequential Read/Write Performance

<div style="text-align: justify"> 
<p>
Let's start by looking at sequential read/write performance. These benchmarks can be run easily from the command line. Here is an example for a sequential read experiments issuing 100M read operations of size 1K against a 100GB file. We further use 32 warmup read operations which are excluded from the measurements.
</p>
</div>
```
./bin/crail iobench -t writeClusterHeap -s 1024 -k 100000000 -w 32 -f /tmp.dat
```    
<br>
<div style="text-align:center"><img src ="http://crail.io/img/blog/crail-memory/read.svg" width="550"/></div>
<br><br>