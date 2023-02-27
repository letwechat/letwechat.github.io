---
title: 交易反欺诈离线计算的工程化
tags: 系统设计 交易反欺诈 离线计算
---

# 交易反欺诈离线计算的工程化

# 概述

离线特征计算，顾名思义，就是用SQL的方式用跑批量的形式来提取用户特征数据，因为属于离线所以这些特征的实时性要求就没有那么高，一般能达到T+1就可以了。在交易反欺诈系统中，离线特征无论是数量，还是在特征中的占比都相对来说比较大，而且跑批的时间也比较长，一般从凌晨2:30开始，大概要执行3.5个小时，涉及到将近10个任务：

![Untitled](/assets/images/blogs/20230227/Untitled.png)

对于离线SQL计算出来的结果，都需要输出至特征存储（Redis集群）中，输出过程大同小异，但存在很多共同的代码，如果将其泛化，抽象出本质，建立高内聚的代码结构。

既然是跑SQL，同样也是基于数据表的，只不是表数据不是在关系型数据库而是在大数据环境中，为提升整体跑批性能，会在前后做一些特定的优化，交易反欺诈系统中的离线分析步骤简单概述如下：

![Untitled](/assets/images/blogs/20230227/Untitled%201.png)

1. 大数据离线分析时在Spark环境上运行，需要建立SparkSession会话，将对应分布式文件系统数据（HDFS）依据相关的格式（Parquet列存储、CSV行存储），建立DataSet以备后续分析；
2. 对文件系统中的数据，可能存在冗余（无效列，错误干扰数据），进行多行数据合并（交易+回传数据合并）单表预处理；
3. 对于多表联合查询的语句（如手机银行和网银等关联渠道数据），为减少后续分析SQL中出现的多个union以及子查询，做临时合并表，简化后续SQL语句的编写；
4. 对于每一类特征，执行SparkSQL，根据数据量预估分片数量（repartition count），支持动态配置的executor；
5. 执行后的SparkSQL结果集Row（类似SQL API中的ResultSet），执行get操作并根据业务导出至Redis数据结构中。

工程角度的基本类图如下，后续章节将会分步骤描述过程：

![Untitled](/assets/images/blogs/20230227/Untitled%202.png)

# 1. 离线分析数据准备

## 1.1 文件读取，临时表创建

Spark引擎擅长的是高效分布式计算，并没有像Hive中存在的元数据管理，因此在执行纯Spark任务时，需要根据本次Spark会话建立临时表，建立的临时表在多个Session中并不进行共享，在本次Session使用完成后即失效，由于我们是使用SparkSQL作为基础的离线分析工具，自然也离不开其核心API：Dataset<Row>。

在离线分析的第一步就是做数据准备工作，在SQL执行之前，SparkSession需要创建Dataset<Row>，具体代码可以参考下面的片段：

```java
List<SparkTempTable> sparkTempTables = getSparkTempTables();
IDatasetTableCreator datasetTableCreator = DATASET_TABLE_CREATOR_MAP.get(offlineCalculateContext.getDatasetCreator());
datasetTableCreator.createSparkTempTables(sparkSession, sparkTempTables);
doCreateMergedTempTable(sparkSession);
```

对于Parquet列式存储的临时表创建实现HdfsDatasetTableCreator，其概要过程如下：

![Untitled](/assets/images/blogs/20230227/Untitled%203.png)

.parquet()方法指明其读取的数据文件为parquet列式存储文件，其参数为分布式文件系统HDFS路径，读取后的Dataset<Row>对象就可以用createOrReplaceView()来创建临时表了，需要注意的是该临时表仅在该Session中有效，当任务执行完成或异常结束时就不存在了，不同的Session使用相同名称的临时表也是可行的，临时表在Session之间是非共享且不可见的。

当然SparkSession能够支持和创建的类型也并非这一种，还支持：

- 文本文件：.textFile，.wholeTextFiles，纯文本文件，但不适合直接分析
- JSON格式文件：.read.json读取JSON格式文件
- CSV文件：支持单分隔符的文本文件
- SequenceFile：序列化文件
- Kudu：三方库来实现，目前在行内使用，作为分布式数据库，替代MySQL

为保障执行性能，在本系统中使用最多的是Parquet文件和Kudu文件，如果两者的schema相同，甚至支持动态切换。

