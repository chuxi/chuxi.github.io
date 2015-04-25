---
layout: post
title: Spark Source Code Study
date: 2015-04-20
categories: spark
---

之前自己也修改过Spark的源代码，但实际上只是外层的一些根据接口做的小调整而已，所以对Spark的运行机制没有什么深入的理解。而自己对于Java和Scala也是极其粗浅的理解，所以看着Spark的源代码很难理解说，作者为什么要这样写这一段代码。所以在正式开始调试跟踪Spark代码之前，自己拿着《Thinking in Java》和《深入理解Java虚拟机》两本书狠狠地啃了一番，对于对象信息，IO接口，序列化等概念也进一步学习掌握，此外还有concurrent和Actor多线程机制进行了深入的学习，至此，自己一共花费两周左右时间，可以开始看懂Spark源代码了。但是的确磨刀不误砍材工，自己在阅读Spark源代码时也就能够顺利看懂其中很多细节上的设计，也是一边看一边对Spark的那些作者们大吐敬仰之情。


---

### Spark Context
其实这一小节并不是说介绍SparkContext这个类文件，而是先明确一些概念，这些概念都在SparkContext里面初始化或者有着根本的定义，研究Spark内整体的运行机制或者各个模块，都能在Spark Context找到对应的入口点。所以，在正式开始调试之前，有必要拎出SparkContext内对于整体环境的配置项，各个模块的初始化及其之间的关系，理解整个容器底层的运行环境。

此处实在不想对于SparkContext内各个模块一一进行描述，具体的不如去查看源代码内的注释信息，而且主要是各个模块在里面过于复杂，底层的一些系统模块都有各自的作用，但仅限于底层实现，在上层的几个关键架构模块则相对比较容易分析，包括侧边的一些监听器、信息采集等功能，而底层则因为涉及到数据存储文件读写等协调操作，所以更加复杂。

SparkContext内的内容粗浅地划分如下三块：

+ （上层）Job执行操作，包括数据输入接口，Action，任务Job提交，DAGScheduler的Stage划分与提交运行，TaskSchedulerImpl和提供不同集群模式支持的各种SchedulerBackend
+ （底层）SparkEnv，包括集群通信，任务调度，数据传输保存（BlockManager），文件读写等任务模块，主要对接上层Shuffle工作内容，以及提供机器运行的基础环境
+ （侧边）监控以及其他，包括LiveListenerBus（一个ArrayList存储所有Spark状态相关监听器），JobProcessListener（单个Job监听器）以及MetricsSystem（信息采集模块），此外还有心跳机制、ui等模块

如果今后对于某一块内容需要深入了解，则可以从SparkContext入手，开始对于那个模块的跟踪调试，这一个过程主要在于你要知道该模块什么时候被调用了，并且线程及时切换过去观察，总的来说这个调试过程非常有趣，此处可以稍作记录。多线程远程调试，想想不会比这个更复杂了吧，但是在对于集群调试的系统却十分必须。

+ 首先是远程调试的方式：

主要是需要先spark-submit你sbt package后的spark app包到远程服务器上，并且设置该JVM运行为远程Debug模式，等待客户端的连接，这里的话就需要我们在submit的时候配置运行时JVM的参数，关于这点如何配置，我试验之后发现就是在submit中配置   ```--driver-java-options -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8888``` 即可，其他另外两个 ```spark.driver.extraJavaOptions``` 与 ```spark.executor.extraJavaOptions``` 都无法使用。

这个命令最后格式大致如下，其他参数随意调整：

    /usr/local/spark/bin/spark-submit --master spark://master:7077 \
                --class SparkPi \
                 --executor-memory 512m \
                  --total-executor-cores 2 \
                  --driver-java-options -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8888 \
                   /tmp/sparkstudy_2.10-1.0.jar

这样你的远程Spark集群上就有准备调试的程序包了，本地只需要连接该JVM开始调试即可，这一点需要在IDE中设置Remote Debug配合使用，依次选择Edit Configurations->点击左上角的+号->Remote->在弹出的页面里面将Host和Port两个选项设置为你Driver运行所在节点机器的IP和Port，完成后设置好Debug断点，就可以开始运行，此时注意选择Run->Debug...->之前设置的Remote Debug模式名称。

接着是关于多线程调试的技巧，这点上我是在Intellij Idea环境内调试程序，所以窗口做下方有整个进程的Frame信息，信息栏上方可以选择具体Debug跟踪的线程，慢慢跟踪调试下去，注意新启动的线程和线程转换的地方，还有因为Spark底层很多属于事件驱动设计模式，所以在变换跟踪的线程时，注意在不同线程多设置断点。F9往往会使线程跑偏，比较靠谱的方式是你提前知道线程运行位置然后设置断点，在单个线程内的时候尽量选择F8（Step Over）或F7（Step In）。

