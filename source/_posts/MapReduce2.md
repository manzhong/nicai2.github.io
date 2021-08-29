---
title: MapReduce2
abbrlink: 63354
date: 2017-07-06 16:44:32
tags: MapReduce
categories: MapReduce
summary_img:
encrypt:
enc_pwd:
---

# MapReduce详解

## 1 .MapReduce的运行机制详解

## 全流程:

![img](/images/MapReduce/1-MapReduce工作机制-全流程.jpg)

### 1.1:MapTask 工作机制

简单概述：inputFile通过split被逻辑切分为多个split文件，通过Record按行读取内容给map（用户自己实现的）进行处理，数据被map处理结束之后交给OutputCollector收集器，对其结果key进行分区（默认使用hash分区），然后写入buffer，每个map task都有一个内存缓冲区，存储着map的输出结果，当缓冲区快满的时候需要将缓冲区的数据以一个临时文件的方式存放到磁盘，当整个map task结束后再对磁盘中这个map task产生的所有临时文件做合并，生成最终的正式输出文件，然后等待reduce task来拉数据

##### 详细步骤

1. 读取数据组件 **InputFormat** (默认 TextInputFormat) 会通过 `getSplits` 方法对输入目录中文件进行逻辑切片规划得到 `block`, 有多少个 `block`就对应启动多少个 `MapTask`。

2. 将输入文件切分为 `block` 之后, 由 `RecordReader` 对象 (默认是LineRecordReader) 进行**读取**, 以 `\n` 作为分隔符, 读取一行数据, 返回 `<key，value>`. Key 表示每行首字符偏移值, Value 表示这一行文本内容

3. 读取 `block` 返回 `<key,value>`, **进入用户自己继承的 Mapper 类中**，执行用户重写的 map 函数, RecordReader 读取一行这里调用一次

4. Mapper 逻辑结束之后, 将 Mapper 的每条结果通过 `context.write` 进行collect数据收集. 在 collect 中, 会先对其进行分区处理，默认使用 **HashPartitioner**

   - > MapReduce 提供 `Partitioner` 接口, 它的作用就是根据 `Key` 或 `Value` 及 `Reducer` 的数量来决定当前的这对输出数据最终应该交由哪个 `Reduce task` 处理, 默认对 Key Hash 后再以 Reducer 数量取模. 默认的取模方式只是为了平均 Reducer 的处理能力, 如果用户自己对 Partitioner 有需求, 可以订制并设置到 Job 上

5. 接下来, 会将数据写入内存, 内存中这片区域叫做环形缓冲区, 缓冲区的作用是批量收集 Mapper 结果, 减少磁盘 IO 的影响. 我们的 **Key/Value 对以及 Partition 的结果都会被写入缓冲区**. 当然, 写入之前，Key 与 Value 值都会被序列化成字节数组

   - > 环形缓冲区其实是一个数组, 数组中存放着 Key, Value 的序列化数据和 Key, Value 的元数据信息, 包括 Partition, Key 的起始位置, Value 的起始位置以及 Value 的长度，环形结构是一个抽象概念

   - > 缓冲区是有大小限制, 默认是 100MB，当 Mapper 的输出结果很多时, 就可能会撑爆内存, 所以需要在一定条件下将缓冲区中的数据临时写入磁盘, 然后重新利用这块缓冲区， 这个从内存往磁盘写数据的过程被称为 Spill, 中文可译为溢写，这个溢写是由单独线程来完成, 不影响往缓冲区写 Mapper 结果的线程. 溢写线程启动时不应该阻止 Mapper 的结果输出, 所以整个缓冲区有个溢写的比例 `spill.percent`，这个比例默认是 0.8, 也就是当缓冲区的数据已经达到阈值 `buffer size * spill percent = 100MB * 0.8 = 80MB`, 溢写线程启动, 锁定这 80MB 的内存, 执行溢写过程. Mapper 的输出结果还可以往剩下的 20MB 内存中写, 互不影响

6. 当溢写线程启动后, 需要**对这 80MB 空间内的 Key 做排序 (Sort)**. 排序是 MapReduce 模型默认的行为, 这里的排序也是对序列化的字节做的排序

   - > 如果 Job 设置过 Combiner, 那么现在就是使用 Combiner 的时候了. 将有相同 Key 的 Key/Value 对的 Value 加起来, 减少溢写到磁盘的数据量. Combiner 会优化 MapReduce 的中间结果, 所以它在整个模型中会多次使用

   - > 那哪些场景才能使用 Combiner 呢? 从这里分析, Combiner 的输出是 Reducer 的输入, Combiner 绝不能改变最终的计算结果. Combiner 只应该用于那种 Reduce 的输入 Key/Value 与输出 Key/Value 类型完全一致, 且不影响最终结果的场景. 比如累加, 最大值等. Combiner 的使用一定得慎重, 如果用好, 它对 Job 执行效率有帮助, 反之会影响 Reducer 的最终结果