## 1.2 单数据表预处理

对于流水数据文件，其交易数据和回传数据是放到同一张数据文件表中的，传统关系型数据库是支持根据流水号进行更新的操作来保证数据表中主键或唯一索引的唯一性，便于后续的数据分析；但关系型数据库受限于其应用场景，无法支撑如此大数据批量的写入操作和海量存储的需求，因此交易反欺诈系统中，使用的是parquet列式存储文件格式，接收Kafka消息队列的数据，每30分钟汇总数据文件，将所有记录append至文件中。

这就导致了一个问题，交易数据和回传数据对应数据文件的两条记录，此时如果仍然按照传统SQL语句方式分析，就需要关联子查询，建立临时表的方式，不仅会使得SQL语句非常臃肿，而且还造成了每次SQL执行时都需要合并数据的重复操作。

因此，经过分析和重新设计，将常用的交易数据表进行预处理，使得经过预处理后的表能以类似传统数据库中的数据一样，每个流水号仅对应一条交易流水记录，并记录最终的交易状态。

### 简单逻辑：SQL实现或Dataset改造

如果是简单的去除不必要的列，或是可通过SQL表示出来的操作，可以直接基于Dataset<Row>对象来进行处理，如下面代码只选取部分被用到的字段列表：

![Untitled](/assets/images/blogs/20230227/Untitled%204.png)

但当逻辑复杂到已经不能用SQL表示了，例如合并后根据时间顺序找出最终状态，并去除某些干扰数据和时间顺序上不可能出现的逻辑，就需要借助更为复杂的API编程了。

### 复杂逻辑：基于JavaRDD改造

SQL的语义毕竟表达能力有限，如果业务逻辑稍微复杂，就得考虑使用代码来实现了，好在Spark的API：Dataset<Row>能够很容易地与底层API：RDD实现互相转换。

在某业务场景中，就根据渠道特定的业务逻辑合并数据，并将其转换回Dataset，这样一来需要特殊处理的SQL被省去了，无论是性能上，还是SQL可读性和结果准确性上都有着极大的提升。

![Untitled](/assets/images/blogs/20230227/Untitled%205.png)

## 1.3 多数据表合并

以上的内容分析只涉及到单个数据表，在机器模型的应用场景中，模型的特征往往会涉及到多渠道数据表之间的关联，如果每个SQL都需要做union操作，合并成临时表再进行处理，无疑也是耗时的，因此在单表建立完成后，需要对多个数据表进行合并，并.cache()以提升SQL可读性。

与单表合并类似，多数据表合并也可根据复杂程度的不同，选择SQL或RDD两种方式处理。

![Untitled](/assets/images/blogs/20230227/Untitled%206.png)

# 2. 执行SQL并输出

在所有的数据表准备完成后，就可以开始执行SQL，进行数据分析了。在Spark环境中，SQL执行是最简单的，因为它只需要一行代码：

```java
sparkSession.sql(getExecuteSql()).repartition(getRepartitionCount()).foreachPartition(
     new PartitionRowBasedCalculationFunc(redisExecContext, getOperations(), getChannelName(), getName(), execAccumulator)
        );
```

但它同时也是最为复杂的，因为你不知道其简单的背后隐藏着多少处理逻辑和优化策略，关于这里面的优化因篇幅有限本文不进行详细阐述。

## 2.1 数据输出的常见处理方式

经过SQL执行后的结果类型为：`org.apache.spark.sql.Row`，我们需要将Row中的数据适当地输出至存储介质，由于我们是实时风控系统，所以采用的是内存型数据库Redis。需要注意的是，我们这里编程使用的是分布式API，数据是按照分片在不同的Spark Executor端中(而非Driver)执行的，绝对不能做全量数据处理的保障，类似在内存中进行聚合计算总数这种操作是行不通的，如果想要统计数据的条目，更多地考虑Spark提供的累加器(Accumulator)API。

Row对象的查询String类型方法: getAs(String)，其API设计有点类似SQL结果集（ResultSet），需要将结果集进行遍历输出，如果我们手动编写处理`org.apache.spark.sql.Row`的方法，代码样式大概是这样的：