---

### RDD与Dependency的进一步理解
主要是考虑RDD与Dependency在Job、Stage以及Task中发挥的作用，想到其他的一些信息，所以稍作记录。

之前理解RDD仅仅是认为这个用来作为数据结构表现方式，因为从DataFrame（之前叫做SchemaRDD）中，自己以为RDD存储实际操作的数据，但自己的这个想法只是对了一点点，作为整个计算链开始的RDD中的确存储了原始数据内容，但在后续链上的RDD，都只是存储了数据的计算方式和流向（Dependency），整个RDD从计算逻辑的角度看是从前往后，但在数据结构本身的处理方式上看，是从后往前，就像是链表的header，最后一个RDD才是程序处理的起点。

那么用链表来看RDD和Dependency后，会发现这是个独特的链表，链表有两种节点（数据结构），RDD与Dependency，Dependency代表数据流向（每一个Partition内的每一条Record），RDD内代表数据的具体处理计算方式（compute方法定义）。RDD存储的Dependency引用是数据流入的Dependency，Dependency存储的是parent RDD，所以整条链连接了起来，并且这个链表从尾部向前遍历，我们最后获得的都是最后一个RDD，这一点在Job提交或者Stage划分中都有明显的特征（存储的不是Stage的第一个RDD，而是尾部RDD）。

一开始在App编程中我们写的数据处理Transformation，其实就是用RDD与Dependency串连起来，最后就是形成了一条计算链，这条计算链的header在尾部最后一个RDD，由这个RDD调用Action函数，Action对于Partition内的数据先计算一个结果，然后再将所有Partition的结果汇聚到driver内计算最终结果。那么，我们后续所有的分析都可以根据RDD与Dependency计算链进行，具体的计算分析过程，则涉及到Job提交，Stage划分与运行，Task提交传输等各种问题。

---

### Spark Job Process
对于Spark Job，由一个Action触发并且提交给Spark集群，其实如果只是跟踪这条线并不需要像我之前一样对于SparkContext有那么多的了解，也不需要进入那些初始化的模块看太多东西，就是直接跟踪一个Job Action开始，该RDD的action通过SparkContext的runJob函数解释封装，然后提交给DAGScheduler的runJob，submittedJob封装之后作为一个事件发送给DAGScheduler处理，如果为什么这么别扭，大概就是为了将DAGScheduler以及所有的调度工具都设计成事件驱动类型的模式，而且这样也更加符合自然处理的逻辑（Job流水线）。

具体如下（参考自JerryLead资料）：

+ SparkContext调用 DAGScheduler 的runJob(rdd, cleanedFunc, partitions, allowLocal, resultHandler)来提交 job
+ DAGScheduler 的 runJob 继续调用submitJob(rdd, func, partitions, allowLocal, resultHandler) 来提交 job
+ submitJob() 首先得到一个 jobId，向 DAGSchedulerEventProcessActor 发送 JobSubmitted 信息，该 actor 收到信息后进一步调用dagScheduler.handleJobSubmitted()来处理提交的 job。（事件驱动）
+ handleJobSubmmitted() 首先调用 finalStage = newStage() 来划分 stage，然后submitStage(finalStage)。由于 finalStage 可能有 parent stages，实际先提交 parent stages，等到他们执行完，finalStage 需要再次提交执行。

其实一个Job提交到集群后，很快就按照Stage进行划分，整个过程也非常简单，后续的处理主要是在Stage与Task两层次的概念上。


---

### Spark Stage Cut and Submit
对于Stage划分算法，自己有一个比较模糊的描述，但大体上是跟着代码做解释，所以可以跟着调试的顺序一步步参照我的解释执行下去。

从下面一句代码开始，Stage划分并且返回finalStage（可以设置断点）：

    finalStage = newStage(finalRDD, partitions.size, None, jobId, callSite)

Stage也是一个链表，header在尾部，也就是finalStage就是链表的起点，内部存储parent Stage的引用，向前遍历。为了便于理解，此处提供我自己写的一个简单Spark App，可以跟踪学习Job与Stage整个处理过程：

{% highlight scala %}

    import org.apache.spark.{Partitioner, SparkContext, SparkConf}

    /**
     * Created by king on 15-4-16.
     */
    object CompJob {

      def main (args: Array[String]) {
        val conf = new SparkConf().setAppName("Complex Job").setMaster("local[*]")
        val sc = new SparkContext(conf)

        val data0 = sc.parallelize((0 until 18).map((i: Int) => (i, (i+'a').toChar)), 3)
        val data1 = sc.parallelize((0 until 18).map(i => (i, (i+'A').toChar)))
        val data2 = data0.filter(_._1 % 3 != 0).repartition(2)
        val data3 = data1.repartition(6)
        val data4 = data2.union(data3)
        val data5 = data4.filter(_._1 % 4 != 0)
        val result = data5.groupByKey().filter(_._1 % 5 != 0)

        val r1 = result.count()
        val r2 = result.collect()

        println(r1)
        r2.foreach(println)

        sc.stop()
      }
    }