7. **合并溢写文件**, 每次溢写会在磁盘上生成一个临时文件 (写之前判断是否有 Combiner), 如果 Mapper 的输出结果真的很大, 有多次这样的溢写发生, 磁盘上相应的就会有多个临时文件存在. 当整个数据处理结束之后开始对磁盘中的临时文件进行 Merge 合并, 因为最终的文件只有一个, 写入磁盘, 并且为这个文件提供了一个索引文件, 以记录每个reduce对应数据的偏移量

##### 配置

| 配置                                 | 默认值                              | 解释            |
| ---------------------------------- | -------------------------------- | ------------- |
| `mapreduce.task.io.sort.mb`        | 100                              | 设置环型缓冲区的内存值大小 |
| `mapreduce.map.sort.spill.percent` | 0.8                              | 设置溢写的比例       |
| `mapreduce.cluster.local.dir`      | `${hadoop.tmp.dir}/mapred/local` | 溢写数据目录        |
| `mapreduce.task.io.sort.factor`    | 10                               | 设置一次合并多少个溢写文件 |

### 1.2 :ReduceTask 工作机制



Reduce 大致分为 copy、sort、reduce 三个阶段，重点在前两个阶段。copy 阶段包含一个 eventFetcher 来获取已完成的 map 列表，由 Fetcher 线程去 copy 数据，在此过程中会启动两个 merge 线程，分别为 inMemoryMerger 和 onDiskMerger，分别将内存中的数据 merge 到磁盘和将磁盘中的数据进行 merge。待数据 copy 完成之后，copy 阶段就完成了，开始进行 sort 阶段，sort 阶段主要是执行 finalMerge 操作，纯粹的 sort 阶段，完成之后就是 reduce 阶段，调用用户定义的 reduce 函数进行处理

##### 详细步骤

1. **`Copy阶段`**，简单地拉取数据。Reduce进程启动一些数据copy线程(Fetcher)，通过HTTP方式请求maptask获取属于自己的文件。
2. **`Merge阶段`**。这里的merge如map端的merge动作，只是数组中存放的是不同map端copy来的数值。Copy过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比map端的更为灵活。merge有三种形式：内存到内存；内存到磁盘；磁盘到磁盘。默认情况下第一种形式不启用。当内存中的数据量到达一定阈值，就启动内存到磁盘的merge。与map 端类似，这也是溢写的过程，这个过程中如果你设置有Combiner，也是会启用的，然后在磁盘中生成了众多的溢写文件。第二种merge方式一直在运行，直到没有map端的数据时才结束，然后启动第三种磁盘到磁盘的merge方式生成最终的文件。
3. **`合并排序`**。把分散的数据合并成一个大的数据后，还会再对合并后的数据排序。
4. **`对排序后的键值对调用reduce方法`**，键相等的键值对调用一次reduce方法，每次调用会产生零个或者多个键值对，最后把这些输出的键值对写入到HDFS文件中。

### 1.3:Shuffle 过程

map 阶段处理的数据如何传递给 reduce 阶段，是 MapReduce 框架中最关键的一个流程，这个流程就叫 shuffle
shuffle: 洗牌、发牌 ——（核心机制：数据分区，排序，分组，规约，合并等过程）

shuffle 是 Mapreduce 的核心，它分布在 Mapreduce 的 map 阶段和 reduce 阶段。一般把从 Map 产生输出开始到 Reduce 取得数据作为输入之前的过程称作 shuffle。

1. **`Collect阶段`**：将 MapTask 的结果输出到默认大小为 100M 的环形缓冲区，保存的是 key/value，Partition 分区信息等。
2. **`Spill阶段`**：当内存中的数据量达到一定的阀值的时候，就会将数据写入本地磁盘，在将数据写入磁盘之前需要对数据进行一次排序的操作，如果配置了 combiner，还会将有相同分区号和 key 的数据进行排序。
3. **`Merge阶段`**：把所有溢出的临时文件进行一次合并操作，以确保一个 MapTask 最终只产生一个中间数据文件。
4. **Copy阶段**：ReduceTask 启动 Fetcher 线程到已经完成 MapTask 的节点上复制一份属于自己的数据，这些数据默认会保存在内存的缓冲区中，当内存的缓冲区达到一定的阀值的时候，就会将数据写到磁盘之上。
5. **`Merge阶段`**：在 ReduceTask 远程复制数据的同时，会在后台开启两个线程对内存到本地的数据文件进行合并操作。
6. **`Sort阶段`**：在对数据进行合并的同时，会进行排序操作，由于 MapTask 阶段已经对数据进行了局部的排序，ReduceTask 只需保证 Copy 的数据的最终整体有效性即可。
   Shuffle 中的缓冲区大小会影响到 mapreduce 程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快缓冲区的大小可以通过参数调整,  参数：mapreduce.task.io.sort.mb  默认100M