```java
protected void writeHandler(Iterator<Row> rows, JedisClusterPipeline jcp){
	final long begin = 0L;
  final long end = DateTime.now().withTimeAtStartOfDay().plusDays(1).getMills() / 1000L;
  rows.foreach(new JavaForeachFunc(){
    @Override
    public void call(Row row){
      jcp.zadd("S2" + row.getAs("cif_no"), row.getAs("value"), row.getAs("trans_time"));
			jcp.zremrangeByScore("S1" + row.getAs("cif_no"), begin, end);
		}

  });
}
```

但试想我们有着上百个SQL以及将其根据字段写入至Redis的不同数据结构（Redis的主要数据结构包括：String, Set, ZSet, Hash, List）， 代码量无疑是惊人的，且很难维护，那么该如何解决这一问题，既便于代码编写，又不损失执行性能，还能方便地静态化查看特征逻辑呢？

## 2.1 泛化，统一类型处理

通过对离线分析过程的分析，能够看出每个离线特征只做两件事情：

1. 执行SparkSQL；
2. 将执行结果导出至Redis；

绝大多数场景是，需要从Row中抽取出对应select语句中的field并根据某种Redis操作type导出到Redis，例如String类型的Set、Hash类型的HSet、ZSet类型的ZAdd等。

假如能够将SQL中的field和具体操作对应起来，将会极大加速研发过程，最终达到下面的目标：

![Untitled](/assets/images/blogs/20230227/Untitled%207.png)

在executeSql中选出的字段，每条数据分别进行 Del，Sadd, Expire 操作，比如Del，就会删除“前缀+ payee_acno字段值”的key，这样就容易理解了。

明确目标之后，就需要我们考虑如何实现将这种操作进行泛化，统一化处理，首先对Row进行操作，为了提升写入Redis的性能，使用了流水线技术Pipeline，遍历每个Partiton（分片）的Iterator<Row>：

```java
this.redisExecContext.setPipeline(new JedisClusterPipeline());
it.foreach(new JavaForEachFuncWithAccumulator(new JavaForeachFunc() {
    @Override
    public void call(Row row) {
        operations.forEach(iRedisOperation -> iRedisOperation.doRedisOperation(row, redisExecContext));
    }
}, execAccumulator));
redisExecContext.getPipeline().sync();
redisExecContext.getPipeline().close();
redisExecContext.setPipeline(null);
```

在正常的字符串与变量之间，我们使用 ${变量名} 作为运行时Row值判断，这就需要在任务运行前，手动编译好变量位置以及需要的变量值，使用Java的正则表达式来实现：

```java
public static final String VAR_STRING = "\\$\\{([a-zA-z0-9]+)}";
public static final Pattern VAR_PATTERN = Pattern.compile(VAR_STRING);

/**
 * 传入的Field，带 ${变量名}，将变量提取出来
 * @param importFields - “xxx${变量1}x${变量2}y”
 * @return - 转换为 ImportVariable(formatString="xxx%sx%sy", fields=["变量1",“变量2”])
 */
protected ImportVariable generateVariable(String importFields) {
    if (importFields == null) {
        return null;
    }
    Matcher matcher = VAR_PATTERN.matcher(importFields);
    List<String> vars = new ArrayList<>();
    while (matcher.find()) {
        vars.add(matcher.group(1));
    }
    return ImportVariable.builder()
            .formatString(importFields.replaceAll(VAR_STRING, "%s"))
            .fields(vars).build();
}
```

生成ImportVariable装入器对象，该对象只是针对这个动作的一个描述，只有在运行时才会根据Row行获取变量值使用String.format(%s,var…)的方式进行替换：

```java
/**
 * 将编译好的ImportVariable根据Row数据转换成对应的值，使用String.format方法
 * @param importVariable
 * @param variables - 借助Row.getAs方法
 * @return - 生成的运行时值
 */
protected String convertedVariable(ImportVariable importVariable, Row variables) {
    if (importVariable == null) {
        return null;
    }
    return String.format(importVariable.getFormatString(),
            importVariable.getFields().stream().map(variables::getAs).toArray(Object[]::new));
}
```

## 2.2 组合，简化程序逻辑

借助上一章节介绍的运行时转化技术，结合目前离线分析过程中可能进行的Redis操作，来设计并实现可组合，可叠加的针对单个SQL运行结果的多输出策略。