{% endhighlight %}

上述程序代码的RDD流图和第一个Job划分Stage图如下：

![](/assets/2015-04-20-files/DAG Diagram.bmp)

在Job提交之后，Job首先被划分Stage，然后根据Stage提交给Spark集群执行每一个Stage的内部Task，从前往后执行，至于Stage之间的数据传输流向，就是Shuffle的过程。

    从最后一个RDD开始newStage，每个Stage属于一个或多个Job
    获取parent Stages
    循环向前搜索到第一层ShuffleDependencies，每一个getShuffleMapStage，获取该ShuffleDependency前的Stage DAG Diagram
        判断接下来第一个Stage是否已经在shuffleToMapStage中注册
    	如果有直接返回注册的Stage
    	否则 {
    	    计算该Stage之前的DAG Diagram，registerShuffleDependencies()
    	    首先获取当前Stage前所有的ShuffleDependency，getAncestorShuffleDependencies()
    	    循环处理{
    	        根据每一个获得的ShuffleDependency，newOrUsedStage()，Stage都是会生成的，区别在于是否有已经计算过的数据，newOrUsedStage便是用之前已经计算获得的结果覆盖当前新生成Stage的结果。在shuffleToMapStage注册Stage
    	    }
    	    newOrUsedStage()，在shuffleToMapStage注册Stage，返回结果Stage
    	    }
    返回所有的parent RDDs
    新建最后一个调用Job Action的Stage，在shuffleToMapStage注册Stage，返回finalStage

1. 整个算法里一些奇怪的循环是因为存在UnionRDD的缘故，这是个NarrowDependency，所以一个RDD的计算可能就是依赖于不止一个Stage（ShuffleDependency）。
2. 一个Stage可能在多个Job内使用，为了避免重复计算，使用之前计算的MapOutPut结果覆盖当前Stage。这句话的意思就是，Stage还是正常划分，每个Job都划分Stage，只是这个Stage划分之后判断是否已经有对应的计算结果，若有则将计算结果覆盖上去（覆盖空白Partition结果）。

划分Stage这个过程，需要自己去跟踪调试，代码更多的展现形式是一个递归改成的循环遍历，所以刚开始看的时候会觉得有点怪异，但在适应之后就不会有什么问题，毕竟递归会产生StackOverFlow的问题（据说是MLlib产生过多的StackFrame）

Stage划分之后，就是提交Stage，一个个Stage提交，上一个Stage的结果Shuffle传递给下一个Stage，这个Shuffle的过程十分复杂，建议分开来调试这个过程。

继续向前遍历，找到没有依赖项的Stages，然后对这些Stages内的计算流程打包成TaskSet与TaskSetManager封装，调用TaskSchedulerImpl的submitTasks方法向Task层提交TaskSet，SchedulerBackEnd将会根据具体集群模式进行调度，如果是集群则将TaskSet打包发送给远程机器，再通过Actor（Akka的多线程工具包）通知Executor调用lunchTask()方法处理接受到的Task;如果是本地的LocalBackend，则直接本地启动Executor即可。

TaskSchedulerImpl开始执行TaskSet内的一个Task时，会向DAGScheduler发送一个BeginEvent事件，通知DAGScheduler，或者CompletionEvent，此时需要判断是否已经完成一个TaskSet任务，可以进入下一个Stage的计算，这一点整个可以在EventLoop类内设置适当的断点跟踪到整个事件的提交与变化流程。

整个过程其实涉及到Task层面状态的变化，Stage根据Task的结果是否继续下一步计算，整个代码看着总觉得有些地方冗余，大概需要原boss作者自己亲自动手修改，使得整个代码更加整齐易读。


---

后续也不想对于Task层面做进一步的讲解，因为这个涉及到数据块的内容，Shuffle的过程，数据流的序列化和传输等等一系列过程，整体自己只是有一个大致的框架概念，整个设计十分复杂，但颇有意思的是多线程的事件驱动设计模式，AKKA的Actor运用，这两者的共同作用使得一些设计的地方显得十分简单，大大降低了集群系统的耦合度，使得系统具有极高的扩展性，可以说Actor的多线程是为集群而设计的。后来自己在看书的时候才发现，Thinking in Java的作者Bruce Eckel在多线程的最后一个小结总结时就指出了Actor的多线程设计模式——通过消息机制交互，或许多年后的确就是Akka实现了这种amazing的多线程模式。