## 2. 案例: Reduce 端实现 JOIN

![img](/images/MapReduce/2-Reduce端join操作.bmp)

![img](/images/MapReduce/3-Reduce端join操作问题.bmp)

### 2.1. 需求

> 假如数据量巨大，两表的数据是以文件的形式存储在 HDFS 中, 需要用 MapReduce 程序来实现以下 SQL 查询运算
>
> ```sql
> select  a.id,a.date,b.name,b.category_id,b.price from t_order a left join t_product b on a.pid = b.id
> ```

##### 商品表

| id    | pname | category_id | price |
| ----- | ----- | ----------- | ----- |
| P0001 | 小米5   | 1000        | 2000  |
| P0002 | 锤子T1  | 1000        | 3000  |

##### 订单数据表

| id   | date     | pid   | amount |
| ---- | -------- | ----- | ------ |
| 1001 | 20150710 | P0001 | 2      |
| 1002 | 20150710 | P0002 | 3      |

### 2.2 实现步骤

通过将关联的条件作为map输出的key，将两表满足join条件的数据并携带数据所来源的文件信息，发往同一个reduce task，在reduce中进行数据的串联

#### Step 1: 定义 Mapper

```java
public class ReduceJoinMapper extends Mapper<LongWritable,Text,Text,Text> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //1:判断数据来自哪个文件
        FileSplit fileSplit = (FileSplit) context.getInputSplit();
        String fileName = fileSplit.getPath().getName();
        if(fileName.equals("product.txt")){
            //数据来自商品表
            //2:将K1和V1转为K2和V2,写入上下文中
            String[] split = value.toString().split(",");
            String productId = split[0];

            context.write(new Text(productId), value);

        }else{
            //数据来自订单表
            //2:将K1和V1转为K2和V2,写入上下文中
            String[] split = value.toString().split(",");
            String productId = split[2];

            context.write(new Text(productId), value);

        }

    }
}

```

#### Step 2: 定义 Reducer

```java
public class ReduceJoinMapper extends Mapper<LongWritable,Text,Text,Text> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //1:判断数据来自哪个文件

        FileSplit fileSplit = (FileSplit) context.getInputSplit();
        String fileName = fileSplit.getPath().getName();
        if(fileName.equals("product.txt")){
            //数据来自商品表
            //2:将K1和V1转为K2和V2,写入上下文中
            String[] split = value.toString().split(",");
            String productId = split[0];

            context.write(new Text(productId), value);

        }else{
            //数据来自订单表
            //2:将K1和V1转为K2和V2,写入上下文中
            String[] split = value.toString().split(",");
            String productId = split[2];

            context.write(new Text(productId), value);

        }



    }
}
```

#### Step 3: 定义主类

```java
public class ReduceJoinReducer extends Reducer<Text,Text,Text,Text> {
    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
       //1:遍历集合,获取V3 (first +second)
        String first = "";
        String second = "";
        for (Text value : values) {
            if(value.toString().startsWith("p")){
                first = value.toString();
            }else{
                second += value.toString();
            }

        }
        //2:将K3和V3写入上下文中
        context.write(key, new Text(first+"\t"+second));
    }
}
```

## 3. 案例: Map端实现 JOIN

![img](/images/MapReduce/4-Map端join操作.bmp)

#### 3.1 概述

​	适用于关联表中有小表的情形.

​	使用分布式缓存,可以将小表分发到所有的map节点，这样，map节点就可以在本地对自己所读到的大表数据进行join并输出最终结果，可以大大提高join操作的并发度，加快处理速度

#### 3.2 实现步骤

  先在mapper类中预先定义好小表，进行join

 引入实际场景中的解决方案：一次加载数据库或者用

##### Step 1：定义Mapper