例如我们要对SQL跑取的客户号cif_no某个特征进行先Del再Hset并设置Expire，那么Del，HSet，Expire就属于叠加操作，且它们之间除了执行顺序，无耦合关系，使用继承的方式设计抽象类型AbstractPatternBasedRowOperation，并组合ImportVariable，来实现单个操作，下面举例说明。

![Untitled](/assets/images/blogs/20230227/Untitled%208.png)

参考Redis的Del实现，在构造时根据传入的key进行编译，运行时先根据编译结果生成要生成的key值，使用Pipeline执行del：

```java
private final ImportVariable keyVariable;

public Del(String key) {
    this.keyVariable = generateVariable(key);
}

@Override
public void doRedisOperation(Row row, RedisExecContext redisExecContext) {
    redisExecContext.getPipeline().del(convertedVariable(keyVariable, row));
}
```

与Del类似，HSet会使用3个变量：key, field, value，分别编译和执行，由此可类推到所有Redis的命令上：

```java
private final ImportVariable keyVariable;
private final ImportVariable fieldVariable;
private final ImportVariable valueVariable;

public HSet(String key, String field, String value) {
    this.keyVariable = generateVariable(key);
    this.fieldVariable = generateVariable(field);
    this.valueVariable = generateVariable(value);
}

@Override
public void doRedisOperation(Row row, RedisExecContext redisExecContext) {
    redisExecContext.getPipeline().hset(
            convertedVariable(keyVariable, row),
            convertedVariable(fieldVariable, row),
            convertedVariable(valueVariable, row)
    );
}
```

注意，这样组合出来的操作，必然不可能是单例的，每次使用都要在各自特征中事先声明，至于大家担心使用正则表达式会不会影响效率的问题，其实编译只发生在构造过程中，即这里的 generateVariable方法，而在遍历Row的过程中是不会出现的，毕竟row.getAs是必须要调用的，而且String.format在性能上也不会拖后腿。

至此我们在编写离线特征时，可以通过组合 Del, HSet, Expire 的方式，对外部数据源（如有可能，不仅限于Redis）进行写入。

## 2.3 不仅仅取值，还要能执行简单的函数

我们自定义的${变量}，都是适用于直接从Spark Row结果集中取值，如果想要更复杂的处理，比如在变量上做一些计算，是实现不了的，或者只能自定义基于刚才提到类似Del，HSet等实现的特殊处理流程。

举个例子，String类型的SDS占用内存较大，我们在做Redis内存存储的空间优化经常要将其合并到Hash数据结构中，但由于使用JedisCluster集群结构，为避免压力都集中到单个Slot，还需要将Key值哈希分片化，引入类似下面的函数来对key进行处理：

![Untitled](/assets/images/blogs/20230227/Untitled%209.png)

如果从抽象的角度来看，row.getAs可以视为一种特殊取值操作，${} 可以看做是直接取值函数，既然可以取值，也能在Row.getAs外围做一些特定的函数调用，获取变量的正则表达式演进成 $函数名(变量)，为兼容原有应用，将 ImportVariable 从Java类演进成接口，只暴露 convertVariableToString(Row row)方法，构建过程中不仅记录变量，还需要记录函数，在类型中引入内部静态类型：FuncVarTuple（func, var）

![Untitled](/assets/images/blogs/20230227/Untitled%2010.png)

当然，函数不能是随意写的，需要实现注册，为满足兼容条件，空字符串仍然被认为是直接获取Row；而上面提到的 RedisShareUtil.getSharedRedisKey 方法被注册进来，在最终实现的 convertVariableToString(Row)方法中，需要对自定义函数进行调用，同时能够兼容原有 Row.getValue()：

![Untitled](/assets/images/blogs/20230227/Untitled%2011.png)

在离线特征的定义中，以下面这种方式使用 splitkey函数：

![Untitled](/assets/images/blogs/20230227/Untitled%2012.png)

## 2.4 考虑其他存储介质

除了写出至Redis存储介质，还有部分离线特征在计算完成后，连带去触发相关实时特征计算，这就需要往它们之间的纽带：Kafka消息队列中发送实时消息，去触发在Flink中部署的实时特征计算模块，因此最上面的IRowOperation接口还支持Kafka类型的实现，由于这种属于特殊情况，仍采用从Row对象中直接取值的方式来实现：

![Untitled](/assets/images/blogs/20230227/Untitled%2013.png)

