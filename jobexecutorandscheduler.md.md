# Job executor and task scheduler
这部分主要来研究一下Spark如何来执行一个Job的，也就是说，我们写了一个Spark程序(可能有多个Job)，然后提交给配置好的Spark集群，Job是如何执行的，经历了那些模块最终以什么形式进行执行（最终肯定是以线程的形式执行任务的）,分析执行的过程之前我先介绍Spark和调度有关的信息，然后主要详细介绍任务调度和作业调度。

### Spark调度
从上往下大致可以把Spark的调度分为四大块：Spark应用程序之间的调度、Spark应用程序内Job的调度、Stage的调度、Task的调度。在详细介绍这几种调度方法之前，先来介绍一下几个概念：
 - 1.Spark程序：我们写的基于Spark提供的API实现的一个具有各种逻辑操作的一个完整的Application。
 - 2.Job：这里提到的Job是指在一个Spark程序里面，从Transformation操作到Action操作的一个完整的过程也称为DAG图。一个Spark程序可以有多个Action操作，之后会写一个简单的程序展示一下，也就是说具有多个Job。
 - 3.Stage是Spark中很重要的一个概念，一个Job可以根据算子中是否具有Shuffle过程，将一个完整的DAG图划分为多个Stage。
 - 4.Task：是Spark中进行具体的执行，是根据Stage来相应的产生相应的Task。

不理解的同学可以看一下[大神的文章](https://github.com/JerryLead/SparkInternals/blob/master/markdown/3-JobPhysicalPlan.md),写的超级好，那作为菜鸟的我在这里只是向以自己的思路来总结一下对Spakr的理解。

**1.Spark应用程序之间的调度** ： 一个Spark集群可以同时提交多个应用程序，那Spark是如何对这些应用程序进行调度和管理的呢？我们知道Spark可以有多种提交程序的方法：可以使用Spark自带的Standalone模式、Mesos、YARN三种方法来提交作业，这是最高层次的调度，可以直接控制程序执行所占用的物理资源：cpu和内存等。
 - Standalone: 默认情况下，用户使用Standalone模式向运行的Spark集群提交应用是按FIFO的形式执行应用程序，也就是说当前执行的Spark程序独占所有的集群资源，哪怕cpu和内存利用率不高，只有等当前程序执行完成，队列中的程序才有机会获得执行。那如果想同时执行多个Spark应用程序怎么办？需要使用配置参数，可以在应用程序中使用SparkConf对象的set方法（set("spark.cores.max","40")）来设置当前程序占用的资源数量。
 - Mesos: Mesos可以支持粗粒度和细粒度两种调度模式，粗粒度模式和Standalone的设置max cores类似，程序固定占用物理资源，通过设置spark.mesos.max 为true（和上面一样的设置方法）同时设置spark.cores.max 和spark.executor.memory来限制Spark程序占用的资源，这里说明一点 **Spark中启动的工作守护进程是Worker，Worker是按你的配置文件进行启动，具体分配多少内存、多少cpu以及一台物理机器启动多少个Worker实例，但是一个Spark应用程序只能在每个Worker中具有一个Executor**，这里粗粒度的资源调度相当与给每个应用程序定死了计算资源的数量。而细粒度的调度模式有所不同，Spark应用程序依然会分配固定的cpu和内存，但是当应用程序占用的资源比分配的要少的时候，其他应用程序可以使用这部分空闲资源，用户只需要使用 conf.setMaster("mesos://HOST:5050"),同样设置内存和cpu，但是spark.mesos.max 参数不需要设置为true。
 - YARN： YARN来提交应用程序的时候可以在提交任务的时候加上一些配置信息，比如：--num-executors选项来控制这个应用程序分配多少个Executor，通过--executor-memory和--executo-cores来控制应用被分配到的每个Executor的内存大小和cpu核数即线程数。这里需要注意，参数的设置需要合理，如果Worker的配置资源数无法满足的话程序是提交不上集群的。
 
**2.Job的调度**
这里的Job再次说明一下是应用程序中action操作触发的RDD DAG的Job，可以这么理解，看程序中的action操作都会调用sc.runJob()方法，其中有多少个这样被调用的方法就有多少Job。
 - FIFO模式：默认情况下，Spark的调度器是以FIFO方式调度Job的执行，
 
 - FAIR模式
