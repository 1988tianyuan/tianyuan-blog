title: 对于Map-Reduce并行度的理解
author: 天渊
tags:
  - map-reduce
categories:
  - 大数据
date: 2019-07-24 21:08:00
---

hadoop计算框架map-reduce有一个并行度的概念，每个job，对于输入文件A，需要对A进行切片（即`split`），再针对各个`split`单独启动独立的`mapTask`进行计算（hadoop 2.0后由yarn完成），切片完成后启动多个`mapTask`即为mr任务的并行度

<!-- more -->

### map-reduce的split方式

默认情况下，文件的单个split大小（即`split-size`）通常与HDFS的`block-size`保持一致（即默认的128MB），该工作由`FileInputFormat`调用`getSplits()`方法来完成，通过读取文件metadata进行切分，生成对应的`FileSplit`对象，其中就包含了各个文件切片的offset和length等信息，再序列化到`job.splits`文件中：

```java
// InputFormat类中的方法，从JobContext中获取输入文件的信息
// 根据输入文件信息生成切分信息
public abstract List<InputSplit> getSplits(JobContext context)
```

文件的切分信息保存在`FileSplit`对象中，主要保存的了文件在相应`FileSystem`上的path，切分开始位置和切分长度，以及主机信息和当前Split的具体位置：

```java
public class FileSplit extends InputSplit implements Writable {
  private Path file;
  private long start;
  private long length;
  private String[] hosts;
  private SplitLocationInfo[] hostInfos;
    
  ......
}
```

### map-reduce任务提交过程

map-reduce在客户端完成切分工作后上传到服务器，针对每个`Split`单独启动mapTask，下面来看看在客户端提交job后是怎么进行split的：

1. `job.submit()`后，初始化一个`JobSubmitter`对象进行job的提交工作

2. `JobSubmitter`调用`submitJobInternal(job, cluster)`方法进行方法的提交，在进行一系列的初始化过程后，调用`writeSplits(job, submitJobDir)`方法进行切分

3. `writeSplits`最终就会调用上面提到的`InputFormat`的`getSplits`方法

4. 在`getSplits`方法中，首先计算split的最大和最小限制：

   ```java
   // 默认是1，可以通过mapreduce.input.fileinputformat.split.minsize属性进行设置
   long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job));
   // 默认是Long.MAX_VALUE，可以通过mapreduce.input.fileinputformat.split.maxsize属性来设置
   long maxSize = getMaxSplitSize(job);
   ```

5. 在确认最大最小范围后，需要确定真正需要`splitSize`，使用`computeSplitSize`方法进行确认，可以看出通常情况下`splitSize`即为`blockSize`：

   ```java
   long splitSize = computeSplitSize(blockSize, minSize, maxSize);
   //computeSplitSize方法：
   protected long computeSplitSize(long blockSize, long minSize, long maxSize) {
       // 在maxSize和blockSize中取小值，最后保证比minSize大
       return Math.max(minSize, Math.min(maxSize, blockSize));
   }
   ```

6. 对文件进行split，代码如下：

   ```java
   // 剩余还未split的数量
   long bytesRemaining = length;
   // 如果剩余数量多于1.1倍的splitSize，则持续进行split
   while (((double) bytesRemaining)/splitSize > SPLIT_SLOP) {
       // 获取当前offset所处的block
       int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
       // 生成split
       splits.add(makeSplit(path, length-bytesRemaining, splitSize,
                            blkLocations[blkIndex].getHosts(),
                            blkLocations[blkIndex].getCachedHosts()));
       // 更新剩余数量
       bytesRemaining -= splitSize;
   }
   // 剩下的数量小于等于1.1倍的splitSize，直接把他们放到一个split里面去
   if (bytesRemaining != 0) {
       int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
       splits.add(makeSplit(path, length-bytesRemaining, bytesRemaining,
                            blkLocations[blkIndex].getHosts(),
                            blkLocations[blkIndex].getCachedHosts()));
   }
   ```

7. 完成split后，通过`JobSplitWriter`将splits信息保存到一个临时文件`job.splits`中：

   ```java
   private <T extends InputSplit>
     int writeNewSplits(JobContext job, Path jobSubmitDir) throws IOException,
         InterruptedException, ClassNotFoundException {
       Configuration conf = job.getConfiguration();
       InputFormat<?, ?> input =
         ReflectionUtils.newInstance(job.getInputFormatClass(), conf);
       // 获取splits      
       List<InputSplit> splits = input.getSplits(job);
       T[] array = (T[]) splits.toArray(new InputSplit[splits.size()]);
       Arrays.sort(array, new SplitComparator());
       // 将splits写到本地临时文件      
       JobSplitWriter.createSplitFiles(jobSubmitDir, conf, 
           jobSubmitDir.getFileSystem(conf), array);
       // 返回split的数量      
       return array.length;
     }
   ```

   其中`jobSubmitDir`是在提交阶段在本地创建的用于保存提交信息的临时文件夹，最后将split数目返回，即为需要启动的`mapTask`数目，也就是并行度

8. 最后提交本次job，其中`submitClient`即为提交客户端，如果在yarn环境下是由`YARNRunner`这个类来完成：

   ```java
   status = submitClient.submitJob(jobId, submitJobDir.toString(), job.getCredentials());
   ```

至此整个提交过程完成，`yarn`会根据提交数据（包括split信息和job配置信息）再结合各计算节点的资源利用率，将job提交给各个计算节点启动多个`mapTask`进行计算

关于任务提交后的流程就得研究`yarn`的运行机制了