```java
public class MapJoinMapper extends Mapper<LongWritable,Text,Text,Text>{
    private HashMap<String, String> map = new HashMap<>();

    //第一件事情:将分布式缓存的小表数据读取到本地Map集合(只需要做一次)

    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        //1:获取分布式缓存文件列表
        URI[] cacheFiles =  context.getCacheFiles();

        //2:获取指定的分布式缓存文件的文件系统(FileSystem)
        FileSystem fileSystem = FileSystem.get(cacheFiles[0], context.getConfiguration());

        //3:获取文件的输入流
        FSDataInputStream inputStream = fileSystem.open(new Path(cacheFiles[0]));

        //4:读取文件内容, 并将数据存入Map集合
           //4.1 将字节输入流转为字符缓冲流FSDataInputStream --->BufferedReader
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
           //4.2 读取小表文件内容,以行位单位,并将读取的数据存入map集合


        String line = null;
        while((line = bufferedReader.readLine()) != null){
            String[] split = line.split(",");

            map.put(split[0], line);

        }


        //5:关闭流
        bufferedReader.close();
        fileSystem.close();


    }

    //第二件事情:对大表的处理业务逻辑,而且要实现大表和小表的join操作

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            //1:从行文本数据中获取商品的id: p0001 , p0002  得到了K2
            String[] split = value.toString().split(",");
            String productId = split[2];  //K2

            //2:在Map集合中,将商品的id作为键,获取值(商品的行文本数据) ,将value和值拼接,得到V2
            String productLine = map.get(productId);
            String valueLine = productLine+"\t"+value.toString(); //V2
            //3:将K2和V2写入上下文中
            context.write(new Text(productId), new Text(valueLine));
    }
}

```

##### Step 2：定义主类

```java
public class JobMain  extends Configured implements Tool{
    @Override
    public int run(String[] args) throws Exception {
        //1:获取job对象
        Job job = Job.getInstance(super.getConf(), "map_join_job");

        //2:设置job对象(将小表放在分布式缓存中)
            //将小表放在分布式缓存中
           // DistributedCache.addCacheFile(new URI("hdfs://node01:8020/cache_file/product.txt"), super.getConf());
           job.addCacheFile(new URI("hdfs://node01:8020/cache_file/product.txt"));

           //第一步:设置输入类和输入的路径
            job.setInputFormatClass(TextInputFormat.class);
            TextInputFormat.addInputPath(job, new Path("file:///D:\\input\\map_join_input"));
            //第二步:设置Mapper类和数据类型
            job.setMapperClass(MapJoinMapper.class);
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(Text.class);

            //第八步:设置输出类和输出路径
            job.setOutputFormatClass(TextOutputFormat.class);
            TextOutputFormat.setOutputPath(job, new Path("file:///D:\\out\\map_join_out"));


        //3:等待任务结束
        boolean bl = job.waitForCompletion(true);
        return bl ? 0 :1;
    }

    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();
        //启动job任务
        int run = ToolRunner.run(configuration, new JobMain(), args);
        System.exit(run);
    }
}
```

  

## 4. 案例:求共同好友

分析图:

![img](/images/MapReduce/5-求共同好友.bmp)

### 4.1 需求分析

以下是qq的好友列表数据，冒号前是一个用户，冒号后是该用户的所有好友（数据中的好友关系是单向的）

```java
A:B,C,D,F,E,O
B:A,C,E,K
C:A,B,D,E,I 
D:A,E,F,L
E:B,C,D,M,L
F:A,B,C,D,E,O,M
G:A,C,D,E,F
H:A,C,D,E,O
I:A,O
J:B,O
K:A,C,D
L:D,E,F
M:E,F,G
O:A,H,I,J
```

求出哪些人两两之间有共同好友，及他俩的共同好友都有谁？

### 4.2 实现步骤

##### 第一步：代码实现

**Mapper类**

```java
public class Step1Mapper extends Mapper<LongWritable,Text,Text,Text> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
         //1:以冒号拆分行文本数据: 冒号左边就是V2
        String[] split = value.toString().split(":");
        String userStr = split[0];

        //2:将冒号右边的字符串以逗号拆分,每个成员就是K2
        String[] split1 = split[1].split(",");
        for (String s : split1) {
            //3:将K2和v2写入上下文中
            context.write(new Text(s), new Text(userStr));
        }
    }
}
```

**Reducer类:**

```java
public class Step1Reducer extends Reducer<Text,Text,Text,Text> {
    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        //1:遍历集合,并将每一个元素拼接,得到K3
        StringBuffer buffer = new StringBuffer();

        for (Text value : values) {
            buffer.append(value.toString()).append("-");
        }
        //2:K2就是V3
        //3:将K3和V3写入上下文中
        context.write(new Text(buffer.toString()), key);
    }
}
```

**JobMain:**

```java
public class JobMain extends Configured implements Tool {
    @Override
    public int run(String[] args) throws Exception {
        //1:获取Job对象
        Job job = Job.getInstance(super.getConf(), "common_friends_step1_job");

        //2:设置job任务
            //第一步:设置输入类和输入路径
            job.setInputFormatClass(TextInputFormat.class);
            TextInputFormat.addInputPath(job, new Path("file:///D:\\input\\common_friends_step1_input"));

            //第二步:设置Mapper类和数据类型
            job.setMapperClass(Step1Mapper.class);
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(Text.class);

            //第三,四,五,六

            //第七步:设置Reducer类和数据类型
            job.setReducerClass(Step1Reducer.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);

            //第八步:设置输出类和输出的路径
            job.setOutputFormatClass(TextOutputFormat.class);
            TextOutputFormat.setOutputPath(job, new Path("file:///D:\\out\\common_friends_step1_out"));

        //3:等待job任务结束
        boolean bl = job.waitForCompletion(true);


        return bl ? 0: 1;
    }

    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();

        //启动job任务
        int run = ToolRunner.run(configuration, new JobMain(), args);

        System.exit(run);
    }
}
```

