---
typora-root-url: image
typora-copy-images-to: image
---

## 源码

```java
private void runIOTest(
    Class<? extends Mapper<Text, LongWritable, Text, Text>> mapperClass,
    Path outputDir) throws IOException {
    JobConf job = new JobConf(config, TestDFSIO.class);

    FileInputFormat.setInputPaths(job, getControlDir(config));
    job.setInputFormat(SequenceFileInputFormat.class);

    job.setMapperClass(mapperClass);
    job.setReducerClass(AccumulatingReducer.class);

    FileOutputFormat.setOutputPath(job, outputDir);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(Text.class);
    job.setNumReduceTasks(1); 		// Reduce强制配置为1
    JobClient.runJob(job);
}
```



