---
layout: post
title: Java Thread
date: 2015-07-02
categories: java
---


并发在目前的编程概念里可以划分为两种层面，进程与线程层面。最直接的实现方式是在操作系统层面使用进程，彼此之间完全独立，但显然因为资源，上下文切换等问题而受到限制，另一种则是像Java所使用的这种并发系统，共享内存和IO这样的资源，但此时也就不可避免资源竞争的问题。事实上，Thread与Actor都是线程层面，但Actor借鉴了进程的思路，线程的并发任务彼此隔离，通过消息机制取代内存共享避免资源竞争。

Thread与Actor是两种完全不同的多线程设计思路，其中的核心关键在于，是否使用共享内存（Thread角度），或者说，是否使用消息传递数据（Actor角度），实际上前面两句话的意思在这里是一致的，改变线程之间的交互方式，从而引出Thread与Actor两种不同的多线程设计模式。

在平时初学Java多线程或者Windows多线程时，甚至于P-Thread等多线程模型下，思考的方式都是通过共享内存，来打到多线程之间的交互，共享内存包括锁，临界区，互斥量，信号量等形式，也有volatile等标识的可见性内存变量，最后还有Future这一较少涉及到的但十分重要的线程反馈机制。

而Actor的设计思路则是偏向于隔离所有的活动对象，也就是线程，通过消息机制实现线程的协作与功能。程序员不需要再关注资源竞争引发的各种问题，更加侧重于实现活动对象（线程）的功能。

本文内容从Thread开始，穿插介绍Actor对于同样situation的处理方式，但因为两者在设计概念上的不同，所以Actor没有资源竞争的问题，但同样Thread也没有消息队列的问题，但这并不是说两者不能彼此实现，Actor中同样可以使用Thread，Thread也可以实现消息队列传输而不使用共享内存实现交互。


---

## Thread

### Thread Base

实现Thread有两种方式，一种是通过实现Runnable（任务描述）提交给Thread构造器（线程）或者直接在Thread中实现run方法（任务描述），两者本质上是一样的，通过Thread类本身实现新的线程；第二种方法是使用Executor（执行器）管理Thread对象。第一种方法需要程序员自己管理线程的生命周期，而第二种可以异步地管理所有产生的Thread对象。除此之外，Runnable是执行工作的独立任务，不返回任何值，如果你需要在任务完成时返回一个值，则实现Callable接口，并且必须使用ExecutorService.submit()方法调用。在后续的例子中，我们可以看到其实Future使得Thread整个变得更加有趣，因为Future我们可以对Thread运行结果进行检验，选择不同的方式来处理，实际上这也是Thread模拟Actor实现的主要工具。

下例是整合的一些关于Thread类使用的基础样例代码。

```java

    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;

    /**
     *
     * Created by king on 15-7-3.
     */

    class LiftOff implements Runnable {
        protected int countDown = 10;
        private static int taskCount = 0;
        private final int id = taskCount++;
        public LiftOff() {}
        public LiftOff(int countDown) {
            this.countDown = countDown;
        }
        public String status() {
            return "#" + id + "(" + (countDown>0 ? countDown: "LiftOff!") + ").";
        }

        @Override
        public void run() {
            while (countDown-- > 0) {
                System.out.println(status());
                Thread.yield();
            }
        }
    }

    public class MyThreadTest {

        public static void executors() {
            ExecutorService exec = Executors.newCachedThreadPool();
            for (int i = 0; i < 5; i++) {
                exec.execute(new LiftOff());
            }
            exec.shutdown();
        }

        public static void singleThread() {
            Thread t = new Thread(new LiftOff());
            t.start();
            System.out.println("single thread end");
        }

        public static void main(String[] args) {
    //        you can use the runnable class directly, it is invoked by the main thread
    //        LiftOff launch = new LiftOff();
    //        launch.run();
    //        System.out.println("end");
            singleThread();
            executors();
        }

    }

```