##### 第二步：代码实现

**Mapper类**

```java
public class Step2Mapper extends Mapper<LongWritable,Text,Text,Text> {
    /*
     K1           V1

     0            A-F-C-J-E-	B
    ----------------------------------

     K2             V2
     A-C            B
     A-E            B
     A-F            B
     C-E            B

     */
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //1:拆分行文本数据,结果的第二部分可以得到V2
        String[] split = value.toString().split("\t");
        String   friendStr =split[1];

        //2:继续以'-'为分隔符拆分行文本数据第一部分,得到数组
        String[] userArray = split[0].split("-");

        //3:对数组做一个排序
        Arrays.sort(userArray);

        //4:对数组中的元素进行两两组合,得到K2
        /*
          A-E-C ----->  A  C  E

          A  C  E
            A  C  E

         */
        for (int i = 0; i <userArray.length -1 ; i++) {
            for (int j = i+1; j  < userArray.length ; j++) {
                //5:将K2和V2写入上下文中
                context.write(new Text(userArray[i] +"-"+userArray[j]), new Text(friendStr));
            }

        }

    }
}

```

**Reducer类:**

```java
public class Step2Reducer extends Reducer<Text,Text,Text,Text> {
    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        //1:原来的K2就是K3
        //2:将集合进行遍历,将集合中的元素拼接,得到V3
        StringBuffer buffer = new StringBuffer();
        for (Text value : values) {
            buffer.append(value.toString()).append("-");
            
        }
        //3:将K3和V3写入上下文中
        context.write(key, new Text(buffer.toString()));
    }
}
```

**JobMain:**

```java
public class JobMain extends Configured implements Tool {
    @Override
    public int run(String[] args) throws Exception {
        //1:获取Job对象
        Job job = Job.getInstance(super.getConf(), "common_friends_step2_job");

        //2:设置job任务
            //第一步:设置输入类和输入路径
            job.setInputFormatClass(TextInputFormat.class);
            TextInputFormat.addInputPath(job, new Path("file:///D:\\out\\common_friends_step1_out"));

            //第二步:设置Mapper类和数据类型
            job.setMapperClass(Step2Mapper.class);
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(Text.class);

            //第三,四,五,六

            //第七步:设置Reducer类和数据类型
            job.setReducerClass(Step2Reducer.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);

            //第八步:设置输出类和输出的路径
            job.setOutputFormatClass(TextOutputFormat.class);
            TextOutputFormat.setOutputPath(job, new Path("file:///D:\\out\\common_friends_step2_out"));

        //3:等待job任务结束
        boolean bl = job.waitForCompletion(true);
        return bl ? 0: 1;
    }

    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();
        //启动job任务
        int run = ToolRunner.run(configuration, new JobMain(), args);
        System.exit(run);
    }
}
```



## 1. 自定义InputFormat合并小文件

![img](/images/MapReduce/1-自定义InputFormat.bmp)

### **1.1 需求**

无论hdfs还是mapreduce，对于小文件都有损效率，实践中，又难免面临处理大量小文件的场景，此时，就需要有相应解决方案

### **1.2 分析**

小文件的优化无非以下几种方式：

1、 在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS

2、 在业务处理之前，在HDFS上使用mapreduce程序对小文件进行合并

3、 在mapreduce处理时，可采用combineInputFormat提高效率

### **1.3 实现**

本节实现的是上述第二种方式

程序的核心机制：

自定义一个InputFormat

改写RecordReader，实现一次读取一个完整文件封装为KV

在输出时使用SequenceFileOutPutFormat输出合并文件

代码如下：

#### **自定义InputFromat**

```java
public class MyInputFormat extends FileInputFormat<NullWritable,BytesWritable> {
    @Override
    public RecordReader<NullWritable, BytesWritable> createRecordReader(InputSplit inputSplit, TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
        //1:创建自定义RecordReader对象
        MyRecordReader myRecordReader = new MyRecordReader();
        //2:将inputSplit和context对象传给MyRecordReader
        myRecordReader.initialize(inputSplit, taskAttemptContext);
        return myRecordReader;
    }

    /*
     设置文件是否可以被切割
     */
    @Override
    protected boolean isSplitable(JobContext context, Path filename) {
        return false;
    }
}

```

#### **自定义**RecordReader

