---
layout: post
title: Spark JdbcRDD analysis
date: 2014-06-24
categories: spark
---

最近需要做一项关于Spark与传统关系型数据库交互的研究，主要涉及实际应用中遇到现有数据库已经存在并且如何更好的与Spark集群交互的问题，例如MySQL。Spark的Scala与Java编程语言特性让我们可以以普通的数据库查询等方式与数据库进行相关的交互，但这仅限于传统的方式，与Spark集群的特性更是毫无相关，无法充分利用集群的优势对大规模的传统数据库进行有效的操作。所以本文从Scala基础开始，逐步深入到Spark与数据库交互的相关特性研究分析。

### 本文问题：

如何利用Spark对接传统超大规模结构化数据库？（要求充分利用集群特性与分析能力，快速完成大规模数据检索）

---

#### 解决方案一

基于Scala语言编程的前提上，我们利用Slick数据库开发包进行开发，获取数据之后转入Spark schemaRDD。

单节点获取数据，如果要谈RDD这个问题，那就必然涉及到内存大小限制因素，单节点情况下，可以利用硬盘来缓存所有数据然后RDD格式化，再巴拉巴拉一堆或许就OK了。谈不上与Spark对接，只能说最后这样可以转为Spark可用的RDD。

---

#### 解决方案二

将结构化数据库数据导出后，在Hadoop环境基础上，转入HBase等HDFS文件系统的数据库，或者Hive的数据仓库中，完全支撑Spark访问检索需求。

这样完全舍弃原先的结构化数据库，转而进入Hadoop等大规模集群模式下的研究开发，在Yarn的基础上我们可以搭建使用Spark集群，同样，这相当于对原先数据库重新设计，迁移，并且意味着整套系统的重新构建，耗时耗力。实际情况中我们可能更希望将原来的数据无缝对接，保持同样高效逻辑。

---

#### 解决方案三：

JdbcRDD对接结构化数据库，利用Spark SQL做数据进一步分析处理，两者功能叠加之后完整实现结构化数据库与Spark对接，此处利用RDD的Partition各自自动从结构化数据库中select整体数据的一部分，从而充分利用Spark集群优势。

可以说这种方法是前面两种方法的中间方式，显而易见的是避免了前面两种方式的缺陷，总的来说有以下特性：可从一个或多个数据库获取数据；集群式获取结构化数据库数据，内存限制几乎消失；无缝对接Spark SQL特性，完全满足Spark对于数据库数据分析处理。

但这种方式同样也产生许多问题，因为JdbcRDD接口特性的缘故，我们必须大致了解整体数据库表中数据的情况，查询语句需用PreparedStatement来设置一个可为每个partiton确定检索的范围，每个partition都需单独用连接数据库获取数据，个人做过小的测试表明partition个数对于性能存在一定的影响。总之，JdbcRDD估计本身作者也是“随手一写”（人家的随手一写简直就是牛逼极了的存在），并没有特别考虑具体的情况。

---

另外关于Spark SQL扯两句：

近期最新的Spark 1.0.0版本增加了Spark SQL模块（0.9.1里面就已经有了），Spark SQL主要在Spark中实现SQL等语句式的查询方式，针对的数据是一个具体的RDD项，对其中单行的数据项进行检索。而对于传统关系型数据库，Spark SQL缺乏与传统数据库的接口，所以我们无法直接使用Spark SQL来接口结构化数据库。但实际上Spark本身Core中已经配置了对于传统数据库的接口-JdbcRDD，但我也不知道这个类是谁添加的，功能简单有效，设定每一个Partition中的功能为从数据库中自动获取一定范围的数据，也就是只有select功能。而这对于我们来说已经足够了，只要能够获取结构式数据库数据到内存RDD中，我们接下来就能够利用Spark SQL进行一定的筛选操作，从而帮助我们从PT级的结构化数据资料中利用Spark 集群高效快速获取所需信息。


### 代码分析

首先盗用张包峰的博客内容基础上，粗略分析JdbcRDD，然后发表下人生感慨，再然后贴上自己写的样例程序，最后吐槽下那些直接看样例程序的水货，不要学我。

```scala

    class JdbcRDD[T: ClassTag](
        sc: SparkContext,
        getConnection: () => Connection,
        sql: String,
        lowerBound: Long,
        upperBound: Long,
        numPartitions: Int,
        mapRow: (ResultSet) => T = JdbcRDD.resultSetToObjectArray _)
      extends RDD[T](sc, Nil) with Logging {
        ......
    }

```

