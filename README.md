# 关于spark的简单分析以及工作中的一些总结
心血来潮，想spark进行一些深层次的分析，同时也不脱离实践操作，所以有关spark的一些分析，尽量会结合实践附上简单的demo操作。Spark源码采用的是1.2.0版本。
###简单介绍
作为对工作的总结，这边记录了基本上从零开始如何去做一些spark以及hadoop相关的工作，这里开始会将的很基础，从如何安装编译Spark、Spark如何提交任务开始，然后会对Spark的一些重要特性结合进行分析，其中不乏引用好的Spark相关内容（我会以引用的形式标识出来，如有侵权，请和我联系，我及时修正）。在分析Spark过程中会有很多例子程序，其中会有很多是Spark自带的例子也会有网上看到好的例子程序，我都会给程序赋予详细的注释。同时也会结合我实际的工作，把我遇到的实际问题在这边进行详细分析。
###主要内容
>1.[Build and install Spark]-安装和编译Spark

>2.[RDD details]-详细介绍RDD的用法以及其实质

>3.[Spark submit jobs]-从脚本开始介绍Spark如何从jar文件提交任务

>4.[Job executing and task scheduling]-介绍Spark内部如何调度和执行任务

>5.[Deploy]-分析Spark Deploy模块

>6.[SparkStreaming]-sparkStreaming模块源码简单分析

>7.[Custom receiver]-如何自定义SparkStreaming接收器

>8.[Shuffle]-研究Spark Shuffle并和hadoop比较

>9.[Spark fault tolerant]-研究Spark血统容错并和hadoop进行比较

>10.[Broadcast]-介绍Spark全局变量broadcast

>11.[Spark-sql]-介绍Spark sql

>12.[Spark-Mllib]-介绍Spark Mllib

>13.[Spark-Graphx]-介绍Spark Graphx

>14.[Tachyon]-介绍Tachyon

###其它问题
在实际工作中，会遇到其他各种问题，这里将一些重要的问题也以文档的形式记录下来。
>1.[receiver]-分析spark源码，修改sparkStreaming模块源码让其支持动态的添加和停止流

>2.[intellij]-使用intellij打包的一些问题。

[Build and install Spark]:https://github.com/gjhkael/deployDoc/blob/master/2.Build-and-install-Spark.md
[Deploy]:https://github.com/gjhkael/deployDoc/blob/master/spark%20deploy%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md
[SparkStreaming]:https://github.com/gjhkael/deployDoc/blob/master/SparkStreaming%E5%88%86%E6%9E%90.md
[receiver]:https://github.com/gjhkael/deployDoc/blob/master/%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E6%B5%81%E6%8E%A5%E6%94%B6.md
[intellij]:https://github.com/gjhkael/deployDoc/blob/master/intellijExportJar.md
