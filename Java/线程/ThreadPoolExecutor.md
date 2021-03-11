# ThreadPoolExecutor

## 为什么需要线程池？
Java中创建线程通过new Thread()即可完成创建，与创建一个普通Java类一样。为什么需要线程池来创建线程？实际在使用new Thread()创建线程时
需要调用内核API并申请一些相关资源，因此可以任务创建线程（切换、销毁）开销是较大的，所以使用线程池来维护一组线程可以提升性能。

## ThreadPoolExecutor使用
### 构造方法
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```
以上为ThreadPoolExecutor的构造方法，参数含义如下：
* corePoolSize：核心线程数，线程池中最小工作线程数（即使是空闲的状态），也可以通过allowCoreThreadTimeOut来设置核心线程空闲时间
* maximumPoolSize：最大线程数，线程池中最大的线程数
* keepAliveTime & unit：当线程数超过核心线程数时，过量的空闲线程最大的空闲时间，unit为时间单位
* workQueue：工作队列，通过execute方法提交的线程会进入工作队列
* threadFactory：创建线程的工厂，
* handler：拒绝策略，默认的拒绝策略是抛出RejectedExecutionException异常

### 使用注意事项
1.workQueue必须使用有界队列，如果使用无界队列可能导致OOM，这也是不推荐使用Executors工具类来创建线程池的原因。  
2.拒绝策略：当任务过多时，超出工作队列大小，会触发拒绝策略。默认的拒绝策略是抛出RejectedExecutionException异常，在实际应用中要慎用
默认拒绝策略，应该自定义拒绝策略，且自定义拒绝策略往往与降级配合使用。  

### 合适的线程数
首先，肯定不是越多越好，线程的创建、切换、销毁的开销都比较大，过多的线程可能会有性能问题。当创建线程池时需要确定线程池的核心线程数，多
少最合适？从网上查看资料得到对于IO密集型的应用（大部分实践应用）线程的个数可参考 `最佳线程数 =CPU 核数 * [ 1 +（I/O 耗时 / CPU 耗时）]`，当然具体设置多少合适，通过压测数据来决定是最准确的。  
在实际使用中，经常会使用int coreNum = Runtime.getRuntime().availableProcessors()作为CPU的核数来设置线程数量，比如设置核心线程数为coreNum + 1，最大线程数为coreNum * 2等。需要注意的是现在很多应用部署容器化，通过Runtime.getRuntime().availableProcessors()获取的可能是物理机的CPU数量。

## 源码分析
参考https://www.cnblogs.com/sanzao/p/10712778.html