```java
public class MyRecordReader extends RecordReader<NullWritable,BytesWritable>{

    private Configuration configuration = null;
    private  FileSplit fileSplit = null;
    private boolean processed = false;
    private BytesWritable bytesWritable = new BytesWritable();
    private  FileSystem fileSystem = null;
    private  FSDataInputStream inputStream = null;
    //进行初始化工作
    @Override
    public void initialize(InputSplit inputSplit, TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
        //获取文件的切片
          fileSplit= (FileSplit)inputSplit;

        //获取Configuration对象
         configuration = taskAttemptContext.getConfiguration();
    }

    //该方法用于获取K1和V1
    /*
     K1: NullWritable
     V1: BytesWritable
     */
    @Override
    public boolean nextKeyValue() throws IOException, InterruptedException {
        if(!processed){
            //1:获取源文件的字节输入流
            //1.1 获取源文件的文件系统 (FileSystem)
             fileSystem = FileSystem.get(configuration);
            //1.2 通过FileSystem获取文件字节输入流
             inputStream = fileSystem.open(fileSplit.getPath());

            //2:读取源文件数据到普通的字节数组(byte[])
            byte[] bytes = new byte[(int) fileSplit.getLength()];
            IOUtils.readFully(inputStream, bytes, 0, (int)fileSplit.getLength());

            //3:将字节数组中数据封装到BytesWritable ,得到v1

            bytesWritable.set(bytes, 0, (int)fileSplit.getLength());

            processed = true;

            return true;
        }

        return false;
    }

    //返回K1
    @Override
    public NullWritable getCurrentKey() throws IOException, InterruptedException {
        return NullWritable.get();
    }

    //返回V1
    @Override
    public BytesWritable getCurrentValue() throws IOException, InterruptedException {
        return bytesWritable;
    }

    //获取文件读取的进度
    @Override
    public float getProgress() throws IOException, InterruptedException {
        return 0;
    }

    //进行资源释放
    @Override
    public void close() throws IOException {
        inputStream.close();
        fileSystem.close();
    }
}

```

#### Mapper类:

```java
public class SequenceFileMapper extends Mapper<NullWritable,BytesWritable,Text,BytesWritable> {
    @Override
    protected void map(NullWritable key, BytesWritable value, Context context) throws IOException, InterruptedException {
        //1:获取文件的名字,作为K2
        FileSplit fileSplit = (FileSplit) context.getInputSplit();
        String fileName = fileSplit.getPath().getName();

        //2:将K2和V2写入上下文中
        context.write(new Text(fileName), value);
    }
}
```



#### 主类:

```java
public class JobMain extends Configured implements Tool {
    @Override
    public int run(String[] args) throws Exception {
        //1:获取job对象
        Job job = Job.getInstance(super.getConf(), "sequence_file_job");

        //2:设置job任务
            //第一步:设置输入类和输入的路径
            job.setInputFormatClass(MyInputFormat.class);
            MyInputFormat.addInputPath(job, new Path("file:///D:\\input\\myInputformat_input"));

            //第二步:设置Mapper类和数据类型
            job.setMapperClass(SequenceFileMapper.class);
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(BytesWritable.class);

            //第七步: 不需要设置Reducer类,但是必须设置数据类型
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(BytesWritable.class);

            //第八步:设置输出类和输出的路径
            job.setOutputFormatClass(SequenceFileOutputFormat.class);
            SequenceFileOutputFormat.setOutputPath(job, new Path("file:///D:\\out\\myinputformat_out"));


        //3:等待job任务执行结束
        boolean bl = job.waitForCompletion(true);
        return bl ? 0 : 1;
    }

    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();

        int run = ToolRunner.run(configuration, new JobMain(), args);

        System.exit(run);
    }
}
```





## 2. 自定义outputFormat

![img](/images/MapReduce/2-自定义OutputFormat.bmp)

### **2.1** **需求**

现在有一些订单的评论数据，需求，将订单的好评与差评进行区分开来，将最终的数据分开到不同的文件夹下面去，数据内容参见资料文件夹，其中数据第九个字段表示好评，中评，差评。0：好评，1：中评，2：差评

### **2.2 分析**

程序的关键点是要在一个mapreduce程序中根据数据的不同输出两类结果到不同目录，这类灵活的输出需求可以通过自定义outputformat来实现

### **2.3 实现**

实现要点：

1、 在mapreduce中访问外部资源

2、 自定义outputformat，改写其中的recordwriter，改写具体输出数据的方法write()

#### **第一步**：自定义MyOutputFormat

**MyOutputFormat类:**