- getConnection 返回一个已经打开的结构化数据库连接，JdbcRDD会自动维护关闭。
- sql 是查询语句，此查询语句必须包含两处占位符?来作为分割数据库ResulSet的参数，例如："select title, author from books where ? <= id and id <= ?"
- lowerBound, upperBound, numPartitions 分别为第一、第二占位符，partition的个数。例如，给出lowebound 1，upperbound 20， numpartitions 2，则查询分别为(1, 10)与(11, 20)
- mapRow 是转换函数，将返回的ResultSet转成RDD需用的单行数据，此处可以选择Array或其他，也可以是自定义的case class（关于case class在数据库Row中使用方式，参考slick）

JdbcRDD的实现主要继承了RDD，重载了函数getPartitions(), compute(), getNext(), close()四个函数。而JdbcPartition主要增加了lower，upper两个参数，还有给Partition index赋值的工作。仅仅针对JdbcRDD来说，需要理清下面几个方面的问题：

- RDD的划分情况（Partitions列表）
- Partitions内部计算方式
- Partitions的依赖变换
- 其他一些具体情况

JdbcRDD设计方法较为简单，并不考虑Partitions的依赖变换，也不考虑Partition Split计算位置。提到这两点是因为一个RDD需要考虑五点属性：

- A list of partitions
- A function for computing each split
- A list of dependencies on other RDDs
- Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

对应的一个SubRDD一般来说需要实现四个functions与一个变量：

- def compute(split: Partition, context: TaskContext): Iterator[T]
- protected def getPartitions: Array[Partition]
- protected def getDependencies: Seq[Dependency[\_]] = deps
- protected def getPreferredLocations(split: Partition): Seq[String] = Nil
- @transient val partitioner: Option[Partitioner] = None

此处将一个JdbcRDD划分为numPartitions个，其实也就是计算了这个Partition的Index，下界参数，上界参数，最后返回一个JdbcPartition的数组。JdbcPartition则相对简单，继承Partition的基础上除了Partition index参数增加另外两个，主要是为了后续单个JdbcPartition的compute提供参数。

```scala

    override def compute(thePart: Partition, context: TaskContext) = new NextIterator[T] {
        context.addOnCompleteCallback{ () => closeIfNeeded() }
        val part = thePart.asInstanceOf[JdbcPartition]
        val conn = getConnection()
        val stmt = conn.prepareStatement(sql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY)

        // setFetchSize(Integer.MIN_VALUE) is a mysql driver specific way to force streaming results,
        // rather than pulling entire resultset into memory.
        // see http://dev.mysql.com/doc/refman/5.0/en/connector-j-reference-implementation-notes.html
        if (conn.getMetaData.getURL.matches("jdbc:mysql:.*")) {
          stmt.setFetchSize(Integer.MIN_VALUE)
          logInfo("statement fetch size set to: " + stmt.getFetchSize + " to force MySQL streaming ")
        }

        stmt.setLong(1, part.lower)
        stmt.setLong(2, part.upper)
        val rs = stmt.executeQuery()

        override def getNext: T = {
          if (rs.next()) {
            mapRow(rs)
          } else {
            finished = true
            null.asInstanceOf[T]
          }
        }

        override def close() {
          ......
        }
    }

```

compute()函数的输入参数基本上都是一致的，Partition 用 asInstanceOf() 函数转为JdbcPartion。注意compute函数返回值为Iterator[T]类型，此处函数返回一个NextIterator[T]类型，我们查看NextIterator类的源代码，可以根据其代码编写相应的compute()函数。可以发现，在NextInterator类中要求我们重写两个函数：

- protected def getNext(): U
- protected def close()

在此处可以对照下NewHadoopRDD类，里面的compute()函数采用了另外一种Iterator，InterruptibleIterator作为返回类型，参照对比可以进一步了解Spark中的Iterator[T]用法。compute代码本身做过Java数据库编程的可以知道其中内容，整体都没有什么需要特别理解的地方，除了一处mapRow()函数，这个函数是我们传递进来的最后一个参数。

```scala

    def resultSetToObjectArray(rs: ResultSet): Array[Object] = {
        Array.tabulate[Object](rs.getMetaData.getColumnCount)(i => rs.getObject(i + 1))
    }

```