除了上面代码所展示的功能，还有经常用的一些多线程机制：
- sleep(),中止一段时间
- setPriority(),设定优先级
- setDaemon()(注意这个方法涉及到finally模块不会执行的一个细节问题)，该线程是否启动为后台线程（守护线程）
- join(),在一个线程内调用另外一个线程的join方法，将会挂起当前线程，等待另一个线程完成时再继续执行
- 以及其他一些内容：ThreadFactory类，继承Thread类或者Runnable接口

---

### Resources Contend in Thread

基本上所有的并发模式在解决线程冲突的问题时，都是采用序列化访问共享资源的方案，即在给定的时刻只有一个任务访问共享资源（通常通过在代码前面加上一条锁语句实现）。

共享资源一般是以对象形式存在的内存片段，也可以是文件，输入输出端口，打印机等等。控制资源的访问，首先将资源包装进一个对象，然后把所有要访问这个对象的方法或者代码片段用synchronized包装起来。synchronized保证了当前只有一个任务在执行被包装的部分代码块，注意synchronized标识的方法，属性域必须设置为private，除非你是故意的。此外，synchronized关键字几乎可以取代显示的Lock机制，除了在一些特殊问题上，例如尝试获取锁一段时间，然后放弃，或者获取锁最终失败这样的情形。

原子性、可见性

只有理解多线程上下文切换概念的基础上，才能理解原子性的概念，原子性指的是：一个具有原子性的操作能够保证整个操作不被线程调度机制中断;一旦操作开始，它一定能在上下文切换之前执行完毕。但遗憾的是在Java语言中，基本上所有的操作都不具有原子性，因为Java语言运行在JVM上，编译成JVM可执行的编码后，往往需要两条及以上的指令来完成一个功能，这样使得很多的操作都必须保证一串指令执行的原子性，才能保证Java语言层面的原子性。换句话说，除非你能够直接编码JVM的指令代码，否则不要指望Java编程语言来实现原子性操作。

可见性是Java多线程内存概念的产物，需要理解主内存与线程内存之间的概念，线程之间往往不共享内部的变量，如果要实现共享变量则需要保证每次加锁主内存中的某个变量然后对其赋值操作，其他线程通过读取主内存的变量来获取其他某个线程现在的变量值，从而实现线程内的变量与其他线程共享，显然这样产生大量的竞争问题。所以，Java采用volatile关键字对一些需要共享的属性域，该属性域的读写操作全部在主内存中完成，保证所有的线程共享同一个变量。但是，这不能作为控制线程的共享变量，最简单的一个实例volatile不能作为线程内计数器。

---

### Thread States

一个线程可以处于以下四种状态之一：

1. 新建（new），线程刚刚被创建时，已经分配系统资源，完成了初始化工作。
2. 就绪（Runnable），已经开始运行，只需要系统分配CPU时间片就可以运行，这不同于阻塞状态。
3. 阻塞（Blocked），被某个条件所阻止运行，调度器不会分配时间片给阻塞的线程，直到线程重新进入就绪状态
4. 死亡（Dead），线程已经结束运行，正常结束或者被中断。

---

### Active Object

加入这个小节的目的是用Thread与Concurrent实现Actor多线程设计模式。之所以称这些对象是活动的，因为每个对象维护自己的工作线程和消息队列，所有的请求都进入消息队列，任何时刻该对象只处理一个消息。这就达到了我们“序列化”的目的。这个设计思路也就是最早的Actor设计方法。具体实现如下：