```java
public class MyOutputFormat extends FileOutputFormat<Text,NullWritable> {
    @Override
    public RecordWriter<Text, NullWritable> getRecordWriter(TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
        //1:获取目标文件的输出流(两个)
        FileSystem fileSystem = FileSystem.get(taskAttemptContext.getConfiguration());
        FSDataOutputStream goodCommentsOutputStream = fileSystem.create(new Path("file:///D:\\out\\good_comments\\good_comments.txt"));
        FSDataOutputStream badCommentsOutputStream = fileSystem.create(new Path("file:///D:\\out\\bad_comments\\bad_comments.txt"));

        //2:将输出流传给MyRecordWriter
        MyRecordWriter myRecordWriter = new MyRecordWriter(goodCommentsOutputStream,badCommentsOutputStream);

        return myRecordWriter;
    }
}

```

**MyRecordReader类:**

```java
public class MyRecordWriter extends RecordWriter<Text,NullWritable> {
    private FSDataOutputStream goodCommentsOutputStream;
    private FSDataOutputStream badCommentsOutputStream;

    public MyRecordWriter() {
    }

    public MyRecordWriter(FSDataOutputStream goodCommentsOutputStream, FSDataOutputStream badCommentsOutputStream) {
        this.goodCommentsOutputStream = goodCommentsOutputStream;
        this.badCommentsOutputStream = badCommentsOutputStream;
    }

    /**
     *
     * @param text  行文本内容
     * @param nullWritable
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    public void write(Text text, NullWritable nullWritable) throws IOException, InterruptedException {
        //1:从行文本数据中获取第9个字段
        String[] split = text.toString().split("\t");
        String numStr = split[9];

        //2:根据字段的值,判断评论的类型,然后将对应的数据写入不同的文件夹文件中
        if(Integer.parseInt(numStr) <= 1){
            //好评或者中评
            goodCommentsOutputStream.write(text.toString().getBytes());
            goodCommentsOutputStream.write("\r\n".getBytes());
        }else{
            //差评
            badCommentsOutputStream.write(text.toString().getBytes());
            badCommentsOutputStream.write("\r\n".getBytes());
        }

    }

    @Override
    public void close(TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
        IOUtils.closeStream(goodCommentsOutputStream);
        IOUtils.closeStream(badCommentsOutputStream);
    }
}
```

#### **第二步**：自定义Mapper类

```java
public class MyOutputFormatMapper extends Mapper<LongWritable,Text,Text,NullWritable> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        context.write(value, NullWritable.get());
    }
}
```

#### 第三步:主类JobMain

```java
public class JobMain extends Configured implements Tool {
    @Override
    public int run(String[] args) throws Exception {
        //1:获取job对象
        Job job = Job.getInstance(super.getConf(), "myoutputformat_job");

        //2:设置job任务
            //第一步:设置输入类和输入的路径
            job.setInputFormatClass(TextInputFormat.class);
            TextInputFormat.addInputPath(job, new Path("file:///D:\\input\\myoutputformat_input"));

            //第二步:设置Mapper类和数据类型
            job.setMapperClass(MyOutputFormatMapper.class);
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(NullWritable.class);

            //第八步:设置输出类和输出的路径
            job.setOutputFormatClass(MyOutputFormat.class);
            MyOutputFormat.setOutputPath(job, new Path("file:///D:\\out\\myoutputformat_out"));


        //3:等待任务结束
        boolean bl = job.waitForCompletion(true);
        return bl ? 0 : 1;
    }

    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();
        int run = ToolRunner.run(configuration, new JobMain(), args);
        System.exit(run);
    }
}
```



## 3. 自定义分组求取topN

![img](/images/MapReduce/3-自定义分组求TopN.bmp)

分组是mapreduce当中reduce端的一个功能组件，主要的作用是决定哪些数据作为一组，调用一次reduce的逻辑，默认是每个不同的key，作为多个不同的组，每个组调用一次reduce逻辑，我们可以自定义分组实现不同的key作为同一个组，调用一次reduce逻辑

### **3.1 需求**

有如下订单数据

| 订单id          | 商品id   | 成交金额  |
| ------------- | ------ | ----- |
| Order_0000001 | Pdt_01 | 222.8 |
| Order_0000001 | Pdt_05 | 25.8  |
| Order_0000002 | Pdt_03 | 522.8 |
| Order_0000002 | Pdt_04 | 122.4 |
| Order_0000002 | Pdt_05 | 722.4 |
| Order_0000003 | Pdt_01 | 222.8 |



现在需要求出每一个订单中成交金额最大的一笔交易 

### **3.2 分析**

1、利用“订单id和成交金额”作为key，可以将map阶段读取到的所有订单数据按照id分区，按照金额排序，发送到reduce

2、在reduce端利用分组将订单id相同的kv聚合成组，然后取第一个即是最大值

### **3.3 实现**

#### **第一步:**定义OrderBean

定义一个OrderBean，里面定义两个字段，第一个字段是我们的orderId，第二个字段是我们的金额（注意金额一定要使用Double或者DoubleWritable类型，否则没法按照金额顺序排序）

