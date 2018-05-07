---
typora-root-url: image
typora-copy-images-to: image
---



# 名字解释

TestDFSIO源码就一个文件，可以比较明了地从源码中解读出结果。

示例:

```
INFO fs.TestDFSIO: ----- TestDFSIO ----- : read
INFO fs.TestDFSIO:             Date & time: Tue Apr 17 23:42:41 EDT 2018
INFO fs.TestDFSIO:         Number of files: 1
INFO fs.TestDFSIO:  Total MBytes processed: 30000
INFO fs.TestDFSIO:       Throughput mb/sec: 149.42
INFO fs.TestDFSIO:  Average IO rate mb/sec: 149.42
INFO fs.TestDFSIO:   IO rate std deviation: 0.02
INFO fs.TestDFSIO:      Test exec time sec: 234
```

下述公式中的N为map task个数，即任务数。

* 平均IO速率

$$
\text{Average IO rate(N)} = \frac{\sum_{i=0}^{N} rate_i} {N} =\frac{\sum_{i=0}^N \frac{filesize_i}{time_i}} {N}
$$

* 平均吞吐

$$
\text{Throughput(N)} = \frac{\sum_{i=0}^N {filesize_i}} {\sum_{i=0}^N {time_i}}
$$

* IO速率的方差

  ps: 方差应该是"离差平方和平均后的方根"?

$$
\text{IO rate std deviation} = \sqrt{\frac{\sum_{i=0}^{N} {rate_i^2}} {N} - \text{(Average IO rate(N))}^2}
$$

* 运行时间
  Test exec time sec 为NameNode开始为Job分配资源算起至Reduce结束，因此不是TestDFSIO命令运行时就算起。

其中，某个map task的log记录，上述公式的 $filesize_i$ 与 $ time_i $ 即来源于此。

```
org.apache.hadoop.mapred.Task:  Using ResourceCalculatorProcessTree : [ ]
org.apache.hadoop.mapred.MapTask: Processing split: hdfs://hadoop2:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_26:0+113
org.apache.hadoop.mapred.MapTask: numReduceTasks: 1
org.apache.hadoop.mapred.MapTask: (EQUATOR) 0 kvi 26214396(104857584)
org.apache.hadoop.mapred.MapTask: mapreduce.task.io.sort.mb: 100
org.apache.hadoop.mapred.MapTask: soft limit at 83886080
org.apache.hadoop.mapred.MapTask: bufstart = 0; bufvoid = 104857600
org.apache.hadoop.mapred.MapTask: kvstart = 26214396; length = 6553600
org.apache.hadoop.mapred.MapTask: Map output collector class = org.apache.hadoop.mapred.MapTask$MapOutputBuffer
org.apache.hadoop.fs.TestDFSIO: in = org.apache.hadoop.hdfs.client.HdfsDataInputStream
org.apache.hadoop.fs.TestDFSIO: Number of bytes processed = 1048576000
org.apache.hadoop.fs.TestDFSIO: Exec time = 45101
org.apache.hadoop.fs.TestDFSIO: IO rate = 22.172457
org.apache.hadoop.mapred.MapTask: Starting flush of map output
org.apache.hadoop.mapred.MapTask: Spilling map output
```

# 关键源码

## Map

TestDFSIO测试map如ReadMapper, ReadMapper等的基类:

```java
private abstract static class IOStatMapper extends IOMapperBase<Long> {
    @Override // IOMapperBase
    void collectStats(OutputCollector<Text, Text> output,
                      String name,
                      long execTime,
                      Long objSize) throws IOException {
        long totalSize = objSize.longValue();
        float ioRateMbSec = (float)totalSize * 1000 / (execTime * MEGA);
        LOG.info("Number of bytes processed = " + totalSize);
        LOG.info("Exec time = " + execTime);
        LOG.info("IO rate = " + ioRateMbSec);

        output.collect(new Text(AccumulatingReducer.VALUE_TYPE_LONG + "tasks"),
                       new Text(String.valueOf(1)));
        output.collect(new Text(AccumulatingReducer.VALUE_TYPE_LONG + "size"),
                       new Text(String.valueOf(totalSize)));
        output.collect(new Text(AccumulatingReducer.VALUE_TYPE_LONG + "time"),
                       new Text(String.valueOf(execTime)));
        output.collect(new Text(AccumulatingReducer.VALUE_TYPE_FLOAT + "rate"),
                       new Text(String.valueOf(ioRateMbSec*1000)));
        output.collect(new Text(AccumulatingReducer.VALUE_TYPE_FLOAT + "sqrate"),
                       new Text(String.valueOf(ioRateMbSec*ioRateMbSec*1000)));
    }
}
```

