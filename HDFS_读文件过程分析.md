---
typora-root-url: image
typora-copy-images-to: image
---

# 读IO

TODO

# Data Locality分析

从日志中跟踪

can run a MapReduce job using TRACE level logging by setting mapreduce.map.log.level=TRACE and then collecting the logs using ‘yarn logs –applicationId’ command. Be prepared for big log files, so I would recommend running small jobs for this. Newer Hadoop versions have an ‘-out’ option for this command which is very useful since it’s creating a separate file for each container, otherwise you have to use a little scripting to do that yourself. This command does the job: csplit -n 4 <log filename> ‘/=================/-1' '{*}'

# Example local read

local read第次读1024k or  1000000？

```
TRACE [main] org.apache.hadoop.hdfs.client.impl.BlockReaderLocal: read(arr.length=1000000, off=0, len=1000000, filename=/benchmarks/TestDFSIO/io_data/test_io_117, block=BP-1009768169-172.24.37.12-1525258466502:blk_1073744393_3569, canSkipChecksum=false): starting
TRACE [main] org.apache.hadoop.hdfs.client.impl.BlockReaderLocal: loaded 1048576 bytes into bounce buffer from offset 54525952 of BP-1009768169-172.24.37.12-1525258466502:blk_1073744393_3569
TRACE [main] org.apache.hadoop.hdfs.client.impl.BlockReaderLocal: read(arr.length=1000000, off=0, len=1000000, filename=/benchmarks/TestDFSIO/io_data/test_io_117, block=BP-1009768169-172.24.37.12-1525258466502:blk_1073744393_3569, canSkipChecksum=false): returning 1000000
TRACE [main] org.apache.hadoop.hdfs.shortcircuit.ShortCircuitCache: ShortCircuitReplica{key=1073744393_BP-1009768169-172.24.37.12-1525258466502, metaHeader.version=1, metaHeader.checksum=DataChecksum(type=CRC32C, chunkSize=512), ident=0x67a056f1, creationTimeMs=468447421}: could not add no-checksum anchor to slot Slot(slotIdx=0, shm=DfsClientShm(2764f0fab00f2aecbd922d78a8d7a3de))
```

## Example remote read

remote read第次读65kb。

```
2018-05-02 08:05:35,936 TRACE [main] org.apache.hadoop.hdfs.client.impl.BlockReaderRemote: Finishing read #3b987007-1265-40d9-b4a2-c319cc745870
2018-05-02 08:05:35,937 TRACE [main] org.apache.hadoop.hdfs.client.impl.BlockReaderRemote: Starting read #34fdeab7-fb66-4c4c-a960-b4ad3eee3c24 file /benchmarks/TestDFSIO/io_data/test_io_117 from datanode hadoop6
2018-05-02 08:05:35,937 TRACE [main] org.apache.hadoop.hdfs.protocol.datatransfer.PacketReceiver: readNextPacket: dataPlusChecksumLen=66048, headerLen=25
2018-05-02 08:05:35,937 TRACE [main] org.apache.hadoop.hdfs.client.impl.BlockReaderRemote: DFSClient readNextPacket got header PacketHeader with packetLen=66052 header data: offsetInBlock: 65536
seqno: 1
lastPacketInBlock: false
dataLen: 65536
```

## Split info

## Unusable short circuit

## 参考

https://community.hpe.com/t5/Around-the-Storage-Block/Data-Locality-in-Hadoop-Taking-a-Deep-Dive/ba-p/6969665#.WumiknWFPnp

