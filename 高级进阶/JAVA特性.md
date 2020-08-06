# Java New一个对象分配多少内存

# CountDownLatch(1.5引入)
CountDownLatch这个类能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。 

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。 

https://blog.csdn.net/scandly_java/article/details/51333231


# CyclicBarrier(1.5引入)

# Semaphore(1.5引入)

# concurrentHashMap(1.5引入)

# BlockingQueue(1.5引入)

# LinkedBlockingQueue()
有界队列

# AtomicInteger

# volatile
集群中的zookeeper需要超过半数，整个集群对外才可用

# PriorityQueue


# ArrayDeque

String.intern()