```
    public class ActiveObjectDemo {
        private ExecutorService ex = Executors.newSingleThreadExecutor();
        private Random rand = new Random(47);

        private void pause(int factor) {
            try {
                TimeUnit.MILLISECONDS.sleep(
                        100 + rand.nextInt(factor)
                );
            } catch (InterruptedException e) {
                System.out.println("sleep interrupted");
            }
        }

        public Future<Integer> calculateInt(final int x, final int y) {
            return ex.submit(new Callable<Integer>() {
                @Override
                public Integer call() throws Exception {
                    System.out.println("Starting " + x + " + " + y);
                    pause(500);
                    return x + y;
                }
            });
        }

        public Future<Float> calculateFloat(final float x, final float y) {
            return ex.submit(new Callable<Float>() {
                @Override
                public Float call() throws Exception {
                    System.out.println("Starting " + x + " + " + y);
                    pause(2000);
                    return x + y;
                }
            });
        }

        public void shutdown() {
            ex.shutdown();
        }

        public static void main(String[] args) {
            ActiveObjectDemo d1 = new ActiveObjectDemo();
            List<Future<?>> results = new CopyOnWriteArrayList<>();

            for (float f = 0.0f; f < 1.0f; f += 0.2f) {
                results.add(d1.calculateFloat(f, f));
            }

            for (int i = 0; i < 5; i++) {
                results.add(d1.calculateInt(i, i));
            }

            System.out.println("-------------asynch calls made----------------");

            while (results.size() > 0) {
                for (Future<?> f: results)
                    if (f.isDone()) {
                        try {
                            System.out.println(f.get());
                        } catch (Exception e) {
                            throw new RuntimeException(e);
                        }
                        results.remove(f);
                    }
            }
            d1.shutdown();

        }

    }
```

上述程序中，由List<Future<?>>来捕获发送给活动对象的calculateFloat和calculateInt方法，并返回的Future对象。


## Actor

Actor设计理念便是为了解决Java多线程并发所产生的各种复杂问题，使得程序员设计多线程程序时能够完全关注线程功能实现而不需要再考虑资源冲突的问题，在这点上其实有很多的"happen before"设计规则在Akka的Actor文档中有详细的描述，等到以后看懂了再添加进来。= =

Actor采用Scala语言描述，官方的akka也有Java实现版本，但是从函数式编程语言更适合实现Actor设计的机制来说，选择Scala编写akka Actor更加简洁高效。

### Actor Base

下面是一个最简单的actor程序，主要是为了展示Actor之间通过消息机制进行交互的方式。

```
    case object PingMessage
    case object PongMessage
    case object StartMessage
    case object StopMessage

    class Ping(pong: ActorRef) extends Actor {
      var count = 0

      def incrementAndPrint {
        count += 1; println("ping")
      }

      def receive = {
        case StartMessage =>
          incrementAndPrint
          pong ! PingMessage
        case PongMessage =>
          incrementAndPrint
          if (count > 99) {
            sender ! StopMessage
            println("ping stopped")
            context.stop(self)
          } else {
            sender ! PingMessage
          }
        case _ => println("Ping got something unexpected.")
      }
    }

    class Pong extends Actor {
      def receive = {
        case PingMessage =>
          println(" pong")
          sender ! PongMessage
        case StopMessage =>
          println("pong stopped")
          context.stop(self)
        case _ => println("Pong got something unexpected.")
      }
    }

    object PingPongTest extends App {
      val system = ActorSystem("PingPongSystem")
      val pong = system.actorOf(Props[Pong], name = "pong")
      val ping = system.actorOf(Props(new Ping(pong)), name = "ping")
      // start the action
      ping ! StartMessage
      // commented-out so you can see all the output
      //system.shutdown
    }

```

首先任何自定义的Actor都必须继承自trait（特质，类似与JavaInterface）Actor。在自定义的Actor中，实现receive方法，采用match-case模式，匹配的内容就是我们定义的消息。

上例中ActorSystem是整个Actor多线程的管理器，通过actorsystem生成新的actor，类似于concurrent库中的Executor，但是actorsystem管理的事情更多，在API文档中定义如下：

    “An actor system is a hierarchical group of actors which share common configuration,
    e.g. dispatchers, deployments, remote capabilities and addresses. It is also the entry point
    for creating or looking up actors.”

actorsystem通过调用方法actorOf生成新的Actor，生成一个新的Actor我们需要知道Actor的内部定义，所有的相关配置项，而这一点通过Props类实现。然后我们就可以看到该Actor已经启动，由actorsystem管理其生命周期。我们通过 ！ 给一个actor发送消息，通知其执行相应的方法。最后需要通过调用shutdown关闭actorsystem。


### Actor States

下图展示了一个Actor的生命周期：

![](/assets/2015-07-02.png)
