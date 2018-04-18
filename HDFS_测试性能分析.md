---
typora-root-url: image
typora-copy-images-to: image
---



# Map为何不是平均分配

5个DataNode，为何15个Map不是平均分配至各节点？

![1523606530989](/1523606530989.png)

# 为何Shuffle会花这么长时间?

对于 TestDFSIO读写，Shuffle这个子流程在做什么，时间统计是否把等待也统计进去了，还是实际就消耗这么多时间?



TestDFSIO 3 replication write 500G。

![1523606760262](/1523606760262.png)

Job只有一个Map时，Shuffle的时间并不长

| Job Name:            | hadoop-mapreduce-client-jobclient-3.0.0-tests.jar |
| -------------------- | ------------------------------------------------- |
| User Name:           | root                                              |
| Queue:               | default                                           |
| State:               | SUCCEEDED                                         |
| Uberized:            | false                                             |
| Submitted:           | Sun Apr 15 06:04:15 EDT 2018                      |
| Started:             | Sun Apr 15 06:04:23 EDT 2018                      |
| Finished:            | Sun Apr 15 06:28:19 EDT 2018                      |
| Elapsed:             | 23mins, 56sec                                     |
| Diagnostics:         |                                                   |
| Average Map Time     | 23mins, 48sec                                     |
| Average Shuffle Time | 2sec                                              |
| Average Merge Time   | 0sec                                              |
| Average Reduce Time  | 0sec                                              |

read的Shuffle时间比。

![1523607324624](/1523607324624.png)

## write任务

Shuffle Time为何很大?

| Job Name:            | hadoop-mapreduce-client-jobclient-3.0.0-tests.jar |
| -------------------- | ------------------------------------------------- |
| User Name:           | root                                              |
| Queue:               | default                                           |
| State:               | SUCCEEDED                                         |
| Uberized:            | false                                             |
| Submitted:           | Fri Apr 13 03:34:24 EDT 2018                      |
| Started:             | Fri Apr 13 03:34:30 EDT 2018                      |
| Finished:            | Fri Apr 13 03:42:47 EDT 2018                      |
| Elapsed:             | 8mins, 17sec                                      |
| Diagnostics:         |                                                   |
| Average Map Time     | 3mins, 56sec                                      |
| Average Shuffle Time | 5mins, 58sec                                      |
| Average Merge Time   | 0sec                                              |
| Average Reduce Time  | 0sec                                              |

Container的内存与CPU被限制了? 

导致最多只能起8个container?

| Total Vmem allocated for Containers   | 32 GB |
| ------------------------------------- | ----- |
| Vmem enforcement enabled              | false |
| Total Pmem allocated for Container    | 8 GB  |
| Pmem enforcement enabled              | true  |
| Total VCores allocated for Containers | 8     |

# 磁盘IO读写平滑性为何差距很大

写

![1523608741738](/1523608741738.png)

读

![1523608805484](/1523608805484.png)

## 可能原因

各种寻查网络请求延时，特别是向NameNode请求?

数据传输网络延迟?

Hadoop对单个文件都是顺序流式读写，所以节点间要等待。

磁盘寻道时间多少?

已经验证了是单文件内部顺序读，那预读取是有利还是有弊。

那使用缓存读为何也慢，是压根就没从缓存读(从测试数据上看好像有一些)，还是说向NameNode请求太慢(网络延迟小于1ms)。

单磁盘随机读测试，看是否还真100M左右?

为何EC比3复本读的时候磁盘IO更平滑呢?

# 中途出现磁盘与网络全无数据

为何? 是Shuffle在等，还是在等被killed的task?



![1523871366690](/1523871366690.png)

# 跟踪读block

文件只有128M，即只有一个block大小，而且map task是分配至locality DataNode，却要读10秒钟。