```java
public class OrderBean  implements WritableComparable<OrderBean>{
    private  String orderId;
    private  Double price;

    public String getOrderId() {
        return orderId;
    }

    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }

    public Double getPrice() {
        return price;
    }

    public void setPrice(Double price) {
        this.price = price;
    }

    @Override
    public String toString() {
        return  orderId + "\t" + price;
    }

    //指定排序规则
    @Override
    public int compareTo(OrderBean orderBean) {
        //先比较订单ID,如果订单ID一致,则排序订单金额(降序)
        int i = this.orderId.compareTo(orderBean.orderId);
        if(i == 0){
            i = this.price.compareTo(orderBean.price) * -1;
        }

        return i;
    }

    //实现对象的序列化
    @Override
    public void write(DataOutput out) throws IOException {
         out.writeUTF(orderId);
         out.writeDouble(price);
    }

    //实现对象反序列化
    @Override
    public void readFields(DataInput in) throws IOException {
        this.orderId = in.readUTF();
        this.price  = in.readDouble();
    }
}
```

#### 第二步: 定义Mapper类

```java
public class GroupMapper extends Mapper<LongWritable,Text,OrderBean,Text> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //1:拆分行文本数据,得到订单的ID,订单的金额
        String[] split = value.toString().split("\t");

        //2:封装OrderBean,得到K2
        OrderBean orderBean = new OrderBean();
        orderBean.setOrderId(split[0]);
        orderBean.setPrice(Double.valueOf(split[2]));

        //3:将K2和V2写入上下文中
        context.write(orderBean, value);
    }
}
```

#### **第三步:**自定义分区

自定义分区，按照订单id进行分区，把所有订单id相同的数据，都发送到同一个reduce中去

```java
public class OrderPartition extends Partitioner<OrderBean,Text> {
    //分区规则: 根据订单的ID实现分区

    /**
     *
     * @param orderBean K2
     * @param text  V2
     * @param i  ReduceTask个数
     * @return 返回分区的编号
     */
    @Override
    public int getPartition(OrderBean orderBean, Text text, int i) {
        return (orderBean.getOrderId().hashCode() & 2147483647) % i;
    }
}

```



#### **第四步:**自定义分组

按照我们自己的逻辑进行分组，通过比较相同的订单id，将相同的订单id放到一个组里面去，进过分组之后当中的数据，已经全部是排好序的数据，我们只需要取前topN即可

```java
// 1: 继承WriteableComparator
public class OrderGroupComparator extends WritableComparator {
    // 2: 调用父类的有参构造
    public OrderGroupComparator() {
        super(OrderBean.class,true);
    }

    //3: 指定分组的规则(重写方法)
    @Override
    public int compare(WritableComparable a, WritableComparable b) {
        //3.1 对形参做强制类型转换
        OrderBean first = (OrderBean)a;
        OrderBean second = (OrderBean)b;

        //3.2 指定分组规则
        return first.getOrderId().compareTo(second.getOrderId());
    }
}
```

#### 第五步:定义Reducer类

```java
public class GroupReducer extends Reducer<OrderBean,Text,Text,NullWritable> {
    @Override
    protected void reduce(OrderBean key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        int i = 0;
        //获取集合中的前N条数据
        for (Text value : values) {
            context.write(value, NullWritable.get());
            i++;
            if(i >= 1){
                break;
            }
        }
    }
}
```



#### **第六步:**程序main函数入口

```java
public class JobMain extends Configured implements Tool {
    @Override
    public int run(String[] args) throws Exception {
        //1:获取Job对象
        Job job = Job.getInstance(super.getConf(), "mygroup_job");

        //2:设置job任务
            //第一步:设置输入类和输入路径
            job.setInputFormatClass(TextInputFormat.class);
            TextInputFormat.addInputPath(job, new Path("file:///D:\\input\\mygroup_input"));

            //第二步:设置Mapper类和数据类型
            job.setMapperClass(GroupMapper.class);
            job.setMapOutputKeyClass(OrderBean.class);
            job.setMapOutputValueClass(Text.class);

            //第三,四,五,六
            //设置分区
            job.setPartitionerClass(OrderPartition.class);
            //设置分组
            job.setGroupingComparatorClass(OrderGroupComparator.class);

            //第七步:设置Reducer类和数据类型
            job.setReducerClass(GroupReducer.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(NullWritable.class);

            //第八步:设置输出类和输出的路径
            job.setOutputFormatClass(TextOutputFormat.class);
            TextOutputFormat.setOutputPath(job, new Path("file:///D:\\out\\mygroup_out"));

        //3:等待job任务结束
        boolean bl = job.waitForCompletion(true);



        return bl ? 0: 1;
    }

    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();

        //启动job任务
        int run = ToolRunner.run(configuration, new JobMain(), args);

        System.exit(run);
    }
}

```





 

 

