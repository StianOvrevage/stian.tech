---
layout: post
title:  "Disk performance on Azure Kubernetes Service (AKS) - Part 1: Benchmarking"
date:   2019-02-23 00:00:00 +0000
categories:
excerpt_separator: <!--more-->
---

DRAFT

<!--more-->

Table of contents

  - [Background](#Background)
    - [Metric Methodologies](#MetricsMethodologies)
  - [Storage Background](#StorageBackground)
  - [What to measure?](#WhatToMeasure)
  - [Test 1 - Learning to dislike Azure Cache](#Test1)
  - [Test 2 - Sequential write, direct/sync, 4KB block size, IO depth 1](#Test2)
  - [Test 3 - Sequential write, direct/sync, 4KB block size, IO depth 16](#Test3)
  - [Test 4 - Sequential write, direct/sync, 128KB block size](#Test4)
  - [Test 5 - Sequential write, direct/sync, 256KB block size](#Test5)
  - [Test 6 - Sequential write, buffered, 256KB block size](#Test6)
  - [Test 7 - Random write, buffered, 256KB block size](#Test7)
  - [Test 8 - Random write, buffered, 256KB block size, IO depth 16](#Test8)
  - [Test 9 - Random write, buffered, 4KB block size, IO depth 16](#Test9)
  - [Conclusion](#Conclusion)

#### Microsoft Azure

> [If you don't have a Azure subscription already you can try services for $200 for 30 days.](https://azure.microsoft.com/en-us/free/) The VM size **Standard_B2s** is Burstable, has 2vCPU, 4GB RAM, 8GB temp storage and costs roughly $38 / month. For $200 you can have a cluster of 3-4 B2s nodes plus traffic, loadbalancers and other additional costs.

> See my blog post [Managed Kubernetes on Microsoft Azure (English)](2017-12-23-managed-kubernetes-on-azure.md) for information on how to get up and running with Kubernetes on Azure.

> _I have no affiliation with Microsoft Azure except using them through work._

<a id="Background"></a>
## Background

I'm part of a team at Equinor building an internal PaaS based on Kubernetes running on AKS (Azure managed Kubernetes). We use Prometheus for monitoring each cluster as well as InfluxDB for collecting metrics from k6io which runs continous tests on our public endpoints.

A couple of weeks ago we discovered some potential problems with both Prometheus and InfluxDB with memory usage and restarts. High CPU usage of type `iowait` suggested that there might be some disk issues contributing to the problems.

> iowait: "Percentage of time that the CPU or CPUs were idle during which the system had an outstanding disk I/O request." ([hpe.com](https://support.hpe.com/hpsc/doc/public/display?docId=c02783994)). You can see `iowait` on your Linux system by running `top` and looking at the `wa` percentage.
> 
> PS: You can have a disk IO bottleneck even with low `iowait`, and a high `iowait` does not always indicate a disk IO bottleneck ([ibm.com](https://www.ibm.com/developerworks/community/blogs/AIXDownUnder/entry/iowait_a_misleading_indicator_of_i_o_performance54?lang=en)).

First off we need to benchmark the underlying disk to get an understanding of it's performance limits and characteristics. That is what we will cover in this post.

<a id="MetricsMethodologies"></a>
### Metric Methodologies

There are two helpful methodologies when monitoring information systems. The first one is Utilization, Saturation and Errors (USE) from [Brendan Gregg](http://www.brendangregg.com/usemethod.html) and the second one is Rate, Errors, Duration (RED) from [Tom Wilkie](https://www.slideshare.net/weaveworks/monitoring-microservices). RED is best suited when observing workloads and transactions while USE is best suited for observing resources.

I'll be using the USE method here. USE can be summarised as:

  * **For every resource, check utilization, saturation, and errors.**
    * **resource**: all physical server functional components (CPUs, disks, busses, ...)
    * **utilization**: the average time that the resource was busy servicing work
    * **saturation**: the degree to which the resource has extra work which it can't service, often queued
    * **errors**: the count of error events

<a id="StorageBackground"></a>
## Storage Background

Disk usage has two dimensions, throughput/bandwidth(BW) and operations per second (IOPS), and the underlying storage system will have upper limits of how much data it can receive (BW) and the number of operations it can perform per second (IOPS).

> Background - harddrive types: harddrives come in two types, Solid State Disks (SSD) and spindle (HDD). A SSD disk is a microship capable of permanently storing data while a HDD uses spinning platters to store data. HDDs have a fixed rate of rotation (RPM), typically 5.400 and 7.200 RPM for lower cost drives for home use and higher cost 10.000 and 15.000 RPM drives for server use. Over the last 20 years of HDDs their storage density has increased, but the RPM has largely stayed the same. A disk with twice the density (500GB to 1TB for example) can read twice as much data on a single rotation and thus increase the bandwidth significantly. However, reading or writing a random block still requires waiting for the disk to spin enough to reach the relevant sector on the disk. So IOPS has not increased much for HDDs and is still a low 125-150 IOPS for a 10.000 RPM enterprise disk. A SSD does not have any moving parts so is able to reach MUCH higher IOPS. A low end Samsung 960 EVO with 500GB capacity costs $150 and can achieve a whopping 330.000 IOPS! ([wikipedia.com](https://en.wikipedia.org/wiki/IOPS))

> Background - access patterns: The way a program uses storage also has a huge impact on the performance one can achieve. Sequential access is when we read or write a large file. When this happens the operating system and harddrive can optimize and "merge" operations so that we can read or write a much bigger chunk of data at a time. If we can read 1MB at a time 150 times per second we get 150MB/s of bandwidth. However, fully random access where the smallest chunk we read or write is a 4KB block the same 150 IOPS would only give a bandwidth of 0.6MB/s!

> Background - cloud vs physical: Now we know what HDDs are limited to a low IOPS and low IOPS combined with a random access pattern gives us a low overall bandwidth. There is a huge gotcha here when it comes to cloud. On Azure when using Premium Managed SSD the IOPS you are given is a factor of the disk size you provision ([microsoft.com](https://azure.microsoft.com/en-us/pricing/details/managed-disks/)). A 512GB disk is limited to 2.300 IOPS and 150MB/s. With 100% random access that only gives about 9MB/s of bandwidth!

> Background - OS caching: To overcome some of the limitations of the underlying disk (mostly IOPS) there are potentially several layers of caching involved. Linux file systems can have `writeback` enabled which causes Linux to temporarily store data that is going to be written to disk in memory. This can give a big performance increase when there are sudden spikes of writes exceeding the performance of the underlying disk. It also increases the chance that operations can be `merged` where several write operations to areas of the disk that are nearby can be executed as one. This caching works best for sudden peaks and will not necessarily be enough if there is continous random writes to disk. This caching also means that even though an application thinks it has saved some data to disk it can be lost in the case of a power outage or other failure. Applications can also explicitly request `direct` access where every operation is persisted to disk before receiving a confirmation. This is a trade-off between performance and durability that needs to be decided based on the application itself and the environment.

> Background - Azure caching: Azure also provides read and write cache for its `disks` which is enabled by default. As we will see soon for our use case it's not a good idea to use.

<a id="WhatToMeasure"></a>
## What to measure?

> These metrics are collected by the Prometheus `node-exporter` and follows it's naming.

With the USE methodology as a guideline and the two separate but related "resources", bandwidth and IOPS we can look for some useful metrics.

Utilization:

  - `rate(node_disk_written_bytes_total)` - Write bandwidth. The maximum is given by Azure and is 25MB/s for our disk size.
  - `rate(node_disk_writes_completed_total)` - Write operations. The maximum is given by Azure and is 120 IOPS for our disk size.
  - `rate(node_disk_io_time_seconds_total)` - Disk active time in percent. The time the disk was busy servicing requests. 100% means fully utilized.

Saturation:

 - `rate(node_cpu_seconds_total{mode="iowait"}` - CPU iowait. The percentage of time a CPU core is blocked from doing useful work because it's waiting for an IO operation to complete (typically disk, but can also be network).

Useful calculated metrics:
 - `rate(node_disk_write_time_seconds_total) / rate(node_disk_writes_completed_total)` - Write latency. How long from a write is requested until it's completed.
 - `rate(node_disk_written_bytes_total) / rate(node_disk_writes_completed_total)` - Write size. How big the **average** write operation is. 4KB is minimum and indicates 100% random access while 512KB is maximum and indicates sequential access.

## How to measure

The best tool for measuring disk performance is `fio`, even though it might seem a bit intimidating at first due to it's insane number of options.

Installing `fio` on Ubuntu:

    apt-get install fio

`fio` executes `jobs` described in a file. Here is the file we start of with:

    [global]
    ioengine=libaio   # sync|libaio|mmap
    direct=1          # If value is true, use non-buffered I/O. This is usually O_DIRECT
    size=5g           # Size of test file
    group_reporting
    thread
    filename=/data/fio-test-file

    [test1]
    loops=10
    readwrite=write   # read|write|randread|randwrite|readwrite|randrw
    iodepth=1         # How many operations to queue to the disk
    blocksize=4k
    cpus_allowed=1    # Only use this CPU core

The fields we will be changing for the various tests are `direct`, `readwrite`, `iodepth` and `blocksize`. Save the contents in a file named `job.fio` and we run a test with `fio job.fio` and wait for a while. We don't necessarily wait until the tests finish but until we reach some steady state of performance as shown in Grafana.

<a id="Test1"></a>
## Test 1 - Learning to dislike Azure Cache

I run the first tests on the OS disk of a Kubernetes node. The OS disks have Azure caching enabled.

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-os-disk-30gb-readwrite-cache,1thread,1k-bs,1cpu.png "graph")

> OS disk 30GB. Azure read + write cache. 1KB block size.

I run the test twice for a few minutes to see the results.

The first 20-30 seconds of the test I get very good performance of 250MB/s and 500 IOPS but that drops to 0 very quickly as the cache is full and writing things to the actual disk. When that happens everything freezes and I cannot even execute shell commands even. The 20-30 seconds of writing causes a 7-8 minutes hang while the cache empties. On the second run of the same test the spikes are smaller at 50MB/s and 100 IOPS and the hangs much shorter.

I'm not sure what to make of this. It's not acceptable that a Kubernetes node becomes unresponsive for many minutes following a short burst of writing. There are scattered recommendations online of disabling caching for write-heavy applications. Since I have not found any way to measure the Azure cache itself, the results are unpredictable and potentially very impactful as well as making it very hard to use the metrics we do have to evaluate application and storage behaviour I've concluded that it's best to use data disks with caching disabled for our workloads (you cannot disable caching on an AKS node OS disk).

<a id="Test2"></a>
## Test 2 - Sequential write, direct/sync, 4KB block size, IO depth 1

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,write,direct,1thread,4k-bs,1cpu.png "graph")

> By disabling the OS cache (`direct=1`) the results are consistent and predictable. There is no `iowait` since the application does not have multiple writes pending at the same time. We are limited by the 125 IOPS limit on the disk and since we use `direct` the OS cannot merge multiple 4KB writes together even though the writes are sequential. This gives us a maximum of 0.5MB/s bandwidth.

<a id="Test3"></a>
## Test 3 - Sequential write, direct/sync, 4KB block size, IO depth 16

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,write,direct,1thread,depth16,4k-bs,1cpu.png "graph")

> For this test we only increase the IO depth from 1 to 16. IO depth is the number of write operations `fio` will execute simultaneously. Since we are using `direct` these will be queued by the OS for writing. This increases the write latency as well by a factor of 16 and we can get a feeling of the write queue by watching `Write time sdc` in the `Disk IO` panel. The `Node disk IO time` shows a consistent `1`, which means the disk is fully utilizied the whole time.

<a id="Test4"></a>
## Test 4 - Sequential write, direct/sync, 128KB block size

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,write,direct,1thread,128k-bs,1cpu.png "graph")

> We increase the block size to 128KB. We are still limited by 125 IOPS but by increasing the block size we are now getting over 15MB/s bandwidth.

<a id="Test5"></a>
## Test 5 - Sequential write, direct/sync, 256KB block size

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,write,direct,1thread,256k-bs,1cpu.png "graph")

> We increase the block size to 256KB. This crosses the threshold where we go from being limited by IOPS to limited by bandwidth. With 100 IOPS of 256KB blocks we hit the ceiling of 25MB/s.

<a id="Test6"></a>
## Test 6 - Sequential write, buffered, 256KB block size

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,write,buffered,1thread,256k-bs,1cpu.png "graph")

> We have now enabled the OS cache/buffer (`direct=0`). We can see that the writes hitting the disk are now merged to 512KB blocks and reducing the IOPS needed to hit the limit of 25MB/s to 50. Enabling the cache also has other effects: CPU suddenly shows significant IO wait. The write latency shoots through the roof. Also note that the writing continued for 3 minutes after I stopped the test at 20:42! **This also means that the bandwidth and IOPS that `fio` sees and reports is higher than what is actually hitting the disk.**

<a id="Test7"></a>
## Test 7 - Random write, buffered, 256KB block size

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,random-write,-buffered,1thread,256k-bs,1cpu.png "graph")

> Here we go from sequential writes to random writes. We are limited by bandwidth. The average size of the blocks actually written to disks, and the IOPS required to hit the bandwidth limit is actually varying a bit throughout the test.

<a id="Test8"></a>
## Test 8 - Random write, buffered, 256KB block size, IO depth 16

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,random-write,-buffered,1thread,qlen16,256k-bs,1cpu.png "graph")

> Changing the queue length from 1 to 16 does not make any difference. This is because the OS is accepting all the writes from `fio` and caching them before writing them to disk.

<a id="Test9"></a>
## Test 9 - Random write, buffered, 4KB block size, IO depth 16

![graph](/assets/2019-02-23-disk-performance-on-aks-part-1.md/2019-02-22-data-disk-30gb-no-cache,random-write,-buffered,1thread,qlen16,4k-bs,1cpu.png "graph")

> For the final test we reduce the block size to 4KB. Which would be a worst case scenario of random small writes all over the disk. We are now limited by 125 IOPS and the bandwidt is about 0.5-1MB/s. Because of the OS cache `fio` was able to create a lot of data and a 3 minute `fio` job took over **60 minutes** to be written to disk!

<a id="Conclusion"></a>
## Conclusion

Access patterns and block sizes have a tremendous impact on the amount of data we are able to write to disk. We also saw in the last test that generating more data than the underlying storage can receive is not sustainable over time.

