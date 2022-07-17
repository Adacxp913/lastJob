# lastJob
# 期末作业

题目一： 分析一条 TPCDS SQL（请基于 Spark 3.1.1 版本解答）  
SQL 从中任意选择一条：
https://github.com/apache/spark/tree/master/sql/core/src/test/resources/tpcds
（1）运行该 SQL，如 q38，并截图该 SQL 的 SQL 执行图  
（2）该 SQL 用到了哪些优化规则（optimizer rules）   
（3）请各用不少于 200 字描述其中的两条优化规则   

帮助文档：如何运行该 SQL：

1. 从 github 下载 TPCDS 数据生成器
>git clone https://github.com/maropu/spark-tpcds-datagen.git  
>cd spark-tpcds-datagen
> 
2. 下载 Spark3.1.1 到 spark-tpcds-datagen 目录并解压
>wget https://archive.apache.org/dist/spark/spark-3.1.1/spark-3.1.1-bin-hadoop2.7.tgz  
>tar -zxvf spark-3.1.1-bin-hadoop2.7.tgz
> 
3. 生成数据
>mkdir -p tpcds-data-1g  
>export SPARK_HOME=./spark-3.1.1-bin-hadoop2.7 
>./bin/dsdgen --output-location tpcds-data-1g
> 
4. 下载三个 test jar 并放到当前目录
>wget https://repo1.maven.org/maven2/org/apache/spark/spark-catalyst_2.12/3.1.1/spark-catalyst_2.12-3.1.1-tests.jar  
>wget https://repo1.maven.org/maven2/org/apache/spark/spark-core_2.12/3.1.1/spark-core_2.12-3.1.1-tests.jar  
>wget https://repo1.maven.org/maven2/org/apache/spark/spark-sql_2.12/3.1.1/spark-sql_2.12-3.1.1-tests.jar  

5. 执行 SQL
>spark-submit --class org.apache.spark.sql.execution.benchmark.TPCDSQueryBenchmark --jars spark-core_2.12-3.1.1-tests.jar,spark-catalyst_2.12-3.1.1-tests.jar spark-sql_2.12-3.1.1-tests.jar --data-location tpcds-data-1g --query-filter "q73"

6. 执行结果  
```
1、下载spark、spark-tpcds-datagen（略）
2、生成数据 ./bin/dsdgen --output-location tpcds-data-1g 执行要求需要是java8，否则会失败
3、执行sql测试
  在wsl ubuntu中执行spark-submit无反应，改为直接在windows命令中执行
  其中要打印优化器信息，需增加 --conf "spark.sql.planChangeLog.level=WARN"
  spark-submit --class org.apache.spark.sql.execution.benchmark.TPCDSQueryBenchmark --conf "spark.sql.planChangeLog.level=WARN" --jars D:\\dev\\code\\other\\spark-tpcds-datagen\\spark-core_2.12-3.1.1-tests.jar,D:\\dev\\code\\other\\spark-tpcds-datagen\\spark-catalyst_2.12-3.1.1-tests.jar D:\\dev\\code\\other\\spark-tpcds-datagen\\spark-sql_2.12-3.1.1-tests.jar --data-location D:\\dev\\code\\other\\spark-tpcds-datagen\\tpcds-data-1g --query-filter "q73">>log.txt 2>&1 &

```
运行日志文件
```
  参见log.txt输出！
```
q73对应sql是：
```
  SELECT
  c_last_name,
  c_first_name,
  c_salutation,
  c_preferred_cust_flag,
  ss_ticket_number,
  cnt
FROM
  (SELECT
    ss_ticket_number,
    ss_customer_sk,
    count(*) cnt
  FROM store_sales, date_dim, store, household_demographics
  WHERE store_sales.ss_sold_date_sk = date_dim.d_date_sk
    AND store_sales.ss_store_sk = store.s_store_sk
    AND store_sales.ss_hdemo_sk = household_demographics.hd_demo_sk
    AND date_dim.d_dom BETWEEN 1 AND 2
    AND (household_demographics.hd_buy_potential = '>10000' OR
    household_demographics.hd_buy_potential = 'unknown')
    AND household_demographics.hd_vehicle_count > 0
    AND CASE WHEN household_demographics.hd_vehicle_count > 0
    THEN
      household_demographics.hd_dep_count / household_demographics.hd_vehicle_count
        ELSE NULL END > 1
    AND date_dim.d_year IN (1999, 1999 + 1, 1999 + 2)
    AND store.s_county IN ('Williamson County', 'Franklin Parish', 'Bronx County', 'Orange County')
  GROUP BY ss_ticket_number, ss_customer_sk) dj, customer
WHERE ss_customer_sk = c_customer_sk
  AND cnt BETWEEN 1 AND 5
ORDER BY cnt DESC
```
运用了哪些优化规则
```
  查找结果日志log.txt中 === Applying Rule  关键字，可以看到使用了哪些规则：
  ApplyColumnarRulesAndInsertTransitions 应用列式规则并插入过渡
  CollapseCodegenStages  折叠代码生成阶段
  CleanupAliases 清理别名
  ConstantFolding 常量折叠
  EnsureRequirements 确保要求
  EliminateSubqueryAliases 排除子查询
  InferFiltersFromConstraints 从约束生成附加过滤器
  NullPropagation 空值判断处理
  ColumnPruning 列裁剪
  PushDownPredicates 谓词下推
  RemoveRedundantProjects 删除冗余的判断
  RewritePredicateSubquery 子查询重写为join
  ReorderJoin join条件重新排序
  ResolveTimeZone 时区处理
  TypeCoercion$ImplicitTypeCasts 不匹配类型转换适配
  TypeCoercion$CaseWhenCoercion case＆when语句类型转换
  TypeCoercion$Division 除法类型转换
```
解释其中两条规则
```
   ColumnPruning 列裁剪：简单来说，列裁剪就是将不必要查询的列进行修剪去枝。
   因为大数据的表基本都是列式存储，去掉不必要的列的查询，能规避这些多余列对应文件的读取，进而提高查询性能。
   
   PushDownPredicates 谓词下推：简单来说，就是将判断过滤用的条件尽量往底层更靠近数据读取的环节下推，
   让大量数据的过滤尽早发生，从而避免大量文件的读、中间数据的shuffle以及处理计算，进而提高查询性能。
   
```

