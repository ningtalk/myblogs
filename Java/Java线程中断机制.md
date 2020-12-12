# Java线程中断机制

<优雅停止>



## Java里线程中断方法

在Thread类里与中断有关的`public`的方法有如下三个：
* public void interrupt()
* public boolean isInterrupted()
* public static boolean interrupted()
  
前两个方法都是属于对象方法，最后一个属于类方法，下面对这三个方法的使用进行说明。 
在Java线程实现里，每个线程都有一个类型为boolean类型的中断标志位，默认false。  

### public void interrupt()
当调用一个线程的interrupt方法时，会将此线程的中断标志位置为`true`，仅此而已。  
* 当线程处于阻塞状态(wait,sleep,join等)时，如果调用此线程的interrupt方法，则会抛出InterruptException异常。
* 当线程处于运行状态时，如果调用此线程的interrupt方法，并不会影响线程的执行，线程可通过isInterrupted方法来检测中断标志位来判断是否应该中断，至于是选择中断操作还是继续运行由此线程自身决定。

### public boolean isInterrupted()
线程可通过调用isInterrupted方法来获取中断标志位。

### public static boolean interrupted()
interrupted()方法是Thread类的静态方法，可以看下这个方法的具体实现，  
```java
public class Thread implements Runnable {
    // 省略...
    
    
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
    
    // 省略...
}
```
通过上面的代码可以很明确的得知，interrupted方法用于获取当前线程的线程标志位，且将标志位归为原始值false。
  
## InterruptException