但是发送Kafka存在一个问题，由于我们使用了Redis的Pipeline流水线技术，Pipeline为了提升Redis操作性能，会积累一定的命令后（当前设置=500）批量提交，这就导致了部分Kafka消息被Flink实时任务消费处理了，但Redis集群中并不存在任务的尴尬情况，因此还需要使用观察者模式将Pipeline的sync周期管理起来，特别定了3个接口：

- start(): SQL执行开始，注册观察者（一般为Kafka发送端）；
- finish()：SQL执行结束，注销观察者；
- doCallback()：Pipeline.sync()执行完成，累积在内存中的ProducerRecord统一发送，当然，这可能会导致一些问题，比如executor程序崩溃导致堆积的消息并没有完全被释放，但同时Pipeline也会丢失未同步的数据，好在大数据集群能够保证总是能够执行成功一次，只要保证后置任务的幂等性数据正确性就不会出问题。

![Untitled](/assets/images/blogs/20230227/Untitled%2014.png)

# 3. 离线特征文档化



至此，我们的离线特征执行程序都已经完成，其基本类结构图参考第1章节的UML类图。离线特征的逻辑都在代码中，如果能够在每次投产升级迭代后都能及时输出离线特征的文档表示，就可以在研发、数据分析、产品经理之间建立起良好连接的纽带，毕竟代码导出的文档是相对最为准确的（曾经尝试过各种调度部门成员更新文档的方式，但均已失败告终），而一键生成文档，甚至将文档输出定义在Jenkins Build任务中的方式，最能体现程序员的“懒”带来的产出。

`IChannelOfflineCalculation`: 离线计算渠道定义，每个渠道都包含多个计算过程（IComputeOperation）；

`IComputeOperation`：渠道（例如手机银行）单个特征计算，主要包括执行SQL和多个针对字段的操作（IRowOperation）；

`IRowOperation`：针对SQL的行处理器，实现类型为Del，HSet，Expire等具体Redis操作（也可以为写出Kafka数据）；

可以看出这三类接口之间是包含的关系，如果我们要将其文档化，可参考设计模式中的 Composite模式（组合模式），借助每个接口重写的 toString()方法，导出对应文档。

![Untitled](/assets/images/blogs/20230227/Untitled%2015.png)

举例说明，由于我们导出的文档是自动更新到wiki中的，类似MarkDown的格式，代码类似如下：

```java
//IChannelOfflineCalculation
@Override
public String toString() {
    StringBuilder result = new StringBuilder();
    result.append(String.format("Channel=%s", getName())).append(System.lineSeparator());
    List<IComputeOperation> computeOperations = getComputeOperations();
    for (IComputeOperation computeOperation : computeOperations) {
        result.append(computeOperation.getOutputDocument()).append(System.lineSeparator());
    }
    return result.toString();
}
//
@Override
public String toString() {
    List<String> result = getOperations().stream().map(Object::toString).collect(Collectors.toList());
    return String.format("|%s|%s|%s|%s|", getName(),
            getRepartitionCount(), getExecuteSql(),
            StringUtils.join(result, ","));
}
```

最终导出的文档格式如下，能够完整反映出离线特征的特性：

![Untitled](/assets/images/blogs/20230227/Untitled%2016.png)

# 4.总结

本文从交易反欺诈系统的典型离线业务需求出发，介绍了数据表构建、前期数据处理、SQL执行、数据导出、文档化过程，期间的工程化处理方式，聚焦核心业务逻辑，借助各种设计模式，如组合模式解决文档导出问题、策略模式加载不同类型数据源、模板方法模式指定渠道流程标准化处理方式并预留临时表处理，至此，一个简单的离线处理任务的框架就搭建完成了。

目前，该系统已经正常运行2年多，运行在上面的离线特征有将近200多个，其性能优异（并没有在文档中详细介绍，主要是SparkSQL调优，以及RedisPipeline改造等），尤其是扩展性能够满足业务建设的需要，随着技术的推进，只要核心的SQL和输出模式不做重大变更，能够做到无感知的技术升级（例如从SparkSQL升级至FlinkSQL，或替换为Hive等）。

至于交易反欺诈系统中同样重要的实时以及特征运算部分，后续也会安排进行详细讲解，谢谢大家的关注。
