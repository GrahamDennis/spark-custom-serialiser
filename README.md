# Spark Custom Serialisation

Spark cannot currently use a custom serialiser that is shipped with the application jar.  Trying to do this causes a `java.lang.ClassNotFoundException` when trying to instantiate the custom serialiser in the Executor processes.  This occurs because Spark attempts to instantiate the custom serialiser *before* the application jar has been shipped to the Executor process.  This repository is a simple demonstration of the problem.

I've verified this problem in Spark 1.0.2.

To build this repository and demonstrate the problem, run:

    # Build the .jar file
    ./gradlew build -i
    # Run the application locally but in cluster-simulation mode
    spark-submit --class org.example.SparkDriver --master local-cluster[2,1,512] build/libs/spark-custom-serialiser-0.0.1.jar
    # Note that succeeds using --master local because the application jar is available to the process at launch time.

The problem appears in the master logs as:

    Exception in thread "main" org.apache.spark.SparkException: Job aborted due to stage failure: Task 1.0:0 failed 1 times, most recent failure: TID 0 on host 10.202.68.146 failed for unknown reason
    Driver stacktrace:
    	at org.apache.spark.scheduler.DAGScheduler.org$apache$spark$scheduler$DAGScheduler$$failJobAndIndependentStages(DAGScheduler.scala:1044)
    	at org.apache.spark.scheduler.DAGScheduler$$anonfun$abortStage$1.apply(DAGScheduler.scala:1028)
    	at org.apache.spark.scheduler.DAGScheduler$$anonfun$abortStage$1.apply(DAGScheduler.scala:1026)
    	at scala.collection.mutable.ResizableArray$class.foreach(ResizableArray.scala:59)
    	at scala.collection.mutable.ArrayBuffer.foreach(ArrayBuffer.scala:47)
    	at org.apache.spark.scheduler.DAGScheduler.abortStage(DAGScheduler.scala:1026)
    	at org.apache.spark.scheduler.DAGScheduler$$anonfun$handleTaskSetFailed$1.apply(DAGScheduler.scala:634)
    	at org.apache.spark.scheduler.DAGScheduler$$anonfun$handleTaskSetFailed$1.apply(DAGScheduler.scala:634)
    	at scala.Option.foreach(Option.scala:236)
    	at org.apache.spark.scheduler.DAGScheduler.handleTaskSetFailed(DAGScheduler.scala:634)
    	at org.apache.spark.scheduler.DAGSchedulerEventProcessActor$$anonfun$receive$2.applyOrElse(DAGScheduler.scala:1229)
    	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:498)
    	at akka.actor.ActorCell.invoke(ActorCell.scala:456)
    	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:237)
    	at akka.dispatch.Mailbox.run(Mailbox.scala:219)
    	at akka.dispatch.ForkJoinExecutorConfigurator$AkkaForkJoinTask.exec(AbstractDispatcher.scala:386)
    	at scala.concurrent.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
    	at scala.concurrent.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
    	at scala.concurrent.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
    	at scala.concurrent.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

The root cause of the problem appears in the worker node logs as:

    14/08/19 11:46:51 ERROR OneForOneStrategy: org.example.WrapperSerializer
    java.lang.ClassNotFoundException: org.example.WrapperSerializer
        at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        at java.lang.Class.forName0(Native Method)
        at java.lang.Class.forName(Class.java:270)
        at org.apache.spark.SparkEnv$.instantiateClass$1(SparkEnv.scala:161)
        at org.apache.spark.SparkEnv$.instantiateClassFromConf$1(SparkEnv.scala:182)
        at org.apache.spark.SparkEnv$.create(SparkEnv.scala:185)
        at org.apache.spark.executor.Executor.<init>(Executor.scala:87)
        at org.apache.spark.executor.CoarseGrainedExecutorBackend$$anonfun$receiveWithLogging$1.applyOrElse(CoarseGrainedExecutorBackend.scala:60)
        at scala.runtime.AbstractPartialFunction$mcVL$sp.apply$mcVL$sp(AbstractPartialFunction.scala:33)
        at scala.runtime.AbstractPartialFunction$mcVL$sp.apply(AbstractPartialFunction.scala:33)
        at scala.runtime.AbstractPartialFunction$mcVL$sp.apply(AbstractPartialFunction.scala:25)
        at org.apache.spark.util.ActorLogReceive$$anon$1.apply(ActorLogReceive.scala:53)
        at org.apache.spark.util.ActorLogReceive$$anon$1.apply(ActorLogReceive.scala:42)
        at scala.PartialFunction$class.applyOrElse(PartialFunction.scala:118)
        at org.apache.spark.util.ActorLogReceive$$anon$1.applyOrElse(ActorLogReceive.scala:42)
        at akka.actor.ActorCell.receiveMessage(ActorCell.scala:498)
        at akka.actor.ActorCell.invoke(ActorCell.scala:456)
        at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:237)
        at akka.dispatch.Mailbox.run(Mailbox.scala:219)
        at akka.dispatch.ForkJoinExecutorConfigurator$AkkaForkJoinTask.exec(AbstractDispatcher.scala:386)
        at scala.concurrent.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
        at scala.concurrent.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
        at scala.concurrent.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
        at scala.concurrent.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

