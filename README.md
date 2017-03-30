# Note about Java Thread pool

This is a reading note about Java Thread pool

## Example 1

MyTask.java

```java

public class MyTask implements Runnable{
  private int taskNum;
  
  public MyTask(int taskNum){
    this.taskNum = taskNum;
  }
  
  @Override
  public void run(){
    System.out.println("Thread is running: " + taskNum);
    try{
      Thread.sleep(4000);
    }catch(InterruptedException e){
      e.printStackTrace();
    }
    System.out.println("Task is done:"+ taskNum);
  }
}

```

Main.java

```java
public class TaskExecutor{
  public static void main(String[] args){
    //corePoolSize,maximumPoolSize,keepAliveTime,Unit,BlockingQueue
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
      5,10,200,TimeUnit.MILLISECONDS,new ArrayBlockingQueue<Runnable>(5));
    )
    for(int i=0; i< 5; i++){
      MyTask task = new MyTask(i);
      executor.execute(task);
      System.out.println("Threads in pool:" + executor.getPoolSize() + 
      "queue size:"+ executor.getQueue().size() + "completed: "+ executor.getCompletedTaskCount());
    }
    executor.shutdown();
  }
  
  // If there are i < 15, the problem will throw RejectedExecutionException since the maximumPoolSize is 10 
  // and there are 5 in the BlockingQueue.
  
  /*
  *
  Take this example. Starting thread pool size is 1, core pool size is 5, max pool size is 10 and 
  the queue is 100. As requests come in, threads will be created up to 5 and then tasks will be added 
  to the queue until it reaches 100. When the queue is full new threads will be created up to maxPoolSize. 
  Once all the threads are in use and the queue is full tasks will be rejected. As the queue reduces, so 
  does the number of active threads.
  http://stackoverflow.com/questions/17659510/core-pool-size-vs-maximum-pool-size-in-threadpoolexecutor
  */
}
```
Executor Diagram:

![Alt text](https://github.com/shao02/note/blob/master/Executor_Diagram.png "Executor Diagram")


- Executor、ExecutorService、ScheduledExecutorService are threadPool interfaces.
- ThreadPoolExecutor, ScheduledThreadPoolExecutor are threadPool implementations.

Let's look at implementation of ThreadPoolExecutor:

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

- corePoolSize: initially, there is 0 thread. if number of threads is greater than corePoolSize, threads will be put into the
Queue.  prestartAllCoreThreads() and prestartCoreThread() allows to 
- maximumPoolSize: maximum pool size, if threads is greater than this number, it throws RejectedExecutionException
- keepAliveTime: it only applies to threads greater than corePoolSize. allowCoreThreadTimeOut(boolean) for all threads.
- unit: keepAliveTime unit
- workQueue: ArrayBlockingQueue, LinkedBlockingQueue(bounded by Integer.MAX_VALUE), SynchronousQueue, PriorityBlockingQueue.

- rejectedExecutionHandler: there are a few handling ways -> AbortPolicy, CallerRunsPolicy, DiscardOldestPolicy, DiscardPolicy

AbstractExecutorService.java
```java
public abstract class AbstractExecutorService implements ExecutorService {
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) { };
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) { };
    public Future<?> submit(Runnable task) {};
    public <T> Future<T> submit(Runnable task, T result) { };
    public <T> Future<T> submit(Callable<T> task) { };
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks, boolean timed, long nanos) throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
    };
}
```

ExecutorService.java
```java
public interface ExecutorService extends Executor {
 
    void shutdown();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
 
    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```

Executor.java
```java
public interface Executor {
    void execute(Runnable command);
}
```

Summary: 
  * Executor is an interface, there is only one method -> execute(Runnable).
  * ExecutorService extends Executor with methods submit, invokeAll, invokeAny, and shutDown etc.
  * AbstractExecutorService extends ExecutorService, implements most of the methods
  * ThreadPoolExecutorService extends AbstractExecutorService
  * Key methods for ThreadPoolExecutorService
    * execute() - submit a task for the threadPool to execute
    * submit() - return task result
      ```
      A task queued with execute() that generates some Throwable will cause the UncaughtExceptionHandler 
      for the Thread running the task to be invoked. The default UncaughtExceptionHandler, which typically 
      prints the Throwable stack trace to System.err, will be invoked if no custom handler has been 
      installed. 
      
      On the other hand, a Throwable generated by a task queued with submit() will bind the 
      Throwable to the Future that was produced from the call to submit(). Calling get() on that Future 
      will throw an ExecutionException with the original Throwable as its cause (accessible by calling 
      getCause() on the ExecutionException).
      ```
    * shutdown() - 
    * shutdownNow()
