---
typora-root-url: image
typora-copy-images-to: image
---

原文: http://geek.csdn.net/news/detail/77338

英特尔的人写的，陈述到位。



### HDFS EC疑问点

- 是否有短路读、本地读?
- 数据如何从DFSClient组织发往DataNode，也是像3复本一样以64k Packet发送? 是的，重命名为DFSPacket。
- 恢复丢失的块(3复本与EC)在哪恢复?  是在源节点(有数据的节点上恢复)，然后再写入目标节点。
- 一个文件，EC的块组未写满HDFS的内部块，这些内部块能否用于其它文件? 不可以，内部块ID从EC块组的ID生成的，即有强关联。这样的话，对于小文件，内部块就不会写满了。(待验证)