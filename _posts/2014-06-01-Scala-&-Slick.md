---
layout:	post
title:	Scala & Slick
date:	2014-05-31 18:32:00
categories:	scala
---

Spark基于Scala语言开发，而Scala语言实际上是Java语言的进一步变化形态，外形上变得更像Python,Ruby,Perl等脚本语言，而实际执行效率却与Java极为相近。所以整体Scala语言开发效率极高，同时也对程序员的能力水平要求相应提高。给我的个人感觉就是，这是一门未来会在编程语言顶端层次的语言，会这门语言的人应对的系统也往往是集群式的开发项目，大概也是“低端”的项目用不上Scala（实际上Scala有许多的特性也使得其十分适合集群项目的开发）。

所以，学习Scala是在深入了解Spark之前的必修课。

本文将从以下几个步骤指导Scala的初步上手：

+ IntelliJ IDEA 新建以sbt开发管理的Scala项目
+ Scala关于RDBMS的操作（Slick Project）

---

### IntelliJ IDEA

1. 按照董西成大叔的博客学习，做完准备工作，同时你可以按照说明完成Spark源码的阅读环境，阅读Spark源码能够极大的提升你Scala的学习水平。此处，我们可以选择在安装Scala插件时安装SBT插件

2. 在IDEA中编写自己的Scala项目程序。New Scala Project（using sbt)--> helloscala

3. 之后我们根据maven的习惯建立文件目录结构，在Scala下略做修改之后，我们可以看到，在src/main/目录下划分有java, scala子目录，一般来说maven仅有java路径。此时建议参考Spark源码目录结构，以Spark为标准学习。除此之外，
![](/assets/2014-05-31-02.png)

4. 接着你可以自己按照scala语法自己写一个Hello Scala的程序了。

### Slick

总有一个经验是这样的，你看完了一门新语言的语法基础，然后屁颠颠的开始写这语言的第一个程序，必然选择Hello XXX。同样，在这之后往往就是写再大一点的，行数再多一点的小程序来证明自己算是初步掌握了Scala的开发，这个小程序，又估计基本上都会选择数据库操作来练手。

Slick是Scala语言开发的集成式数据库连接工具集，原名Scala Language Integrated Connecting Kits，就像Spring中我们使用的数据库连接函数库一样，Slick支持所有数据库的开发，甚至于你自己定义的都可以配置。所以根据maven使用经验，我们可以利用sbt来配置管理slick程序的开发。

在做这些之前，我们需要学习Slick的编程方法，这个在其官网上有详细的说明http://slick.typesafe.com/doc/2.1.0-M2/index.htmlm/） 同时slick也给出了好几个样例程序，在文档里也对一些命令接口进行了说明，但令人尴尬的是这些说明里有一些bug对于我这样的新手来说十分折腾。

在build.sbt文件中，添加以下依赖项：

```scala

    libraryDependencies += "com.typesafe.slick" % "slick_2.10" % "2.1.0-M1"
    libraryDependencies += "mysql" % "mysql-connector-java" % "5.1.30"
    
```


其中，slick基于scala2.11的新版本还在试验中，我们采用slick2.10版本，另外，scala版本也最好选择2.10.4，至少在2.11版本的scala基础上程序包中某个文件有错误而无法编译。mysql-connector的依赖包也必须包含，如果要使用其他数据库，则可以在maven库中寻找。其中oracle，Microsoft SQL Server等则在slick的官方库中，所以需要同时添加源。

```scala

    import scala.slick.jdbc.JdbcBackend.Database
    import scala.slick.driver.MySQLDriver.simple._
    import scala.slick.jdbc.{GetResult, StaticQuery => Q}

```

可以在github上查看到slick项目的源代码，此处我们的基本操作需要包含以上几个类包，一般来说，六个及六个以上类的import则用_代替，Session类包含在simple中。

```scala

    object MySQLHelper extends App{
    
        // Case classes for our data
        case class Person(name:String, age:Int)
    
        implicit val getPersonResult = GetResult(r => Person(r.nextString(), r.nextInt()))
    
        val db=Database.forURL("jdbc:mysql://localhost/spark", "root", """vlis@zju""", null, """com.mysql.jdbc.Driver""")
    
        db.withSession { implicit session :Session =>
            Q.queryNA[Person]("select * from person") foreach { c=> println(" "+ c.name + "\t" + c.age)}
        }
    
    }

```

在slick文档中有关于数据库连接与Session的说明，数据库连接格式如上，也真是不断瞎蒙下弄出来的，不过同样可以适用与其他数据库连接。注意必须实现GetResult方法，这样才能在println中使用c.name。

以上是SQL语句的实现方式，而实际上Slick中采用Scala来替代SQL实现，通过一系列的map filter等等。

```scala

    class MyDBHelper{
      class Student(tag: Tag) extends Table[(Int, String, Int)](tag, "PERSON"){
        def id = column[Int]("ID", O.PrimaryKey)
        def name = column[String]("NAME", O.NotNull)
        def age = column[Int]("AGE")
        def * = (id, name, age)
      }
    
      val stu=TableQuery[Student]
    
      def create(implicit session: Session) = stu.ddl.create;
    
      def insert(d: Int, n:String, a:Int)(implicit session: Session) = stu.insert(d, n, a)
    
    //  def sqlexc(ss: String)(implicit  session: Session) : Query[Student,(Int, String, Int)] = Q.query[Student, (Int, String, Int)](ss)
    
    }
    
    object MySQLHelper extends App{
    
      val s=new MyDBHelper();
      val db=Database.forURL("jdbc:mysql://localhost/spark", "root", """vlis@zju""", null, """com.mysql.jdbc.Driver""")
      db.withSession{implicit session :Session =>
        s.create
        s.insert(1, "hanwei", 16)
        s.insert(2, "hanwei1", 21)
        s.insert(3, "hanwei2", 31)
        s.stu.filter(_.age>20).foreach{case (id, name, age) => println(id + " " + name + " " + age)}
      }
    }

```

更详细的对比在文档 http://slick.typesafe.com/doc/2.1.0-M2/from-sql-to-slick.html 中。实际上主要在于建立基础的TableQuery[]，其所有sql的操作都是建立在我上方 `val stu=TableQuery[Student]` 上。设计方式跟Python Django的数据库schema建立十分相似（说这句话真是为了莫名其妙的装逼下自己知道Django这回事的事实）。

对于Slick，官方的文档说明相当详细了，更多的操作方法都在文档里有相应的指导。我自己也没有具体用过更加复杂的SQL，用Scala来替换的经验，但免去了SQL语句转换直接用Scala连接操作数据库的好处还是显而易见的，更快更安全。

---

回过头一看，觉得当时自己写的还是好幼稚，写的内容只是做个笔记方便今后使用回顾，而记录API则意义不大。

