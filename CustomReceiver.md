# Custom Receiver
Spark Streaming已经提供了很多的InputDstream，除了与Socket相关的离散流（SocketInputDstream，RawInputDstream等）以外，还有其他整合的工具，比如基于flume、kafka、mqtt、twitter、zeromq等工具都有实现相关的输入流，这其中的重点是基于[Kafka](http://spark.apache.org/docs/latest/streaming-kafka-integration.html)和[flume](http://spark.apache.org/docs/latest/streaming-flume-integration.html)的输入流，现在很多大公司都是基于kafka日志收集系统（当然不仅仅局限于日志的分析，其实质是一个分布式的文件系统，而Spark Streaming在基于kafka的支持，提供了两种方式，一种是接收器的方式，另外一种是基于文件的形式，第二种效率要高很多，在空闲时间的中我会另起博文进行详细阐述）利用storm或则Spark Streaming进行日志实时分析，好了不扯淡了。

### 简单的InputStream和Receiver
首先我们来看看Spark源码SocketInputDStream、ReceiverInputDStream、InputDStream，这三个对象从前往后继承，然后再看看抽象类Receiver。这里说明一点：在写自己的Receiver的时候，可以实现自己的InputDstream同时实现其getReceiver()方法；也可以只是实现了一个Receiver，然后使用StreamingContext的方法receiverStream将Receiver传进去。下面我都会介绍。
**SocketInputDStream**
```
private[streaming]  //这里是限制类的访问权限，只能在streaming包下可以访问。
class SocketInputDStream[T: ClassTag](   //而这些参数再实现自己的DStream的时候可以直接复制过去
    @transient ssc_ : StreamingContext,
    host: String,
    port: Int,
    bytesToObjects: InputStream => Iterator[T],
    storageLevel: StorageLevel   
  ) extends ReceiverInputDStream[T](ssc_) {   //继承ReceiverInputDStream类

  def getReceiver(): Receiver[T] = {   //很简单，只需要实现 一个方法：getReceiver(),该方法需要返回Receiver类型的对象
    new SocketReceiver(host, port, bytesToObjects, storageLevel)  //返回SocketReceiver对象，该对象继承抽象类Receiver
  }
}

private[streaming]
class SocketReceiver[T: ClassTag](
    host: String,
    port: Int,
    bytesToObjects: InputStream => Iterator[T],
    storageLevel: StorageLevel
  ) extends Receiver[T](storageLevel) with Logging {  //继承Receiver

  def onStart() {          //实现onStart()方法，很简单在里面需要实现如何把自己需要接收的数据接收进来
    // Start the thread that receives data over a connection
    new Thread("Socket Receiver") {
      setDaemon(true)
      override def run() { receive() }   //将负责接收数据的receive的方法使用线程启动
    }.start()
  }

  def onStop() {      //这个方法没用，在需要停止线程的执行的时候，将receiverState 设置为 Stopped，isStopped()返回true
    // There is nothing much to do as the thread calling receive()  
    // is designed to stop by itself isStopped() returns false
  }

  /** Create a socket connection and receive data until receiver is stopped */
  def receive() {
    var socket: Socket = null
    try {
      logInfo("Connecting to " + host + ":" + port)
      socket = new Socket(host, port)
      logInfo("Connected to " + host + ":" + port)
      val iterator = bytesToObjects(socket.getInputStream())  //bytesToObjects方法负责从socket拿数据并装换成object
      while(!isStopped && iterator.hasNext) {   //这里isStopped是Receiver的一个方法，并不是一个变量，用它来控制线程的停止
        store(iterator.next)                  //将数据存到Spark平台，存过去的数据放到ArrayBuffer数据结构中，循环的获取数据
      }
      logInfo("Stopped receiving")
      restart("Retrying connecting to " + host + ":" + port)   //断线重连，
    } catch {
      case e: java.net.ConnectException =>
        restart("Error connecting to " + host + ":" + port, e)  //断线重连，
      case t: Throwable =>
        restart("Error receiving data", t)            //断线重连，
    } finally {
      if (socket != null) {
        socket.close()                    //关闭Socket
        logInfo("Closed socket to " + host + ":" + port)
      }
    }
  }
}

private[streaming]
object SocketReceiver  {

  /**
   * This methods translates the data from an inputstream (say, from a socket)
   * to '\n' delimited strings and returns an iterator to access the strings.
   */
  def bytesToLines(inputStream: InputStream): Iterator[String] = {    //Socket中的输入流装换为String并返回为迭代器
    val dataInputStream = new BufferedReader(new InputStreamReader(inputStream, "UTF-8"))
    new NextIterator[String] {
      protected override def getNext() = {
        val nextValue = dataInputStream.readLine()
        if (nextValue == null) {
          finished = true
        }
        nextValue
      }
      protected override def close() {
        dataInputStream.close()
      }
    }
  }
}
```
总结下来就是，要实现一个自己的输入流分一下三部曲：
 - 第一**继承ReceiverInputDStream实现getReceiver方法**
 - 第二**继承Receiver并以对象的形式在getReceiver中返回**
 - 第三**实现Receiver的onStart()、onStop()三个方法**
 
有兴趣的可以去看看ReceiverInputDStream和InputDStream类，这两个类中最重要的就是ReceiverInputDStream的compute方法，用来构造RDD的
原理很简单：根据用户传入的时间间隔参数，然后获取到这个时间间隔里的块信息，而块是没200ms就会生成，根据所有的块信息就可一构建RDD了。
```
override def compute(validTime: Time): Option[RDD[T]] = {
    val blockRDD = {

      if (validTime < graph.startTime) {
        // If this is called for any time before the start time of the context,
        // then this returns an empty RDD. This may happen when recovering from a
        // driver failure without any write ahead log to recover pre-failure data.
        new BlockRDD[T](ssc.sc, Array.empty)
      } else {
        // Otherwise, ask the tracker for all the blocks that have been allocated to this stream
        // for this batch
        val blockInfos =
          ssc.scheduler.receiverTracker.getBlocksOfBatch(validTime).get(id).getOrElse(Seq.empty)
        val blockStoreResults = blockInfos.map { _.blockStoreResult }
        val blockIds = blockStoreResults.map { _.blockId.asInstanceOf[BlockId] }.toArray

        // Check whether all the results are of the same type
        val resultTypes = blockStoreResults.map { _.getClass }.distinct
        if (resultTypes.size > 1) {
          logWarning("Multiple result types in block information, WAL information will be ignored.")
        }

        // If all the results are of type WriteAheadLogBasedStoreResult, then create
        // WriteAheadLogBackedBlockRDD else create simple BlockRDD.
        if (resultTypes.size == 1 && resultTypes.head == classOf[WriteAheadLogBasedStoreResult]) {
          val logSegments = blockStoreResults.map {
            _.asInstanceOf[WriteAheadLogBasedStoreResult].segment
          }.toArray
          // Since storeInBlockManager = false, the storage level does not matter.
          new WriteAheadLogBackedBlockRDD[T](ssc.sparkContext,
            blockIds, logSegments, storeInBlockManager = true, StorageLevel.MEMORY_ONLY_SER)
        } else {
          new BlockRDD[T](ssc.sc, blockIds)
        }
      }
    }
    Some(blockRDD)
  }
```
**Receiver**
```
abstract class Receiver[T](val storageLevel: StorageLevel) extends Serializable {

  def onStart()             //启动接收流

  /**
   * This method is called by the system when the receiver is stopped. All resources
   * (threads, buffers, etc.) setup in `onStart()` must be cleaned up in this method.
   */
  def onStop()             //停止接收流，一般不做处理

  /** Override this to specify a preferred location (hostname). */
  def preferredLocation : Option[String] = None       //如果需要指定接收器的位置，可以通过这个方法设置

  
  /** Report exceptions in receiving data. */
  def reportError(message: String, throwable: Throwable) {
    executor.reportError(message, throwable)
  }

  /**
   * Restart the receiver. This method schedules the restart and returns
   * immediately. The stopping and subsequent starting of the receiver
   * (by calling `onStop()` and `onStart()`) is performed asynchronously
   * in a background thread. The delay between the stopping and the starting
   * is defined by the Spark configuration `spark.streaming.receiverRestartDelay`.
   * The `message` will be reported to the driver.
   */
  def restart(message: String) {                                  //重启接收器，当接收器出现故障的时候
    executor.restartReceiver(message)
  }

  /**
   * Restart the receiver. This method schedules the restart and returns
   * immediately. The stopping and subsequent starting of the receiver
   * (by calling `onStop()` and `onStart()`) is performed asynchronously
   * in a background thread. The delay between the stopping and the starting
   * is defined by the Spark configuration `spark.streaming.receiverRestartDelay`.
   * The `message` and `exception` will be reported to the driver.
   */
  def restart(message: String, error: Throwable) {
    executor.restartReceiver(message, Some(error))
  }

  /**
   * Restart the receiver. This method schedules the restart and returns
   * immediately. The stopping and subsequent starting of the receiver
   * (by calling `onStop()` and `onStart()`) is performed asynchronously
   * in a background thread.
   */
  def restart(message: String, error: Throwable, millisecond: Int) {
    executor.restartReceiver(message, Some(error), millisecond)
  }

  /** Stop the receiver completely. */
  def stop(message: String) {
    executor.stop(message, None)
  }

  /** Stop the receiver completely due to an exception */
  def stop(message: String, error: Throwable) {
    executor.stop(message, Some(error))
  }

  /** Check if the receiver has started or not. */
  def isStarted(): Boolean = {
    executor.isReceiverStarted()
  }

  /**
   * Check if receiver has been marked for stopping. Use this to identify when
   * the receiving of data should be stopped.
   */
  def isStopped(): Boolean = {
    executor.isReceiverStopped()
  }
}
```
把一些和存储有关的代码删除之后，上面类简洁了很多，需要实现的就两个方法。
说了这么多，接下来带大家写一个简单的自己实现的custom Receiver，之后再来个复杂的。
### 自己实现一个Custom Receiver
```
class SocketReceiver(host: String, port: Int)
  extends Receiver[String](StorageLevel.MEMORY_AND_DISK_2) with Logging {
  def onStart() {
    new Thread("Socket Receiver") {
      override def run() { receive() }
    }.start()
  }
  def onStop() {
  }
  private def receive() {
    var socket: Socket = null
    var userInput: String = null
    try {
     socket = new Socket(host, port)
     val reader = new BufferedReader(new InputStreamReader(socket.getInputStream(), "UTF-8"))
     userInput = reader.readLine()
     while(!isStopped && userInput != null) {
       store(userInput)
       userInput = reader.readLine()
     }
     reader.close()
     socket.close()
     restart("Trying to connect again")
    } catch {
     case e: java.net.ConnectException =>
       restart("Error connecting to " + host + ":" + port, e)
     case t: Throwable =>
       restart("Error receiving data", t)
    }
  }
}
```
和上面SocketInputStreaming实现一个功能，从socket中接收数据，很简单吧，继承Receiver，实现两个方法，搞定。
下面看看怎么使用这个接收器：
```
object CustomWordcount {
  def main(args: Array[String]) {
    if (args.length < 1) {
      System.err.println("Usage: NetworkWordCount <master> <hostname> <port>\n" +
        "In local mode, <master> should be 'local[n]' with n > 1")
      System.exit(1)
    }
   // StreamingExamples.setStreamingLogLevels()
   val sparkConf = new SparkConf().setAppName("helloworld").setMaster(str)
   var ssc = new StreamingContext(sparkConf, Seconds(1))
   val loadnode = scala.xml.XML.loadFile("/home/havstack/eclipse/workspace/wordcount/hostAndIp.xml")  //这里从配置文件中读取多路数据进行处理
   var HostName: Seq[String] = null
   var Port: Seq[String] = null
   loadnode match {
    case <catalog>{ therms @ _* }</catalog> =>
      HostName = for (therm @ <HostPort>{ _* }</HostPort> <- therms) yield (therm \ "HostName").text
      Port = for (therm @ <HostPort>{ _* }</HostPort> <- therms) yield (therm \ "Port").text
   }
   HostName.foreach(println)
   Port.foreach(println)
  /*(0 until Port.length).map{
    i=>ssc.receiverStream(new CustomReceiver(HostName(i), Port(i).toInt)).flatMap(_.split(" ")).
    map(x => (x, 1)).reduceByKey(_ + _).foreachRDD(rdd => rdd.foreach(record => println(record._1 + " " + record._2)))
  }*/
    val fis = (0 until Port.length).map { i => ssc.receiverStream(new CustomReceiver(HostName(i), Port(i).toInt)) }
    val unionStreams = ssc.union(fis)
    //val customReceiverStream = ssc.receiverStream(new CustomReceiver(args(1), args(2).toInt))
    val words = unionStreams.flatMap(_.split(" "))
    val wordCounts = words.map(x => (x, 1)).reduceByKey(_ + _)
    wordCounts.foreachRDD(rdd => rdd.foreach(record => println(record._1 + " " + record._2)))
}

```