题目二：架构设计题
你是某互联网公司的大数据平台架构师，请设计一套基于 Lambda 架构的数据平台架构，要
求尽可能多的把课程中涉及的组件添加到该架构图中。并描述 Lambda 架构的优缺点，要求
不少于 300 字。
```
架构图参见： arch.png。
此图是我公司目前数仓的架构图，时间有限，就没完全按本次作业要求制作。
```
```
Lambda 架构的优点：既支持批量离线数据的处理、又支持实时流式数据的处理.在传统离线处理的基础上，不用对架构进行大的更改，即可支持流式数据处理。
```
```
Lambda 架构的缺点：同1套数据的处理，要编写两套打码。1套离线1套实时，增加了维护成本。
```
题目三：简答题（三选一）   
A：简述 HDFS 的读写流程，要求不少于 300 字  
```
本答案参考《大数据技术原理与应用（第2版）3.6 HDFS的数据读写过程》
读：
1、通过FileSystem.open()方法打开文件，内部通过ClientProtocal.getBlockLocations()远程调用namenode，得到文件开始部分数据块的保存位置，并根据block和客户端的远近排序
2、调用FileSystem.read()读取文件，选择距离最近的block开始读，读取完毕后，关闭和该datanode的连接。
3、调用内部通过ClientProtocal.getBlockLocations()继续获取后续block的位置信息，继续读取
4、全部读取完毕后，掉用FileSystem.close关闭输入流结束读取
5、如果读取异常，则会尝试更换当前block的其他datanode来读取当前block。
   
写：
1、FileSystem.create()方法创建文件，内部通过DistributedFileSystem以rpc的方式调用namenode,新建1个文件。namenode会进行一些存在性、权限等检查，通过后创建1个输出流返回；
2、FileSystem.write()方法开始写文件。文件会被分为1个1个的小包存入1个内部队列。同时输出流会向namenode申请保存文件和副本block的datanode，并将这些datanode形成1个datanode管道；
3、1个个的小包会按顺序写入datanode管道，管道里的各个datanode收到数据包后，先写入自己的磁盘，然后把这个小包发给下游的datanode，下游datanode执行保存后再发给下游；
4、最后1个datanode写入成功后，会向它的前1个datanode反馈1个确认包，各个datanode按反的顺序往前面的datanode持续反馈确认包，直到回到第1个datanode，最后再回到输出流(客户端)；
5、所有文件包都写成功后，调用FileSystem.close()方法关闭输出流，结束文件的写出。
6、一般来说，1个block的多个副本的存放规则是：离客户端最近的节点存1份A，A不同机架的节点存1份B，A同1个机架的其他节点存1份C，如果有更多副本，就随机存储到更多节点上。

```
B：简述 Spark Shuffle 的工作原理，要求不少于 300 字
```
不同于mapreduce，每个环节都进行文件的落盘，导致shuffle的产生，spark创造了有向无环图的思路来优化数据处理流程。通过对处理过程进行分析，确定了宽依赖、窄依赖两种不同的数据转换形态。
窄依赖的两个转换之间，属于同1个jvm，因此其shuffle过程直接是内存处理，不存在文件写和读，因此效率大大的超过mapreduce。
宽依赖由于存在分布式跨节点数据交互，因此必然存在shuffle过程。又细分为shuffle write和shuffle read两个环节，分别进行文件的输出和文件的读取。
shuffle write时，会先将数据存在内存中，达到一定的阈值后再写到hdfs。实际会根据下游stage的情况确定文件数量，并对文件根据hash或排序规则进行分组，写到不同的文件中。
shuffle read时，会根据上游stage的情况，读取当前task需要的文件，一边读一边进行聚合处理。
```
C：简述 Flink SQL 的工作原理，要求不少于 300 字  
```
1、flink sql 同spark sql一样，在sql提交后，都会执行sql分析和优化过程，生成逻辑计划和物理计划，并最终通过codegen生成执行代码；
2、执行代码进一步编译，变成JobGraph提交运行,jobGraph中将窄依赖的task合并为1个stage，将宽依赖的task转换为独立的stage；
3、flink继续优化，根据数据流(文件)的分区（文件数）情况，生成对应的并行度，同1个task的多个并行度对应多个分区概念，不同stage之间的多个分区数据会存在shuffle的过程；
4、如果是批量数据sql处理，则类似于spark执行，最终计算完毕后数据进行输出或者写到输出表(sink)中；
5、如果是流式数据sql处理，则必须有窗口机制，将每个窗口内的数据进行聚合计算，然后输出到表(sink)中；
```