## Reduce

从代码可以看出，其使用最简单的累加器。

```java
job.setMapperClass(mapperClass);
job.setReducerClass(AccumulatingReducer.class);
```

Reduce的结果保存在HDFS中的位置与内容:

```
hdfs dfs -cat /benchmarks/TestDFSIO/io_read/part-00000
f:rate	213701.23
f:sqrate	1102287.4
l:size	26843545600
l:tasks	50
l:time	6486094
```

## Result

使用Reduce的汇总来计算最终数值.

```java
private void analyzeResult( FileSystem fs,
                           TestType testType,
                           long execTime,
                           String resFileName
    ) throws IOException {
        try {
            in = new DataInputStream(fs.open(reduceFile));
            lines = new BufferedReader(new InputStreamReader(in));
            String line;
            while((line = lines.readLine()) != null) {
                StringTokenizer tokens = new StringTokenizer(line, " \t\n\r\f%");
                String attr = tokens.nextToken();
                if (attr.endsWith(":tasks"))
                    tasks = Long.parseLong(tokens.nextToken());
                else if (attr.endsWith(":size"))
                    size = Long.parseLong(tokens.nextToken());
                else if (attr.endsWith(":time"))
                    time = Long.parseLong(tokens.nextToken());
                else if (attr.endsWith(":rate"))
                    rate = Float.parseFloat(tokens.nextToken());
                else if (attr.endsWith(":sqrate"))
                    sqrate = Float.parseFloat(tokens.nextToken());
            }
        } finally {
        }
        double med = rate / 1000 / tasks;
        double stdDev = Math.sqrt(Math.abs(sqrate / 1000 / tasks - med*med));
        DecimalFormat df = new DecimalFormat("#.##");
        String resultLines[] = {
                "----- TestDFSIO ----- : " + testType,
                "            Date & time: " + new Date(System.currentTimeMillis()),
                "        Number of files: " + tasks,
                " Total MBytes processed: " + df.format(toMB(size)),
                "      Throughput mb/sec: " + df.format(toMB(size) / msToSecs(time)),
                " Average IO rate mb/sec: " + df.format(med),
                "  IO rate std deviation: " + df.format(stdDev),
                "     Test exec time sec: " + df.format(msToSecs(execTime)),
                "" };
    }
```

# Throughput与IO rate区别

下面的对比就比较明白了，源数据是一样的，只是计算求和与顺序不一样，根据数学定理，当各map task指标平均时，两个值是一样的。

下面以实际的测试结果为例，以下结果从Jobhistory中查看可得。

多DataNode读:

```
INFO fs.TestDFSIO: ----- TestDFSIO ----- : read
INFO fs.TestDFSIO:             Date & time: Mon Apr 16 09:50:58 EDT 2018
INFO fs.TestDFSIO:         Number of files: 300
INFO fs.TestDFSIO:  Total MBytes processed: 300000
INFO fs.TestDFSIO:       Throughput mb/sec: 77.07
INFO fs.TestDFSIO:  Average IO rate mb/sec: 90.76
INFO fs.TestDFSIO:   IO rate std deviation: 47.17
INFO fs.TestDFSIO:      Test exec time sec: 66.1
```

只有本地Shortcircuit连续读(一块磁盘):

```
INFO fs.TestDFSIO: ----- TestDFSIO ----- : read
INFO fs.TestDFSIO:             Date & time: Tue Apr 17 23:42:41 EDT 2018
INFO fs.TestDFSIO:         Number of files: 1
INFO fs.TestDFSIO:  Total MBytes processed: 30000
INFO fs.TestDFSIO:       Throughput mb/sec: 149.42
INFO fs.TestDFSIO:  Average IO rate mb/sec: 149.42
INFO fs.TestDFSIO:   IO rate std deviation: 0.02
INFO fs.TestDFSIO:      Test exec time sec: 234
```



# 参考文章

https://discuss.pivotal.io/hc/en-us/articles/200864057-Running-DFSIO-MapReduce-benchmark-test