可以看到这个函数的返回值是一个Array[Object]，Array Object中定义了tabulate方法，返回n个function处理后获得的Array，此处n = rs.getMetaData.getColumnCount，function = rs.getObject(i+1)，成功的将ResultSet转换为Array[Object]。

所以通过JdbcRDD源代码分析，我们了解到其与结构化数据库接口的定义方法，同时对于RDD的定义，构建自己的subRDD实现更多功能有了一个初步的了解。在阅读完上面的内容之后其实可以进一步拓展到RDD类本身，还有其一系列派生类，例如CheckpointRDD，NewHadoopRDD类等，同时还有特殊的PairRDDFunctions，DoubleRDDFunctions等，对这些类的分析能够更准确的掌握RDD的操作与编程技巧，也是可以从中找出自己后面编程所需的基本解决方法。

而后我粗略的写了一个sample程序，作为JdbcRDD的运用与Spark SQL结合的展示程序。我从MySQL官网下载了employees数据库作为测试数据（employees表有30万行测试数据）。

```scala

    case class Employee(id: Int, fname: String, lname: String, hdate: Timestamp)

    object JDBCRDD {
      def main(args: Array[String]){
        val conf = new SparkConf().setAppName("JdbcRDD").setMaster("local")
        val sc = new SparkContext(conf)

    //    val time1 = System.currentTimeMillis();
        Class.forName("com.mysql.jdbc.Driver")
        val query = "select * from employees where ? <= emp_no and emp_no <= ?"
    
    //    //connection test
    //    val conn=DriverManager.getConnection("jdbc:mysql://localhost/employees", "root", "vlis@zju")
    //    if (conn!= null)
    //      println("connected")
    //    try {
    //      val select = conn.prepareStatement(query, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY)
    //      select.setFetchSize(Integer.MIN_VALUE)
    //      val s=select.getFetchSize
    //      select.setLong(1, 1)
    //      select.setLong(2, 500000)
    //      val result = select.executeQuery()
    //    }catch {
    //      case e: SQLException if e.getSQLState == "X0Y32" =>
    //      // table exists
    //    } finally {
    //      conn.close()
    //    }
    //    val time2 = System.currentTimeMillis();
    //    println(time2  - time1);

    val Jrdd = new JdbcRDD(
        sc,
          () => {DriverManager.getConnection("jdbc:mysql://localhost/employees", "root", "vlis@zju")},
        query,
        1, 500000, 5,
          (r: ResultSet) => {Employee(r.getInt(1), r.getString(3), r.getString(4), r.getTimestamp(6))}
        ).cache()
        // println(rdd.collect()(2)(2))
        // val time2 = System.currentTimeMillis();
        // println(time2  - time1);
        val sqlContext = new SQLContext(sc)
        import sqlContext._
        val Srdd = Jrdd.filter(_.id < 20000)
        Srdd.registerAsTable("employees")
        sql("SELECT * FROM employees").collect().foreach(println)
        sc.stop();
        //这样做有什么好处？
      }
    }
    
```

其中或许需要注意的点是数据库中的SQL语句里的Date数据类型在schemas中并没有对应的定义，需要用Timestamp替代，具体可以查看 package org.apache.spark.sql.catalyst 中的ScalaReflection类，有对应的说明。此处我将返回的RDD数据类型用Case Class Employee替代了Array[Object]，发现也是可以的（此处乱猜写的，谁知道可以说下）。

如此总结下之前的工作，发现自己也没写什么有效的内容，主要是掌握下JdbcRDD的运用，再延伸到Spark SQL，因为只有进入了Spark SQL才算是进入Spark的计算框架中。自己刚好最近一项工作涉及到Spark SQL这类相关的内容，然后就发现1.0.0版本就增加了这方面的文档与内容（虽然之前就奇怪的搜索出了1.0.0版本文档），也算是运气使然。

除此之外自己也还做了一些关于Hive, Hbase方面的调查，相对来说，出现Spark SQL之后Hive基本已经集成到了Spark框架中，之前存在的一个项目Shark，功能也已经被Spark SQL取代，包括SQL语句分析优化等功能。所以说，1.0.0版本对于整个Spark涉及到的产品做了一次集合，Spark的功能也越来越强大稳定。后续继续学习Spark与Yarn的结合运用，毕竟目标是集群式的Hadoop基础上运行Spark，也计划后面搭建起一个更加稳健的集群系统整合大数据的一些内容。
