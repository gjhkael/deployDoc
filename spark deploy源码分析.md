#Spark Deploy源码分析
spark在资源管理和调度上提供了多种方式（Mesos、YARN、Standalone）其中Standalone是spark自带的资源管理和调度器。deploy模块就是Standalone。下面对Deploy模块进行分析。参考文献：[this way]

##Deploy模块整体架构
**deploy**模块主要包含3个子模块：master, worker,client。他们继承于**Actor**，通过actor实现互相之间的通信。

- **Master**：master的主要功能是接收worker的注册并管理所有的worker，接收client提交的application，(FIFO)调度等待的application并向worker提交。默认的应用程序调度是FIFO，先提交的app先执行并且占用整个集群资源，通过设置spark.cores.max来限制一个application的最大核数，让多个程序可以同时共享资源。
- **Worker**：worker的主要功能是向master注册自己，根据master发送的application配置进程环境。
- **Client**：client的主要功能是向master注册并监控application。当用户创建SparkContext时会实例化SparkDeploySchedulerBackend，而实例化SparkDeploySchedulerBackend的同时就会启动client，通过向client传递启动参数和application有关信息，client向master发送请求注册application。

下面是deploy模块的类图：
![需要重画](http://jerryshao.me/img/2013-04-30-deploy/deploy_uml.png)
##Deploy模块通信消息
Deploy模块并不复杂，代码也不多，主要集中在各个子模块之间的消息传递和处理上，因此在这里列出了各个模块之间传递的主要消息：
- **client** to **master**
      RegisterApplication (向master注册application)
- **master** to **client**
      RegisteredApplication (作为注册application的reply，回复给client)
      ExecutorAdded (通知clientworker已经启动了Executor环境，当向worker发送LaunchExecutor后通知client)
      ExecutorUpdated (通知clientExecutor状态已经发生变化了，包括结束、异常退出等，当worker向master发送ExecutorStateChanged后通知client)
- **master** to **worker**
      LaunchExecutor (发送消息启动Executor环境)
      RegisteredWorker (作为worker向master注册的reply)
      RegisterWorkerFailed (作为worker向master注册失败的reply)
      KillExecutor (发送给worker请求停止executor环境)
- **worker** to **master**
      RegisterWorker (向master注册自己)
      Heartbeat (定期向master发送心跳信息)
      ExecutorStateChanged (向master发送Executor状态改变信息)

##Deploy 模块代码详解
###worker
打开SPARK_HOME/sbin目录下的start-slave.sh,其中调用脚本启动Worker
```sh
"$sbin"/spark-daemon.sh start org.apache.spark.deploy.worker.Worker "$@"
```
接着看spark源码core/deploy/worker模块中的Worker.scala,拖到最后找到Worker伴生对象Worker的主函数
```scala
 def main(argStrings: Array[String]) {
    SignalLogger.register(log)
    val conf = new SparkConf
    val args = new WorkerArguments(argStrings, conf)
  //后面这4个参数应该是从脚本bin/load-spark-env.sh中获得
    val (actorSystem, _) = startSystemAndActor(args.host, args.port, args.webUiPort, args.cores,
      args.memory, args.masters, args.workDir)
    actorSystem.awaitTermination()  //等待调用脚本stop-slaves.sh停止
  }
```
```scala
val (actorSystem, boundPort) = AkkaUtils.createActorSystem(systemName, host, port,
      conf = conf, securityManager = securityMgr)
    actorSystem.actorOf(Props(classOf[Worker], host, boundPort, webUiPort, cores, memory,
      masterUrls, systemName, actorName,  workDir, conf, securityMgr), name = actorName)
    (actorSystem, boundPort)   //返回actorSystem和一个数字（1，2，3。。。）因为start-slaves.sh是循环启动worker
```
利用startSystemAndActor函数创建一个SystemActor，因为创建一个SystemActor之后，Worker类中的prestart（）会先执行。
```
 override def preStart() {            //在Worker启动之前，会先执行这里
    assert(!registered)
    logInfo("Starting Spark worker %s:%d with %d cores, %s RAM".format(
      host, port, cores, Utils.megabytesToString(memory)))
    logInfo("Spark home: " + sparkHome)
    createWorkDir()
    context.system.eventStream.subscribe(self, classOf[RemotingLifecycleEvent])
    shuffleService.startIfEnabled()
    webUi = new WorkerWebUI(this, workDir, webUiPort)  //webUI相关信息
    webUi.bind()
    registerWithMaster()                 //向master注册worker

    metricsSystem.registerSource(workerSource)  //监控信息开始监控
    metricsSystem.start()
  }
```
代码很清晰，主要是向master注册自己，调用registerWithMaster()方法注册
```
if (master != null) {
          master ! RegisterWorker(//如果master已经启动了向master注册自己
            workerId, host, port, cores, memory, webUi.boundPort, publicAddress)
        } else {
          // We are retrying the initial registration
          tryRegisterAllMasters()
        }
```
worker除了向master主动注册自己以外不会在主动和master通信，除了master告诉它注册成功之后，worker才会周期性的向master发送心跳信息。

**这里需要特别说一下，worker向master发送消息之后，master并不需要重新获得worker的引用，因为worker向master通信的时候把自身的引用发送过去，master只要简单的利用actor提供的方法sender！就可以往回发送消息**
下面看看worker的receiveWithLogging里面处理那些master发送给它的事件
```
override def receiveWithLogging = {  //接收消息并做相应的处理
    case RegisteredWorker(masterUrl, masterWebUiUrl) =>    //这里是从master中获取的信息，
//      logInfo("Successfully registered with master " + masterUrl)  //打印成功注册信息
      registered = true                                             //将registered设置为true
      changeMaster(masterUrl, masterWebUiUrl)
      context.system.scheduler.schedule(0 millis, HEARTBEAT_MILLIS millis, self, SendHeartbeat)  //利用akka Scheduler周期性的发送心跳
      if (CLEANUP_ENABLED) {
        logInfo(s"Worker cleanup enabled; old application directories will be deleted in: $workDir")
        context.system.scheduler.schedule(CLEANUP_INTERVAL_MILLIS millis,
          CLEANUP_INTERVAL_MILLIS millis, self, WorkDirCleanup)     //周期性的调用WorkDirCleanup，对work中的内容进行清理
      }

    case SendHeartbeat =>
      if (connected) { master ! Heartbeat(workerId) } //发送心跳信息

    case WorkDirCleanup =>               
      // Spin up a separate thread (in a future) to do the dir cleanup; don't tie up worker actor
      val cleanupFuture = concurrent.future {
        val appDirs = workDir.listFiles()
        if (appDirs == null) {
          throw new IOException("ERROR: Failed to list files in " + appDirs)
        }
        appDirs.filter { dir =>
          // the directory is used by an application - check that the application is not running
          // when cleaning up
          val appIdFromDir = dir.getName
          val isAppStillRunning = executors.values.map(_.appId).contains(appIdFromDir)
          dir.isDirectory && !isAppStillRunning &&
          !Utils.doesDirectoryContainAnyNewFiles(dir, APP_DATA_RETENTION_SECS)
        }.foreach { dir => 
          logInfo(s"Removing directory: ${dir.getPath}")
          Utils.deleteRecursively(dir)
        }
      }

      cleanupFuture onFailure {
        case e: Throwable =>
          logError("App dir cleanup failed: " + e.getMessage, e)
      }

    case MasterChanged(masterUrl, masterWebUiUrl) =>   //如果是恢复模式的话，可能会用到这里，无视
      logInfo("Master has changed, new master is at " + masterUrl)
      changeMaster(masterUrl, masterWebUiUrl)

      val execs = executors.values.
        map(e => new ExecutorDescription(e.appId, e.execId, e.cores, e.state))
      sender ! WorkerSchedulerStateResponse(workerId, execs.toList, drivers.keys.toSeq)

    case Heartbeat =>                                //从driver中接收心跳检测
      logInfo(s"Received heartbeat from driver ${sender.path}")

    case RegisterWorkerFailed(message) =>   //註冊失敗信息
      if (!registered) {
        logError("Worker registration failed: " + message)
        System.exit(1)
      }

    case ReconnectWorker(masterUrl) =>   //重連master
      logInfo(s"Master with url $masterUrl requested this worker to reconnect.")
      registerWithMaster()

    case LaunchExecutor(masterUrl, appId, execId, appDesc, cores_, memory_) =>  //启动Executor,接收从master中来的信息
      if (masterUrl != activeMasterUrl) {
        logWarning("Invalid Master (" + masterUrl + ") attempted to launch executor.")
      } else {
        try {
          logInfo("Asked to launch executor %s/%d for %s".format(appId, execId, appDesc.name))

          // Create the executor's working directory  //新建Id
          val executorDir = new File(workDir, appId + "/" + execId)
          if (!executorDir.mkdirs()) {
            throw new IOException("Failed to create directory " + executorDir)
          }

          val manager = new ExecutorRunner(appId, execId, appDesc, cores_, memory_,
            self, workerId, host, sparkHome, executorDir, akkaUrl, conf, ExecutorState.LOADING)  //启动Executor
          executors(appId + "/" + execId) = manager
          manager.start()
          coresUsed += cores_                //将核数相减
          memoryUsed += memory_
          master ! ExecutorStateChanged(appId, execId, manager.state, None, None)
        } catch {
          case e: Exception => {
            logError(s"Failed to launch executor $appId/$execId for ${appDesc.name}.", e)
            if (executors.contains(appId + "/" + execId)) {
              executors(appId + "/" + execId).kill()
              executors -= appId + "/" + execId
            }
            master ! ExecutorStateChanged(appId, execId, ExecutorState.FAILED,
              Some(e.toString), None)
          }
        }
      }

    case ExecutorStateChanged(appId, execId, state, message, exitStatus) =>
      master ! ExecutorStateChanged(appId, execId, state, message, exitStatus)
      val fullId = appId + "/" + execId
      if (ExecutorState.isFinished(state)) {
        executors.get(fullId) match {
          case Some(executor) =>
            logInfo("Executor " + fullId + " finished with state " + state +
              message.map(" message " + _).getOrElse("") +
              exitStatus.map(" exitStatus " + _).getOrElse(""))
            executors -= fullId
            finishedExecutors(fullId) = executor
            coresUsed -= executor.cores
            memoryUsed -= executor.memory
          case None =>
            logInfo("Unknown Executor " + fullId + " finished with state " + state +
              message.map(" message " + _).getOrElse("") +
              exitStatus.map(" exitStatus " + _).getOrElse(""))
        }
      }

    case KillExecutor(masterUrl, appId, execId) =>   //杀掉executor
      if (masterUrl != activeMasterUrl) {
        logWarning("Invalid Master (" + masterUrl + ") attempted to launch executor " + execId)
      } else {
        val fullId = appId + "/" + execId
        executors.get(fullId) match {
          case Some(executor) =>
            logInfo("Asked to kill executor " + fullId)
            executor.kill()
          case None =>
            logInfo("Asked to kill unknown executor " + fullId)
        }
      }

    case LaunchDriver(driverId, driverDesc) => {  //LaunchDriver
      logInfo(s"Asked to launch driver $driverId")
      val driver = new DriverRunner(conf, driverId, workDir, sparkHome, driverDesc, self, akkaUrl)
      drivers(driverId) = driver   //DriverRunner开始
      driver.start()

      coresUsed += driverDesc.cores
      memoryUsed += driverDesc.mem
    }

    case KillDriver(driverId) => {
      logInfo(s"Asked to kill driver $driverId")
      drivers.get(driverId) match {
        case Some(runner) =>
          runner.kill()
        case None =>
          logError(s"Asked to kill unknown driver $driverId")
      }
    }

    case DriverStateChanged(driverId, state, exception) => {
      state match {
        case DriverState.ERROR =>
          logWarning(s"Driver $driverId failed with unrecoverable exception: ${exception.get}")
        case DriverState.FAILED =>
          logWarning(s"Driver $driverId exited with failure")
        case DriverState.FINISHED =>
          logInfo(s"Driver $driverId exited successfully")
        case DriverState.KILLED =>
          logInfo(s"Driver $driverId was killed by user")
        case _ =>
          logDebug(s"Driver $driverId changed state to $state")
      }
      master ! DriverStateChanged(driverId, state, exception)
      val driver = drivers.remove(driverId).get
      finishedDrivers(driverId) = driver
      memoryUsed -= driver.driverDesc.mem
      coresUsed -= driver.driverDesc.cores
    }

    case x: DisassociatedEvent if x.remoteAddress == masterAddress =>
      logInfo(s"$x Disassociated !")
      masterDisconnected()

    case RequestWorkerState =>
      sender ! WorkerStateResponse(host, port, workerId, executors.values.toList,
        finishedExecutors.values.toList, drivers.values.toList,
        finishedDrivers.values.toList, activeMasterUrl, cores, memory,
        coresUsed, memoryUsed, activeMasterWebUiUrl)

    case ReregisterWithMaster =>
      reregisterWithMaster()

  }

```
worker主要的任务就是接收消息启动LaunchExecutor，和LaunchDriver
###master
打开SPARK_HOME/sbin目录下的start-master.sh，其中调用脚本启动Master
```
"$sbin"/spark-daemon.sh start org.apache.spark.deploy.master.Master 1 --ip $SPARK_MASTER_IP --port $SPARK_MASTER_PORT --webui-port $SPARK_MASTER_WEBUI_PORT
```
master的启动主要是用来接收worker、client和appclient的信息，做相应的处理



#####下面看看从client向master发送注册driver的过程



当我们使用SPARK_HOME/bin/spark-submit脚本来提交我们的应用程序的时候，spark-submit脚本执行了
```sh
exec "$SPARK_HOME"/bin/spark-class org.apache.spark.deploy.SparkSubmit "${ORIG_ARGS[@]}"
```
找到spark下的deploy模块下的SparkSubmit类，经过一系列的选择，默认的是org.apache.spark.deploy.Client来作为客户端来提交driver程序->找到Client类
在preStart（）方法中向master提交driver程序
```
        val driverDescription = new DriverDescription(
          driverArgs.jarUrl,
          driverArgs.memory,
          driverArgs.cores,
          driverArgs.supervise,
          command)

        masterActor ! RequestSubmitDriver(driverDescription)   //给master发送RequestSubmitDriver注册driver信息
```
接下来看看master中接收到这里的消息之后具体做什么事情（找到master中receiveWithLogging方法）
```
 case RequestSubmitDriver(description) => {   //client会调用spark脚本bin/spark-submit.sh来提交作业，client给master发送RequestSubmitDriver消息
      if (state != RecoveryState.ALIVE) {
        val msg = s"Can only accept driver submissions in ALIVE state. Current state: $state."
        sender ! SubmitDriverResponse(false, None, msg)   //给client回送消息
      } else {
        logInfo("Driver submitted " + description.command.mainClass)  //main函数为org.apache.spark.deploy.worker.DriverWrapper
        val driver = createDriver(description)
        persistenceEngine.addDriver(driver)
        waitingDrivers += driver
        drivers.add(driver)                         //添加driver
        schedule()                                  //分配资源

        // TODO: It might be good to instead have the submission client poll the master to determine
        //       the current status of the driver. For now it's simply "fire and forget".

        sender ! SubmitDriverResponse(true, Some(driver.id),    //成功的话，发送消息给它，返回driver的id，为一串时间数字字符
          s"Driver successfully submitted as ${driver.id}")
      }
    }
```
如上面代码所示将driver添加到等待的drivers队列中去，spark默认是采用先进先出的方式来对driver调度
然后调用schedule()方法来对driver进行调度，看是否有足够的cpu来执行driver程序。如果成功的话给client发送SubmitDriverResponse信息
**下面具体看看schedule（）方法，这是整个deploy外部资源调度额核心**
```
 private def schedule() {  //给app或则driver分配资源 外部资源调度
    if (state != RecoveryState.ALIVE) { return }

    // First schedule drivers, they take strict precedence over applications
    // Randomization helps balance drivers
    val shuffledAliveWorkers = Random.shuffle(workers.toSeq.filter(_.state == WorkerState.ALIVE))  //洗牌机制，将ALIVEde worker选出来
    val numWorkersAlive = shuffledAliveWorkers.size
    var curPos = 0
    
    for (driver <- waitingDrivers.toList) { // iterate over a copy of waitingDrivers  //这里是为driver寻找有空闲核的worker来启动driver
      // We assign workers to each waiting driver in a round-robin fashion. For each driver, we
      // start from the last worker that was assigned a driver, and continue onwards until we have
      // explored all alive workers.
      var launched = false
      var numWorkersVisited = 0
      while (numWorkersVisited < numWorkersAlive && !launched) {
        val worker = shuffledAliveWorkers(curPos)
        numWorkersVisited += 1
        if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
          launchDriver(worker, driver)   //这里是选一个worker启动driver程序
          waitingDrivers -= driver
          launched = true
        }
        curPos = (curPos + 1) % numWorkersAlive
      }
    }

    // Right now this is a very simple FIFO scheduler. We keep trying to fit in the first app
    // in the queue, then the second app, etc.
    if (spreadOutApps) { //默认是为真，将一个app的任务尽量分配到所有的worker中去执行
      // Try to spread out each app among all the nodes, until it has all its cores
      for (app <- waitingApps if app.coresLeft > 0) {
        val usableWorkers = workers.toArray.filter(_.state == WorkerState.ALIVE)
          .filter(canUse(app, _)).sortBy(_.coresFree).reverse
        val numUsable = usableWorkers.length
        val assigned = new Array[Int](numUsable) // Number of cores to give on each node
        var toAssign = math.min(app.coresLeft, usableWorkers.map(_.coresFree).sum)
        var pos = 0
        while (toAssign > 0) {
          if (usableWorkers(pos).coresFree - assigned(pos) > 0) {
            toAssign -= 1
            assigned(pos) += 1
          }
          pos = (pos + 1) % numUsable
        }
        // Now that we've decided how many cores to give on each node, let's actually give them
        for (pos <- 0 until numUsable) {
          if (assigned(pos) > 0) {
            val exec = app.addExecutor(usableWorkers(pos), assigned(pos))
            launchExecutor(usableWorkers(pos), exec)   //在worker中为某个app launchExecutor，创建进程，这里是尽量将executor启动到各个worker中去
            app.state = ApplicationState.RUNNING       //跟改app的状态 ，每个app在每个worker中只有一个executor，这里也是验证了这个信息
          }
        }
      }
    } else {   //集中式的执行任务
      // Pack each app into as few nodes as possible until we've assigned all its cores
      for (worker <- workers if worker.coresFree > 0 && worker.state == WorkerState.ALIVE) {
        for (app <- waitingApps if app.coresLeft > 0) {
          if (canUse(app, worker)) {
            val coresToUse = math.min(worker.coresFree, app.coresLeft)
            if (coresToUse > 0) {
              val exec = app.addExecutor(worker, coresToUse)
              launchExecutor(worker, exec)
              app.state = ApplicationState.RUNNING
            }
          }
        }
      }
    }
  }

```
如上述注释代码所示：如果是driver的话调用launchDriver来启动driver如果是app的话，调用launchExecutor来启动Executor，整个资源的分配基本完备。

#####下面看看从Appclient向master发送注册app的过程
Client是由SparkDeploySchedulerBackend创建被启动的，因此client是被嵌入在每一个application中，只为这个applicator所服务，在client启动时首先会先master注册application：
```
override def preStart() {
      context.system.eventStream.subscribe(self, classOf[RemotingLifecycleEvent])
      try {
        registerWithMaster()           //向master注册自己
      } catch {
        case e: Exception =>
          logWarning("Failed to connect to master", e)
          markDisconnected()
          context.stop(self)
      }
    }
    def tryRegisterAllMasters() {
      for (masterUrl <- masterUrls) {
        logInfo("Connecting to master " + masterUrl + "...")
        val actor = context.actorSelection(Master.toAkkaUrl(masterUrl)) //获得master引用
        actor ! RegisterApplication(appDescription) //向master注册app
      }
    }
```
接下来看master接收到RegisterApplication消息之后具体做什么工作
```
 case RegisterApplication(description) => {    //app程序的入口程序sparkcontext启动创建调度模块之后会启动Appclient和Client不同
      if (state == RecoveryState.STANDBY) {
        // ignore, don't send response
      } else {
        logInfo("Registering app " + description.name)
        val app = createApplication(description, sender)
        registerApplication(app)                //注册app
        logInfo("Registered app " + description.name + " with ID " + app.id)  //
        persistenceEngine.addApplication(app)
        sender ! RegisteredApplication(app.id, masterUrl)
        schedule()                              //分配资源
      }
    }
```
首先调用registerApplication(app) 将该app添加到hashmap中最后还是会调用schedule() 对其进行资源调度。又回到了刚刚schedule（）方法了。整个过程结束。接下来分析一下一个app内部的任务调度、DAG调度等信息。
[this way]:http://jerryshao.me/architecture/2013/04/30/Spark%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B-deploy%E6%A8%A1%E5%9D%97/