```
2018-04-17 10:10:31,068 INFO [main] org.apache.hadoop.mapred.MapTask: Processing split: hdfs://hadoop2:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_100:0+114
2018-04-17 10:10:31,122 INFO [main] org.apache.hadoop.mapred.MapTask: numReduceTasks: 1
2018-04-17 10:10:31,185 INFO [main] org.apache.hadoop.mapred.MapTask: (EQUATOR) 0 kvi 26214396(104857584)
2018-04-17 10:10:31,185 INFO [main] org.apache.hadoop.mapred.MapTask: mapreduce.task.io.sort.mb: 100
2018-04-17 10:10:31,185 INFO [main] org.apache.hadoop.mapred.MapTask: soft limit at 83886080
2018-04-17 10:10:31,185 INFO [main] org.apache.hadoop.mapred.MapTask: bufstart = 0; bufvoid = 104857600
2018-04-17 10:10:31,185 INFO [main] org.apache.hadoop.mapred.MapTask: kvstart = 26214396; length = 6553600
2018-04-17 10:10:31,190 INFO [main] org.apache.hadoop.mapred.MapTask: Map output collector class = org.apache.hadoop.mapred.MapTask$MapOutputBuffer
2018-04-17 10:10:31,212 INFO [main] org.apache.hadoop.fs.TestDFSIO: in = org.apache.hadoop.hdfs.client.HdfsDataInputStream
2018-04-17 10:10:41,360 INFO [main] org.apache.hadoop.fs.TestDFSIO: Number of bytes processed = 134217728
2018-04-17 10:10:41,360 INFO [main] org.apache.hadoop.fs.TestDFSIO: Exec time = 10147
2018-04-17 10:10:41,360 INFO [main] org.apache.hadoop.fs.TestDFSIO: IO rate = 12.614566
2018-04-17 10:10:41,363 INFO [main] org.apache.hadoop.mapred.MapTask: Starting flush of map output
2018-04-17 10:10:41,363 INFO [main] org.apache.hadoop.mapred.MapTask: Spilling map output
```



对比读时间，没有明显的规律性，有些先打开读InputStream的，反而要等很久才执行完

```
2018-04-17 10:30org.apache.hadoop.mapred.MapTask: bufstart = 0; bufvoid = 104857600
2018-04-17 10:30org.apache.hadoop.mapred.MapTask: kvstart = 26214396; length = 6553600
2018-04-17 10:30org.apache.hadoop.mapred.MapTask: Map output collector class = org.apache.hadoop.mapred.MapTask$MapOutputBuffer
2018-04-17 10:30org.apache.hadoop.fs.TestDFSIO: in = org.apache.hadoop.hdfs.client.HdfsDataInputStream
2018-04-17 10:30org.apache.hadoop.fs.TestDFSIO: Number of bytes processed = 134217728
2018-04-17 10:30org.apache.hadoop.fs.TestDFSIO: Exec time = 35667
2018-04-17 10:30org.apache.hadoop.fs.TestDFSIO: IO rate = 3.5887516
```

而有些后打开InputStream的反而立刻执行

```
org.apache.hadoop.mapred.MapTask: kvstart = 26214396; length = 6553600
org.apache.hadoop.mapred.MapTask: Map output collector class = org.apache.hadoop.mapred.MapTask$MapOutputBuffer
org.apache.hadoop.fs.TestDFSIO: in = org.apache.hadoop.hdfs.client.HdfsDataInputStream
org.apache.hadoop.fs.TestDFSIO: Number of bytes processed = 134217728
org.apache.hadoop.fs.TestDFSIO: Exec time = 897
org.apache.hadoop.fs.TestDFSIO: IO rate = 142.69788
org.apache.hadoop.mapred.MapTask: Starting flush of map output
org.apache.hadoop.mapred.MapTask: Spilling map output
```



# HDF连续读与fio对比

fio测试脚本如下，当bs=512k时，读IO达最大为170MB/s。即此HDD最大读IO。897

```
fio -filename=/mnt/sdb/test_fio -direct=1 -iodepth=8 -rw=read -ioengine=libaio -bs=4k -size=3G -numjobs=1 -runtime=60 -group_reporting -name=fiotest1 -thread
```

下图左边是HDFS的连续读。

![1524027910525](/1524027910525.png)