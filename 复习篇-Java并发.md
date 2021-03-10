# Java并发

## Java多线程实现方式

### 1. 继承Thread类，重写run()方法

- 每次创建一个新的线程，都要新建一个Thread子类的对象；
- 启动线程，new  Thread子类().start()；
- 创建线程实际调用的是父类的Thread空参的构造器，`public Thread()`；

```java
public class ThreadTest01 {

    public static void main(String[] args){
        for (int i = 0; i < 10; i ++) {
            new ExtendsThread().start();
        }
        System.out.println(Thread.currentThread().getName());
    }

}

class ExtendsThread extends Thread {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}
```



### 2. 实现Runnable接口，重写run()方法

```java
public interface Runnable {
    public abstract void run();
}
```

- 不论创建多少个线程，只需要创建**一个Runnable接口实现类的对象**；
- 启动线程，new  Thread (Runnbale接口实现类的对象).start()
- 创建线程调用的是Thread类Runnable类型参数的构造器，`public Thread(Runnable target)`；

```java
public class ThreadTest02 {

    public static void main(String[] args) {
        ImplThread implThread = new ImplThread();
        for (int i = 0; i < 10; i++) {
            new Thread(implThread).start();
        }
        System.out.println(Thread.currentThread().getName());
    }

}

class ImplThread implements Runnable {
    private volatile int i = 0;
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "---" + i++);
    }
}
```

```java
// Lambda 表达式创建线程
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        synchronized (A) {
            synchronized (B) {
                System.out.println("2");
            } 
        }
    }
});
```





### 3.  实现Callable接口，重写call方法(有返回值)

```java
public interface Callable<V> {
    V call() throws Exception;
}
```

- 自定义类实现Callable接口时，必须指定泛型，该泛型即返回值的类型；
- 每次创建一个新的线程，都要创建一个新的Callable接口的实现类；
- 如何启动线程？
  - 创建一个Callable接口的实现类的对象；
  - 创建一个FutureTask对象，传入Callable类型的参数，`public FutureTask(Callable<V> callable){...}`；
  - 调用Tread类重载的参数为Runnable的构造器创建Thread对象

```shell
# 将FutureTask作为参数传递
# public class FutureTask<V> implements RunnableFuture<V>
# public interface RunnableFuture<V> extends Runnable, Future<V>
```

- 获得返回值：调用FutureTask的get()方法

```java
// Java实现线程的方式3：实现Callable接口，重写call方法
public class ThreadTest03 {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        for (int i = 0; i < 3; i ++) {
            Callable<Integer> implCallable = new ImplCallable();
            FutureTask<Integer> futureTask = new FutureTask<>(implCallable);
            new Thread(futureTask).start();
            System.out.println(Thread.currentThread().getName() + "---" + futureTask.get());
        }
        System.out.println(Thread.currentThread().getName());
    }

}

class ImplCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        int result = 0;
        for(int i=0;i<10;i++){
            result += i;
        }
        System.out.println(Thread.currentThread().getName());
        return result;
    }
}
```



### 4.  使用线程池(有返回值)

- Executors类

```java
/**
 *
 * 线程池
 * 跟数据库连接池类似
 * 避免了线程的创建和销毁造成的额外开销
 *
 * java.util.concurrent
 *
 * Executor    负责现成的使用和调度的根接口
 *    |--ExecutorService    线程池的主要接口
 *          |--ThreadPoolExecutor    线程池的实现类
 *          |--ScheduledExecutorService    接口，负责线程的调度
 *              |--ScheduledThreadPoolExecutor    (extends ThreadPoolExecutor implements ScheduledExecutorService)
 *
 *
 * Executors工具类
 * 提供了创建线程池的方法
 *
 */
public class ThreadPool {
    public static void main(String[] args){

        //使用Executors工具类中的方法创建线程池
        ExecutorService pool = Executors.newFixedThreadPool(5);

        ThreadPoolDemo demo = new ThreadPoolDemo();

        //为线程池中的线程分配任务,使用submit方法，传入的参数可以是Runnable的实现类，也可以是Callable的实现类
        for(int i=1;i<=5;i++){
            pool.submit(demo);
        }

        //关闭线程池
        //shutdown ： 以一种平和的方式关闭线程池，在关闭线程池之前，会等待线程池中的所有的任务都结束，不在接受新任务
        //shutdownNow ： 立即关闭线程池
        pool.shutdown();


    }
}
class ThreadPoolDemo implements Runnable{

    /**多线程的共享数据*/
    private int i = 0;

    @Override
    public void run() {
        while(i<=50){
            System.out.println(Thread.currentThread().getName()+"---"+ i++);
        }
    }
}
```





## Java并发编程的艺术

### 并发编程的挑战

```shell
# 上下文切换
	CPU通过时间片分配算法来循环执行任务，当前任务执行一个时间片后会切换到下一个任务。但是，在切换前会保存上一个任务的状态，以便下次切换回这个任务时，可以再加载这个任务的状态。所以任务从保存到再加载的过程就是一次上下文切换。
	线程有创建和上下文切换的开销，所以并发执行并不一定比串行快。

# 减少上下文切换
	1. 无锁并发编程
		多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些方法来避免使用锁，如将数据的ID按照Hash算法取模分段，不同的线程处理不同段的数据。
	2. CAS算法
		Java的Atomic包使用CAS算法来更新数据，不需要加锁，(JVM字节码指令中有CAS比较，不需要加锁)
	3. 使用最少线程
		避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会造成大量线程都处于等待状态。
	4. 使用协程
		在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换。
```



```shell
# 死锁
	死锁：两个线程互相等待对方线程释放锁；
	避免死锁的常用方法：
		1. 避免一个线程同时获取多个锁
		2. 避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源
		3. 尝试使用定时锁，使用 lock.tryLock(timeout)来代替使用内部锁机制
		4. 对于`数据库锁`，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况
```



### Java并发机制的底层实现原理

- Java中所使用的并发机制依赖于JVM的实现和CPU的指令；

#### volatile的应用

`volatie`变量作用：

- volatile是轻量级的synchronized，在多处理器开发中保证了共享变量的可见性；
- volatile可以防止指令重排序；

```shell
# volatile 防止指令重排序：
	编译后插入内存屏障；内存屏障：是一组处理器指令，用于实现对内存操作的顺序限制；
# volatile 变量保证可见性：
	有volatile修饰的共享变量进行写操作的时候会多出 `lock addl $0x0` 指令：
		1) Lock前缀指令会引起处理器缓存写回到内存；
			Lock信号会锁定具体内存区域的缓存并回写到内存，并且使用`缓存一致性机制`来确保修改的原子性，此操作被称为“缓存锁定”，缓存一致性机制会阻止同时修改两个以上处理器缓存的内存区域数据。
		2) 一个处理器的缓存回写到内存会到导致其他处理器的缓存无效；
			在多核处理器系统进行操作的时候，处理器能`嗅探`其他处理器访问系统内存和它们的内部缓存。处理器使用嗅探技术保证它的内部缓存、系统内存和其他处理器的缓存的数据在总线上保持一致。例如：通过嗅探一个处理器来检测其他处理器打算写内存地址，而这个地址当前处于共享状态，那么正在嗅探的处理器将使它的缓存行无效，在下次访问相同内存地址的时候，强制执行缓存行填充。

# volatile 的使用优化
	处理器的L1、L2、L3缓存的高速缓存行是64字节宽，使用追加到64字节的方式来填满高速缓冲区的缓存行，避免两个变量加载到同一个缓存行，使头、尾节点在修改时不会互相锁定。
```



#### synchronized 的实现原理与应用(锁的升级)

利用synchronized实现同步的基础：Java中的每一个对象都可以作为锁(任何对象都有对应的monitor与之关联，获取monitor对象的所有权即获得对象的锁)；具体表现为以下3种形式：

- 对于普通同步方法，锁是当前实例对象；
- 对于静态同步方法，锁是当前类的Class对象；
- 对于同步方法块，琐是Synchronized括号里配置的对象；

JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，monitorenter指令在编译后插入到同步代码块的开始位置，monitorexit执行编译后插到方法结束处和异常处；线程执行到monitorenter指令时，会尝试获取对象所对应的monitor的所有权，即获得对象的锁。

```shell
# Java 对象头
	synchronized 用的锁是存在Java对象头里的。如果对象是数组类型，则虚拟机用3个字宽(Word)存储对象头，如果对象是非数组类型，则用2个字宽存储对象头。
	1) Mark Word: 存储对象的hashCode或锁信息；
	2) Class Metadata Address: 存储对象到对象类型数据的指针；
	3) Array Length：数组的长度；
```

```shell
# 锁的升级和对比
参考博客：https://blog.csdn.net/lengxiao1993/article/details/81568130
  锁的4种状态(由低到高)：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态
#  1. 偏向锁状态
  	偏向锁的获取方式是将对象头的MarkWord部分，标记上线程ID，以表示哪一个线程获得了偏向锁；首先读取目标对象的MarkWord判断是否处于可偏向的状态；
  	如果为可偏向状态，则尝试用CAS操作，将自己的线程ID写入MarkWord；如果CAS操作成功，则认为已经获取到该对象的偏向锁，执行同步块代码；如果CAS操作失败，则说明有另一位一个线程Thread B抢先获得了偏向锁，此时需要撤销Thread B获得的偏向锁，将Thread B持有的锁升级为轻量级锁，撤销操作需要等待全局安全点(没有线程在执行字节码)。
  	如果是已偏向状态，则检测MarkWord中存储的Thread ID是否等于当前Thread ID；如果相等则证明本线程已经获取到偏向锁，可以继续执行同步代码块；如果不等，则证明该对象目前偏向于其他线程，需要撤销偏向锁。
  	偏向锁的撤销：偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点(在这个时间点上没有正在执行的字节码)。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置为无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的MarkWord要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

#  2. 轻量级锁
	轻量级锁加锁过程：
		1) 在代码进入同步块的时候，如果同步对象为无锁状态(锁标志位为01状态，是否偏向锁为0)，虚拟机首先将在当前线程的栈帧中建立一个名为锁记录(Lock Record)的空间，用于存储锁对象目前的MarkWord的拷贝，官方称为Displaced Mark Word；
		2) 拷贝对象头中的Mark Word复制到锁记录中；
		3) 拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock Record里的owner指针指向object mark word；
		4) 如果更新成功，那么这个线程就拥有了对象的锁，并且将对象Mark Word的锁标志位设置为00，即表示此对象处于轻量级锁定状态；
		5) 如果更新失败，当前线程尝试使用自旋来获取锁；
	轻量级锁解锁
		轻量级锁解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头；如果成功则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

#  3. 重量级加锁过程
	轻量级锁在向重量级锁膨胀的过程中，一个操作系统的互斥量(mutex)和条件变量(condition variable)会和被这个锁的对象关联起来。具体而言，在锁膨胀时，被锁对象的MarkWord会被通过CAS操作尝试更新为一个数据结构的指针，这个数据结构中进一步包含了指向操作系统互斥量和条件变量的指针。
	为了改善性能， 使得 JVM 会根据竞争情况， 使用如下 3 种不同的锁机制
    偏向锁（Biased Lock ）
    轻量级锁（ Lightweight Lock）
    重量级锁（Heavyweight Lock）
上述这三种机制的切换是根据竞争激烈程度进行的，在几乎无竞争的条件下，会使用偏向锁，在轻度竞争的条件下，会由偏向锁升级为轻量级锁，在重度竞争的情况下，会升级到重量级锁。
```



#### 原子操作的实现原理

处理器提供总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性：

- 使用总线锁保证原子性：总线锁就是使用处理器提供的一个 LOCK 信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存；
- 使用缓存锁保证原子性：缓存锁定是指内存区域如果被缓存在处理器的缓存行中，并且在Lock操作期间被锁定，那么当它执行锁操作写回内存时，处理器不在总线上声言LOCK信号，而是修改内部的内存地址，并允许它的缓存一致性来保证操作的原子性。缓存一致性：阻止同时修改两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。

```shell
# Java 如何实现原子操作？
	在Java中可以通过锁和循环CAS的方式来实现原子操作；
	1) 使用循环CAS实现原子操作
		问题：ABA问题  ---  解决方案：使用版本号
		循环时间长开销大   ---  解决方案：JVM支持处理器提供的pause指令
		只能保证一个共享变量的原子操作
	2) 使用锁机制实现原子操作
		锁机制保证了只有获得锁的线程才能操作锁定的内存区域。JVM实现锁的方式使用了循环CAS获取和释放锁。
```



### Java内存模型

本章大致分为4部分：

- Java 内存模型的基础，主要介绍内存模型相关的基本概念；
- Java内存模型中的顺序一致性，主要介绍重排序与顺序一致性内存模型；
- 同步原语：主要介绍3个同步原语(synchronized、volatile和final)的内存语义及重排序规则在处理器中的实现；
- Java内存模型的设计，主要介绍Java内存模型的设计原理，及其与处理器内存模型和顺序一致性内存模型的关系



#### Java内存模型的基础

在并发编程中，需要处理两个关键问题：线程之间如何通信及线程之间如何同步。通信是指两个线程之间以何种机制来交换信息，同步是指程序中用于控制不同线程间操作发生相对顺序的机制。Java 采用的是共享内存的并发模型，线程之间共享程序的公共状态，通过写 - 读内存中的公共状态进行通信。在共享内存的并发模型中，同步是显示进行的，程序员必须显示指定某个方法或某段代码需要在线程之间互斥执行。

本地内存(或工作内存)是JMM的一个抽象概念，并不真实存在，它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

JMM(Java Memory Model) 通过控制主内存与每个线程的本地内存(或工作内存)之间的交互，来为Java程序员提供内存的可见性。

```shell
# Java 源代码到最终实际执行的指令序列，会分别经历下面3种重排序：
	1. 源代码
	2. 编译器优化重排序
	3. 指令级并行重排序：若不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
	4. 内存系统重排序：处理器使用缓存和读/写缓冲区
	5. 最终执行的指令序列
```

现代的处理器使用写缓冲区临时保存向内存写入的数据，由于写缓冲区仅对自己的处理器可见，所以会导致处理器执行内存操作的顺序可能会与内存实际的操作顺序不一致，处理器允许写 - 读操作进行重排序(store - load重排序)。为了保证内存可见性，Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。

```shell
# 屏障类型：StoreLoad Barriers
# 指令示例：Store1; StoreLoad; Load2
# 说明：确保Store1数据对其他处理器变得可见(指刷新到内存)先于Load2及所有后续装载指令的装载。StoreLoad Barriers会使该屏障之前的所有内存访问指令(存储和装载指令)完成之后，才执行屏障之后的内存访问指令。
```

happens - before 关系：一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens - before 关系；

```shell
# 与程序员密切相关的 happens - before 规则如下：
#  1. 程序顺序规则：一个线程中的每个操作，happens - before 于该线程中的任意后续操作；
#  2. 监视器锁规则：对一个锁的解锁，happens - before 于随后对这个锁的加锁；
#  3. volatile变量规则：对一个volatile域的写，happens - before 于任意后续对这个volatile域的读；
#  4. 传递性
```



#### 重排序

重排序是指**编译器和处理器**为了优化程序性能而对指令序列进行重新排序的一种手段。

**数据依赖性**：若两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性，读后写，写后读，写后写。数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作。不同处理器之间和不同线程之间的数据依赖性不能被编译器和处理器考虑。

**as - if - serial**：as - if - serial 语义的意思是：不管怎么重排序，单线程程序的执行结果都不能被改变。

**控制依赖性**：当代码中存在控制依赖性时，会影响指令排列序列的并行度。编译器会采用猜测(Speculation)执行来克服控制相关性对并行度的影响。单线程程序中，对存在控制依赖性的操作重排序不会改变执行结果；但在多线程程序中，对存在控制依赖的操作重排序，可能会改变程序的执行结果。

```java
//  控制依赖性
if (flag) {
    int i = a * a;
}
// 单线程中控制依赖性采用猜测 Speculation
temp = a * a;
if (flag)
    int i = temp;
```



#### 顺序一致性内存模型

顺序一致性内存模型是一个理论参考模型，两大特性：

- 一个线程中的所有操作必须按照程序的顺序来执行；
- (不管线程是否同步)所有线程都只能看到一个单一的操作执行顺序。在顺序一致性内存模型中，每个操作都必须原子执行且立刻对所有线程可见。

同步程序的顺序一致性效果：在JMM中，正确同步的多线程程序临界区内的代码可以重排序；

未同步程序的执行特性：未正确同步的多线程程序，JMM只提供最小安全性：线程执行时读取到的值，要么是某个线程之前写入的值，要么是默认值(0, Null, false)。



#### volatile 的内存语义

volatile 变量自身具有下列特性：

- 可见性。对一个volatile变量的读，总是能看到(任意线程)对这个volatile变量最后的写入；
- 原子性。对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性。

volatile 写 - 读建立的 happens - before 关系：volatile变量规则：对一个volatile域的写，happens - before 于任意后续对这个volatile域的读；

volatile 写的内存语义：当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。

- 在每个volatile写操作的**前面**插入一个StoreStore屏障，目的：保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见；
- 在每个volatile写操作的**后面**插入一个StoreLoad屏障，目的：避免volatile写与后面有可能的volatile读/写操作重排序；

volatile 读的内存语义：当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效，线程接下来将从主内存中读取共享变量。

- 在每个volatile读操作的**后面**插入一个LoadLoad屏障，目的：用来禁止处理器把上面的volatile读与下面的普通读重排序；
- 在每个volatile读操作的**后面**插入一个LoadStore屏障，目的：用来禁止处理器把上面的volatile读与下面的普通写重排序；



#### 锁的内存语义

当线程释放锁时，JMM会把线程对应本地内存中的共享变量刷新到主内存中；当线程获取锁时，JMM会把该线程对应的本地内存置为无效，从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。对比锁释放 - 获取的内存语义与volatile写 - 读的内存语义可以看出：锁释放与volatile写有相同的内存语义，锁获取与volatile读有相同的内存语义。

```shell
# ReentrantLock 分析锁内存语义的具体实现机制, 源码：
public class ReentrantLock implements Lock, java.io.Serializable {
	private final Sync sync;
	abstract static class Sync extends AbstractQueuedSynchronizer {
		//  公平锁和非公平锁的释放方法
		protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }	
	}
	// Sync object for non-fair locks
	static final class NonfairSync extends Sync {
		final void lock();
	}
	static final class FairSync extends Sync {
		final void lock() {
            acquire(1);
        }
		protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
	}	
}
  ReentrantLock 的实现依赖于Java同步器框架 AbstractQueuedSynchronizer(AQS)。AQS 使用一个整型的volatile变量(命名为state)来维护同步状态。
  ReentrantLock 分为公平锁和非公平锁；
  	使用公平锁时，加锁方法lock()调用轨迹如下：
  	ReentrantLock:lock()  --->  FairSync:lock()  --->  AbstractQueuedSynchronizer:acquire(int arg)  ---> FairSync:tryAcquire(int acquires)
  	解锁方法unlock()调用轨迹如下：
  	ReentrantLock:unlock()  ---> AbstractQueuedSynchronizer:release(int arg)  --->  Sync:tryRelease(int releases)
  	公平锁在释放锁的最后写volatile变量state，在获取锁时首先读这个volatile变量。根据volatile的 happens-before 规则，释放锁的线程在volatile变量之前可见的共享变量，在获取锁的线程读取同一个volatile变量后将立即变得对获得锁的线程可见。
  	Java中的compareAndSet()方法调用称为CAS，CAS通过处理器的lock指令可以保证同时具有volatile读和volatile写的内存语义。
  	公平锁与非公平锁的内存语义总结：
  		公平锁和非公平锁释放时，最后都要写一个volatile变量state(该变量在AQS中)；
  		公平锁获取时，首先读volatile变量；
  		非公平锁获取时，首先会用CAS更新volatile变量，这个操作同时具有volatile读和volatile写的内存语义。
```

![4](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210128210410217.png)

#### final域的内存语义

写final域的重排序规则禁止把final域的写重排序到构造函数之外。写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保证。

- JMM禁止编译器把final域的写重排序到构造函数之外。
- 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。

读final域的重排序规则是，在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作。编译器会在读final域的前面插入一个LoadLoad屏障。初次读对象引用与初次读该对象包含的final域，这两个操作之间存在间接依赖关系，由于编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。**读final域的重排序可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。**

**final域为引用类型**：对于引用类型，写final域的重排序规则对编译器和处理器增加了如下约束：在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

**final语义在处理器中的实现**：由于X86处理器不会对写 - 写操作做重排序，所以在X86处理器中，写final域需要的StoreStore屏障会被省略掉；同样，由于X86处理器不会对存在间接依赖关系的操作做重排序，所以读final域需要的LoadLoad屏障也会被省略掉。



#### happens-before

JMM可以通过happens - before 关系向程序员提供跨线程的内存可见性保证。

as-if-serial语义给编写单线程程序的程序员创造了一个幻境：单线程程序员是按程序的顺序来执行的。happens - before关系给正确同步的多线程程序员创造了一个幻境：正确同步的多线程程序是按happens - before指定的顺序来执行的。

```shell
# happens - before 规则
	1. 程序顺序规则：一个线程中的每个操作，happens - before 于该线程中的任意后续操作；
	2. 监视器锁规则：对一个锁的解锁，happens - before 与随后对这个锁的加锁；
	3. volatile变量规则：对一个volatile域的写，happens - before 于任意后续对这个volatile域的读；
	4. 传递性：
	5. start()规则：如果线程A执行ThreadB.start()(启动线程B)，那么A线程的ThreadB.start()操作happens - before 于线程B的任意操作，线程A在ThreadB.start()之前对共享变量所做的修改，在线程B开始执行后都将确保对线程B可见；
	6. join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens - before 于线程A从ThreadB.join()操作成功返回；线程A执行ThreadB.join()并成功返回后，线程B中的任意操作都将对线程A可见。
```



### Java并发编程基础

#### 线程简介

现代操作系统在运行一个程序时，会为其创建一个进程。例如，启动一个Java程序，操作系统就会创建一个Java进程。现代操作系统调度的最小单元是线程，也叫轻量级进程(Light  Weight  Process)，在一个进程里可以创建多个线程，这些线程都有各自的计数器、堆栈和局部变量等属性，并且能够访问共享的内存变量。处理器在这些线程上高速切换，让使用者感觉到这些线程是在同时执行。

**Java线程的状态：**

| 状态名称     | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| NEW          | 初始状态，线程被构建，但是还没有调用start()方法              |
| RUNNABLE     | 运行状态，Java线程将操作系统中的就绪和运行两种状态称为“运行中” |
| WAITING      | 等待状态，表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作(通知或中断) |
| TIME_WAITING | 超时等待状态，该状态不同于WAITING，可以在指定的时间自行返回  |
| BLOCKED      | 阻塞状态，表示线程阻塞于锁                                   |
| TERMINATED   | 终止状态，表示当前线程已经执行完毕                           |

![image-20210129213456791](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210129213456791.png)



#### 启动和终止线程

**构造线程**

一个新构造的线程对象是由其parent线程来进行空间分配的，当前线程就是其该线程的父线程`Thread parent = currentThread()`；而child线程继承了parent是否为Daemon、优先级和加载资源的contextClassLoader以及可继承的ThreadLocal，同时还会分配一个唯一的ID来标识这个child线程。至此，一个能够运行的线程对象就初始化好了，在**堆内存**中等待着运行。

**安全地终止线程**

中断操作时一种简便的线程间交互方式，而这种交互方式最适合用来取消或停止任务。除了中断以外，还可以利用一个boolean变量来控制是否需要停止任务并终止该线程。

```java
public class CountDown {

    public static void main(String[] args) throws InterruptedException {
        Runner one = new Runner();
        Thread thread = new Thread(one, "CountThread");
        thread.start();
        // 中断1s，main线程对CountThread进行中断，使CountThread能够感知中断而结束
        TimeUnit.SECONDS.sleep(1);
        // 中断
        thread.interrupt();
        Runner two = new Runner();
        thread = new Thread(two, "CountThread");
        thread.start();
        TimeUnit.SECONDS.sleep(1);
        // boolean变量控制结束
        two.cancel();
    }


}

class Runner implements Runnable {
    private long i;
    private volatile boolean on = true;
    @Override
    public void run() {
        while (on && !Thread.currentThread().isInterrupted()) {
            i ++;
        }
        System.out.println("Count i = " + i);
    }
    public void cancel() {
        on = false;
    }
}
```



#### 线程间通信(*)

#### volatile和synchronized关键字

关键字volatile可以用来修饰字段(成员变量)，就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回主内存，保证所有线程对变量访问的可见性。

关键字synchronized可以修饰方法或以同步块的形式来进行使用，它主要确保多个线程在同一时刻，只能有一个线程处于方法或同步块中，保证了线程对变量访问的可见性和排他性。任意线程对Object(Object由synchronized保护)的访问，首先要获得Object的监视器。如果获取失败，线程进入同步队列(SynchronizedQueue)，线程状态变为BLOCKED。当访问Object的前驱线程释放了锁，则该释放操作唤醒阻塞在同步队列中的线程，使其重新尝试对监视器的获取。

#### 等待/通知机制

- wait()方法会释放对象的锁，sleep()方法不释放对象的锁；

等待/通知的相关方法是任意Java对象都具备的，因为这些方法被定义在所有对象的超类 java.lang.Object上：

**等待/通知的相关方法，获取了对象的锁之后才可以调用这些方法：**

| 方法名称        | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| notify()        | 通知一个在对象上访问的线程，使其从wait()方法返回，而返回的前提是该线程获取到了对象的锁 |
| notifyAll()     | 通知所有等待在该对象上的线程                                 |
| wait()          | 调用该方法的线程进入WAITING状态，只有等待另外的线程通知或被中断才会返回，需要注意，调用wait()方法后会释放对象的锁 |
| wait(long)      | 超时等待一段时间，这里的参数时间是毫秒，即等待长达n毫秒，如果没有通知就超时返回 |
| wait(long, int) | 对于超时时间更细粒度的控制，可以达到纳秒                     |

```java
public class WaitNotify {

    static boolean flag = true;
    static Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread waitThread = new Thread(new Wait(), "WaitThread");
        waitThread.start();
        TimeUnit.SECONDS.sleep(1);
        Thread notifyThread = new Thread(new Notify(), "NotifyThread");
        notifyThread.start();
    }

    static class Wait implements Runnable {
        @Override
        public void run() {
            // 加锁，拥有lock的Monitor
            synchronized (lock) {
                // 当条件不满足时，继续wait，同时释放了lock的锁
                while (flag){
                    try {
                        System.out.println(Thread.currentThread() + "flag is true. wait @ " +
                                new SimpleDateFormat("HH:mm:ss").format(new Date()));
                        lock.wait();
                    }catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
            // 条件满足时，完成工作
            System.out.println(Thread.currentThread() + "flag is false. running @ " +
                    new SimpleDateFormat("HH:mm:ss").format(new Date()));
        }
    }

    static class Notify implements Runnable {
        @Override
        public void run() {
            // 加锁，拥有lock的Monitor
            synchronized (lock) {
                // 获取lock的锁，然后进行通知，通知时不会释放lock的锁
                // 直到当前线程释放了lock后，WaitThread才能从wait方法中返回
                System.out.println(Thread.currentThread() + "hold lock. notify @ " +
                        new SimpleDateFormat("HH:mm:ss").format(new Date()));
                lock.notify();
                flag = false;
                SleepUtils.second(5);
            }
            // 再次加锁
            synchronized (lock) {
                System.out.println(Thread.currentThread() + "hold lock again. sleep @ " +
                        new SimpleDateFormat("HH:mm:ss").format(new Date()));
                SleepUtils.second(5);
            }
        }
    }

}
```

**调用wait()、notify()、notifyAll() 时需要注意**：

- 使用wait()、notify()、notifyAll() 时需要先调用对象加锁；
- 调用 wait() 方法后，线程状态由RUNNABLE变为WAITING，并将当前线程放置到对象的等待队列(WaitQueue)，同时释放了对象的锁;
- notify()或notifyAll() 方法调用后，等待线程依旧不会从wait() 返回，需要调用notify()或notifyAll() 的线程释放锁之后，等待线程才有机会从wait() 中返回；
- notify() 方法将等待队列中的一个等待线程从等待队列中移到同步队列(SynchronizedQueue)中，而notifyAll() 方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由WAITING变为BLOCKED。
- 从wait() 返回的前提是获得了调用对象的锁；

**Thread.join() 的使用**：如果一个线程A执行了thread.join() 语句，其含义是：当前线程A等待thread线程终止之后才从thread.join() 返回。

**ThreadLocal 的使用**：ThreadLocal即线程变量，是一个以ThreadLocal对象为键、任意对象为值的存储结构，这个结构被附带在线程上，也就是说一个线程可以根据一个ThreadLocal对象查询到绑定在这个线程上的一个值。可以通过set(T)方法来设置一个值，在当前线程下再通过get()方法获取到原先设置的值。



### Java中的锁

#### Lock接口

Lock接口所提供的synchronized关键字所不具备的特性：

| 特性               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| 尝试非阻塞地获取锁 | 当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁 |
| 能被中断地获取锁   | 与synchronized不同，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会抛出，同时锁会被释放 |
| 超时获取锁         | 在指定的截止时间之前获取锁，如果截止时间到了仍旧无法获取锁，则返回 |

```shell
# 面试必问之 synchronized与Lock的区别及底层实现
	1. Synchronized 是Java的一个关键字，而Lock是java.util.concurrent.Locks包下的一个接口
	2. Synchronized 使用过后，会自动释放锁，而Lock需要手动上锁、手动释放锁(在finally块中)
	3. Lock提供了更多的实现方法，而且可响应中断，而synchronized关键字不能响应中断
		响应中断：
	 A、B 线程同时想获取到锁，A获取锁以后，B会进行等待，这时候等待着锁的线程B，会被Tread.interrupt()方法给中断等待状态、然后去执行其他的事情，
	 而synchronized锁无法被Tread.interrupt()方法给中断掉；可中断锁：在利用thread.interrupted监测到中断请求后，会抛出interruptedexception异常，进而中断线程的执行。非可中断锁：当客户端调用interrupt方法时，只是简单的去设置interrupted中断状态，并没有进一步抛出异常。
     4. synchronized 关键字是非公平锁，即，不能保证等待锁的那些线程的顺序；而Lock的子类ReentrantLock可通过实例化一个布尔参数的构造方法，来实现公平锁和非公平锁，默认为非公平锁。
     5. synchronized是一个悲观锁，Lock是一个乐观锁(底层基于volatile关键字和CAS算法实现)
```



#### 队列同步器(AbstractQueuedSynchronizer)(AQS)

重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态：

- getState()：获取当前同步状态；
- setState()：设置当前同步状态；
- compareAndSetState(int expect, int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性。

**队列同步器的实现分析**

**同步队列**

同步器依赖内部的同步队列(一个FIFO双向队列)来完成同步状态的管理，当线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点(Node)并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

同步队列中的节点(Node)用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点。同步器提供了基于CAS的设置尾节点的方法：compareAndSetTail(Node expect, Node update)；同步队列遵循FIFO，首节点是获取同步状态成功的节点在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点；

**独占式同步状态获取与释放**

通过调用同步器的acquire(int arg)方法可以获取同步状态，该方法对中断不敏感，也就是由于线程获取同步状态失败后进入同步队列中，后序对线程进行中断操作时，线程不会从同步队列中移出。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
        selfInterrupt();
    }
}
```

上述代码主要完成了同步状态获取、节点构造、加入同步队列以及在同步队列中自旋等待的相关工作，其主要逻辑是：首先调用**自定义同步器**实现的tryAcquire(int arg)方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点(独占式节点Node.EXCLUSIVE)并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部(compareAndSetTail(Node expect, Node update))，最后调用acquireQueued(Node node, int arg)方法，使得该节点以“死循环”的方式获取同步状态。如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。节点进入同步队列之后，就进入了一个自旋的过程，每个节点(或者说每个线程)都在自省地观察，当条件满足获取了同步状态，就可以从这个自旋过程退出。

调用同步器的release(int arg)方法释放同步状态，释放同步状态后会唤醒后继节点。

![image-20210201111224241](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210201111224241.png)

**总结**：在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列的队尾并在队列中自旋(for死循环)；移出队列(或停止自旋)的条件是前驱节点为头结点且成功获取了同步状态。在释放同步状态时，同步器调用tryReleases(int arg)方法释放同步状态然后唤醒头结点的后继节点。

**共享式同步状态获取与释放**

通过调用同步器的acquireShared(int arg)方法可以共享地获取同步状态：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0) {
        doAcquireShared(arg);
    }
}
```

使用tryAcquireShared(arg)方法获取同步状态，tryAcquireShared(int arg) 方法返回int类型，当返回值大于等于0时，表示能够获取到同步状态。在doAcquireShared(arg) 方法的自旋过程中，当前节点的前驱节点为头节点时，尝试获取同步状态，如果返回值大于等于0，表示该次获取同步状态成功并从自旋过程中退出。

调用releaseShared(int arg)方法可以释放同步状态，必须确保同步状态线程安全释放，一般通过循环和CAS来保证。

**独占式超时获取同步状态**

通过调用同步器的`doAcquireNanos(int arg, long nanosTimeOut)`方法可以超时获取同步状态，即在指定的时间段内获取同步状态。

`acquireInterruptibly(int arg)`方法：这个方法在等待获取同步状态时，如果当前线程被中断会立刻返回并抛出InterruptedException；

独占式超时获取同步状态流程：当前节点的前驱节点为头结点时尝试获取同步状态，如果获取成功则从该方法返回，若当前线程获取同步状态失败，则判断是否超时(nanosTimeout小于等于0表示已经超时)，若没有超时，重新计算超时时间间隔nanosTimeout，然后使当前线程等待nanosTimeou纳秒；

![image-20210201114847239](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210201114847239.png)

#### 重入锁(ReentrantLock)

重入锁ReentrantLock，支持重进入的锁，表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁还支持获取锁时的公平和非公平性选择；

重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实现需要解决以下两个问题：

- 线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取。
- 锁的最终释放。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取次数进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已成功释放。

锁的公平性问题：如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之是不公平的。公平的获取锁，也就是等待时间最长的线程最优先获取锁。ReentrantLock实现公平锁时，加判断 hasQueuedPredecessors()，判断节点是否有前驱节点来判断是否是首节点(获取同步状态的节点)的下一个节点；

#### 读写锁

```shell
# 参考博客：https://www.cnblogs.com/fsmly/p/10721433.html
```

读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁一个写锁，通过分离读锁和写锁，使得并发性比一般的排他锁有了很大提升。

```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     */
    Lock writeLock();
}
```

读写状态就是同步器的同步状态，读写锁的自定义同步器需要在同步状态(一个整型变量)上维护多个读线程和一个写线程的状态；将变量切分成两部分，高16位表示读，低16位表示写。通过位运算可以确定读写状态，状态值S不为0，当写状态等于0时，则读状态大于0，即读锁已被获取。

**写锁的获取与释放**

写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取或者该线程不是已经获取写锁的线程，则当前线程进入等待状态。只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。

**读锁的获取与释放**：读锁是一个支持重进入的共享锁，能被多个线程同时获取。

**锁降级**：锁降级是指把持住(当前拥有的)写锁，再获取到读锁，随后释放先前拥有的写锁的过程。

**LockSupport工具**：LockSupport定义了一组以park开头的方法用来阻塞当前线程，以及unpark(Thread thread)方法来唤醒一个被阻塞的线程。



#### Condition接口

Condition接口提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式。

Condition对象是由Lock对象创建出来的，`Condition condition = lock.newCondition();`

Condition.await() 当前线程释放锁并在此等待，其他线程调用Condition.signal()方法通知当前线程后当前线程从await()方法返回，并且在返回前已经获取了对象的锁。

ConditionObject 是同步器AQS的内部类，每个Condition对象都包含着一个队列，该队列时Condition对象实现等待/通知机制的关键。

**等待队列**

等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，若一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。同步队列和等待队列中节点类型都是同步器的静态内部类`AbstractQueuedSynchronized.Node`。并发包中的Lock(更准确地说是同步器)拥有**一个同步队列和多个等待队列**。

**等待**

调用Condition的await() 方法，会使当前线程进入等待队列并释放锁，同时线程状态变为等待状态。当从await() 方法返回时，当前线程一定获取了Condition相关联的锁。从同步队列和等待队列的角度看await() ，调用await() 方法时，相当于同步队列的首节点移动到Condition的等待队列中；

**通知**

调用Condition的signal() 方法，将会唤醒在等待队列中等待时间最长的节点(首节点)，在唤醒节点之前，会将节点移动到同步队列中。

通过调用同步器的enq(Node node)，等待队列中的头节点线程安全地移动到同步队列，当节点移动到同步队列后，当前线程再使用LockSupport唤醒该节点的线程。

Condition的signalAll() 方法，相当于对等待队列中的每个节点均执行一次signal() 方法，效果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。



### Java并发容器和框架

#### ConcurrentHashMap

详见**复习篇-容器**



#### ConcurrentLinkedQueue

ConcurrentLinkedQueue是一个基于链接节点的无界线程安全队列，采用先进先出的规则对节点进行排序，添加一个元素时，添加到队列的尾部，获取一个元素时，返回队列头部的元素。采用"wait - free"算法(即CAS算法)来实现；

ConcurrentLinkedQueue由head节点和tail节点组成，每个节点(Node)由节点元素(item)和指向下一个节点(next)的引用组成，节点与节点之间就是通过next关联起来的，从而组成一张链表结构的队列。默认情况下head节点存储的元素为空，tail节点等于head节点。

```java
private transient volatile Node<E> head;
private transient volatile Node<E> tail;
```

**入队列**

节点添加到队列的尾部；尾节点可能是tail节点也可能是tail节点的next节点；入队操作：定位出尾节点；使用CAS算法将入队节点设置为尾节点的next节点，如不成功则重试；

**出队列**

首先获取头结点的元素，然后判断头节点是否为空，如果为空，表示另外一个线程已经进行了一次出队操作将该节点的元素取走，如果不为空，则使用CAS的方式将头节点的引用设置为null，如果CAS成功返回头节点元素；否则表示另一个线程已经进行了一次出队操作更新了head节点，导致元素发生了变化，需要重新获取头节点。



#### Java中的阻塞队列(Blocking Queue)

阻塞队列是一个支持两个附加操作的队列：

- 支持阻塞的插入方法：当队列满时，队列会阻塞插入元素的线程，直到队列不满；
- 支持阻塞的移除方法：在队列为空时，获取元素的线程会等待队列变为非空；

| 类名                  | 功能                                                         |
| --------------------- | ------------------------------------------------------------ |
| ArrayBlockingQueue    | 数组实现的有界阻塞队列，默认情况下不保证线程公平的访问队列   |
| LinkedBlockingQueue   | 链表结构组成的有界阻塞队列                                   |
| PriorityBlockingQueue | 一个支持优先级排序的无界阻塞队列                             |
| DelayQueue            | 支持延时获取元素的无界阻塞队列，只有在延时期满之后才可以从队列中提取元素 |
| SynchronousQueue      | 不存储元素的队列，每个put操作必须等待一个take操作否则不能继续添加元素 |
| LinkedTransferQueue   | 链表组成的无界阻塞队列                                       |
| LinkedBlockingDeque   | 链表结构组成的双向队列                                       |



#### Fork/Join框架

Fork/Join框架是Java7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务最终汇总每个小任务结果后得到大任务结果的框架。

工作窃取(work - stealing)算法：某个线程从其他队列里窃取任务来执行；

Fork/Join框架使用两个类完成分割任务和执行任务合并结果这两件事情：

- ForkJoinTask：执行fork() 和 join() 操作；提供了两个子类：RecursiveAction：用于没有返回结果的任务；RecursiveTask：用于有返回结果的任务；
- ForkJoinPool：ForkJoinTask需要通过ForkJoinPool来执行；

**实现原理**：

ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成，ForkJoinTask数组负责将存放程序提交给ForkJoinPool的任务；ForkJoinWorkerThread数组负责执行这些任务。

1. fork() 方法：调用ForkJoinTask的fork() 方法时，程序会调用ForkJoinWorkerThread的pushTask方法异步地执行任务；pushTask方法把当前任务存放在ForkJoinTask数组中，调用ForkJoinPool的signalWork() 方法唤醒或创建一个工作线程来执行任务；
2. join() 方法：阻塞当前线程并等待获取结果；



### Java中的13个原子操作类

java.util.concurrent.atomic包，该包中的类几乎都是使用**Unsafe实现的包装类**；

#### 原子更新基本数据类型

使用原子的方式更新基本类型，Atomic包提供了以下3个类：AtomicBoolean、AtomicInteger、AtomicLong；

```java
//  AtomicInteger.java
public final int getAndIncrement() {
    for (;;) {
        int current = get();	// 取得存储的值
        int next = current + 1;	// 加1操作
        if (CompareAndSet(current, next)) {	// 更新原子操作
            return current;
        }
    }
}

public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

```java
//  Unsafe.java
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```



#### 原子更新数组

通过原子的方式更新数组里的某个元素，Atomic包提供了以下4个类：

- AtomicIntegerArray: 原子更新整型数组里的元素，int addAndGet(int i, int delta)、boolean compareAndSet(int i, int expect, int update)；
- AtomicLongArray: 原子更新长整形数组里的元素；
- AtomicReferenceArray: 原子更新引用类型数组里的元素；



#### 原子更新引用类型

原子更新基本类型的AtomicInteger，只能更新一个变量。如果要原子更新多个变量，就要使用原子更新引用类型提供的类：

- AtomicReference: 原子更新引用类型；
- AtomicReferenceFieldUpdater：原子更新引用类型里的字段；
- AtomicMarkableReference：原子更新带有标记位的引用类型；

```java
public static AtomicReference<User> atomicUserRef = new AtomicReference<User>();
public static void main(String[] args) {
    User user = new User();
    atomicUserRef.set(user);	// user对象设置进AtomicReference中
    User updateUser = new User();
    atomicUserRef.compareAndSet(user, updateUser);	// 调用 compareAndSet() 方法进行原子更新操作
}
```



#### 原子更新字段类

若需要原子地更新类里的某个字段，就需要使用原子更新字段类：

- AtomicIntegerFieldUpdater: 原子更新整型的字段的更新器；
- AtomicLongFieldUpdater: 原子更新长整型字段的更新器；
- AtomicStampedReference: 原子更新带有版本号的引用类型。可以解决使用CAS进行原子操作可能出现的ABA问题；

想要原子地更新字段类需要两步：

- 第一步，因为原子更新字段类都是抽象类，每次使用的时候必须使用静态方法newUpdater() 创建一个更新器，并且设置想要的类和属性；
- 第二步：更新的字段(属性)必须使用 public volatile 修饰符；



### Java中的并发工具类

#### 等待多线程完成的CountDownLatch

`CountDownLatch`：任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后`countDown()`一次，state会`CAS`(Compare and Swap)减1。等到所有子线程都执行完后(即state=0)，会`unpark()`主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

本质：使用了AQS框架实现等待多线程完成，共享式资源获取方式，传入的N即设置的volatile状态State；

CountDownLatch允许一个或多个线程等待其他线程完成操作；

CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果想等待N个点完成，这里就传入N；

调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成零；由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是N个步骤。用在多线程时，只需要把CountDownLatch的引用传递到线程里即可；

```java
//  CountDownLatch.java
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
}
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

// AbstractQueuedSynchronizer.java(AQS)
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED); // 共享
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
}
```



#### 同步屏障CyclicBarrier

CyclicBarrier的字面意思是可循环使用(Cyclic)的屏障(Barrier)。让一组线程到达一个屏障(也可以叫同步点)时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。CyclicBarrier默认的构造方法是 `CyclicBarrier(int parties)`，其参数表示屏障拦截的线程数量，每个线程调用await方法通知CyclicBarrier到达屏障；

CyclicBarrier还提供了一个更高级的构造函数：CyclicBarrier(int  parties, Runnable barrierAction)用于在线程到达屏障时，优先执行barrierAction。

**CountDownLatch 和 CyclicBarrier的区别**

- CountDownLatch 的计数器只能使用一次，而 CyclicBarrier的计数器可以使用reset() 方法重置；



#### 控制并发线程数的Semaphore

- 源代码中使用静态内部类sync继承AbstractQueuedSynchronizer，实现acquire() 和 release() 方法；

Semaphore(信号量)是用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用公共资源。

线程使用Semaphore的acquire() 方法获取一个许可证，使用完之后调用release() 方法归还许可证；



#### 线程间交换数据的Exchanger

Exchanger(交换者)是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交换，提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。



### Java中的线程池

在开发过程中，合理使用线程池能够带来以下好处：

第一：**降低资源消耗**。通过反复利用已创建的线程降低线程创建和销毁造成的消耗。

第二：**提高响应速度**。当任务达到时，任务可以不需要等到线程创建就能立即执行。

第三：**提高线程的课管理性**。使用线程池可以统一分配、调优和监控。

提交一个新任务到线程池时，线程池的处理流程：

![image-20210204095842593](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210204095842593.png)

ThreadPoolExecutor执行execute() 方法，分以下四种情况：

1）如果当前运行的线程少于corePoolSize，则创建新线程来执行任务(注意：执行这一任务需要获取全局锁)

2）如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue；

3）如果无法将任务加入BlockingQueue(队列已满)，则创建新的线程来执行任务(注意：执行这一任务需要获取全局锁)

4）如果创建新的线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution() 方法。

在ThreadPoolExecutor完成预热之后，当前运行的线程数大于等于corePoolSize，几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。

**工作线程**：线程池创建线程时，会将线程封装成工作线程Worker，Worker在执行完任务后还会循环获取工作队列里的任务来执行。

线程池中的线程执行任务分两种情况：

- 在execute() 方法中创建一个线程时，会让这个线程执行当前任务；
- 该线程反复从BlockingQueue获取任务执行；

**1. 线程池的创建**

```java
//   通过ThreadPoolExecutor来创建一个线程池
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

- corePoolSize(线程池的基本大小)：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池的基本大小时就不再创建。
- workQueue(任务队列)：用于保存等待执行的任务的阻塞队列，可以选择：

| 类名                  | 功能                                                         |
| --------------------- | ------------------------------------------------------------ |
| ArrayBlockingQueue    | 数组实现的有界阻塞队列，默认情况下不保证线程公平的访问队列   |
| LinkedBlockingQueue   | 链表结构组成的有界阻塞队列                                   |
| PriorityBlockingQueue | 一个支持优先级排序的无界阻塞队列                             |
| SynchronousQueue      | 不存储元素的队列，每个put操作必须等待一个take操作否则不能继续添加元素 |

- maximumPoolSize(线程池最大数量)：线程池允许创建的最大线程数；若使用无界的阻塞队列这个参数就没什么效果；
- threadFactory：用于设置创建线程的工厂；
- RejectedExecutionHandler(饱和策略)：默认AbortPolicy
  - AbortPolicy：直接抛出异常
  - CallerRunsPolicy：只用调用者所在线程来执行任务
  - DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务
  - DiscardPolicy：不处理，丢弃

**2. 向线程池提交任务**

可以使用两个方法向线程池提交任务：

- execute() 方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功；
- submit() 方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，get() 方法会阻塞当前线程直到任务完成。

**3. 关闭线程池**

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池，原理：遍历线程池中的工作线程，逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。



### Executor框架

#### Executor框架的两级调度模型

在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户级的调度器(Executor框架)将这些任务映射为固定数量的线程；在底层，操作系统内核将这些线程映射到硬件处理器上。

Executor框架主要由3大部分组成：

- 任务。包括被执行任务需要实现的接口：Runnable接口和Callable接口；
- 任务的执行。包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口(ThreadPoolExecutor和ScheduledThreadPoolExecutor)
- 异步计算的结果。包括接口Future和实现Future接口的FutureTask类。

Executor框架的使用过程：

主线程首先要创建实现Runnable或者Callable接口的任务对象。工具类Executors可以把一个Runnable对象封装为一个Callable对象(Executors.callable(Runnable Task) 或 Executors.callable(Runnable task, Object result))。然后可以把Runnable对象直接交给ExecutorService执行(ExecutorService.execute(Runnable command))；或者也可以把Runnable对象或Callable对象提交给ExecutorService执行(ExecutorService.submit(Runnable task)或ExecutorService.submit(Callable<T>  task))。如果执行ExecutorService.submit()将返回一个实现Future接口的对象(FutureTask 对象)。最后主线程可以执行FutureTask.cancel()来取消任务的执行。

![image-20210204171126014](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210204171126014.png)

#### ThreadPoolExecutor详解

Executor框架最核心的类是ThreadPoolExecutor，它是线程池的实现类，由核心线程池大小、阻塞队列、最大线程池的大小以及饱和策略构成。通过Executor框架的**工具类Executors**可以创建3种类型的ThreadPoolExecutor：

**1. FixedThreadPool详解**

FixedThreadPool被称为可重用固定线程数的线程池。源代码：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```

corePoolSize = maximumPoolSize = nThreads

keepAliveTime设置为0L，意味着多余的空闲线程会被立即终止。



**2. SingleThreadExecutor详解**

SingleThreadPoolExecutor是使用单个worker线程的Executor。

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```

corePoolSize = maximumPoolSize = 1



**3. CachedThreadPool详解**

CachedThreadPool是一个会根据需要创建新线程的线程池。

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```

keepAliveTime设置为60L，意味着CachedThreadPool中的空闲线程等待新任务的最长时间为60秒，空闲线程超过60秒后将会被终止。

CachedThreadPool使用SynchronousQueue把主线程提交的任务传递给空闲线程执行。



#### ScheduledThreadPoolExecutor详解

ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，主要用来在给定的延迟之后运行任务，或者定期执行任务。

ScheduledThreadPoolExecutor的执行主要分为两大部分：

1）当调用ScheduledThreadPoolExecutor的scheduleAtFixedRate()方法或者scheduleWithFixedDelay() 方法时，会向ScheduledThreadPoolExecutor的DelayQueue添加一个实现了RunnableScheduledFuture接口的ScheduledFutureTask。

2）线程池中的线程从DelayQueue中获取ScheduledFutureTask，然后执行任务。

ScheduledFutureTask 主要包含3个成员变量：

- long型成员变量time，表示这个任务将要被执行的具体时间。
- long型成员变量sequenceNumber，表示这个任务被添加到ScheduledThreadPoolExecutor中的序号。
- long型成员变量period，表示任务执行的间隔周期。

DelayQueue封装了一个PriorityQueue，排序时先排序time后排序sequenceNumber，值越小排序越前。

ScheduledThreadPoolExecutor中的线程执行**周期任务**的步骤：

1）从DelayQueue中获取已到期的ScheduledFutureTask(DelayQueue.take())。到期任务是指ScheduledFutureTask的time大于等于当前时间；

2）线程执行这个ScheduledFutureTask；

3）线程修改ScheduledFutureTask的time变量为下次要被执行的时间；

4）线程把修改time之后的ScheduledFutureTask放回到DelayQueue中(DelayQueue.add())。



#### FutureTask详解

Future接口和实现Future接口的FutureTask类，代表异步计算的结果。

FutureTask除了实现Future接口外，还实现了Runnable接口。

```java
/**
 * A {@code Future} represents the result of an asynchronous
 * computation.  Methods are provided to check if the computation is
 * complete, to wait for its completion, and to retrieve the result of
 * the computation.*/
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

FutureTask的 ge t和 cancel 的执行示意图：

![image-20210204180931168](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210204180931168.png)

在FutureTask中定义了volatile修饰的状态变量state来进行Executor中的工作线程和应用主线程之间的交互，即工作线程产生任务执行结果，通知应用主线程获取；应用主线程请求取消任务执行，通知工作线程停止该任务执行。在内部实现中通过将state与以下状态常量进行大小比较来获取任务执行情况，如是正常执行成功还是异常退出，被取消等。`awaitDone()`方法中使用了UNsafe类的CAS方法，compareAndSwapObject（）；

```java
/*Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED*/
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
}
```

Executor的工作线程执行该任务时，会调用该任务的run方法，即FutureTask的run方法，如下为FutureTask的run方法定义：首先检查任务状态state是否为NEW，是，即还没执行过也没有被取消等，则进行往下执行。执行完成之后，产生执行结果result，调用set方法来处理这个结果。

```java
public void run() {
    // 赋值runner，指向线程池中当前执行这个task的线程
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 应用代码的task的run方法执行
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            // 执行成功设置结果result，并通知get方法
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```















## 队列同步器AbstractQueuedSynchronizer (AQS)

```shell
# 参考博客：
https://www.cnblogs.com/waterystone/p/4920797.html
```

Java中的大部分同步类(Lock、Semaphore、ReentrantLock等)都是基于`AbstractQueuedSynchronizer`(简称为AQS)(直译：抽象队列同步器)实现的。AQS是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。

AQS的核心思想是：如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并将共享资源设置为锁定状态；如果请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是使用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。

**`CLH`队列是一个虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在节点之间的关联关系)。AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个节点(Node)来实现锁的分配。**

![image-20210128214534986](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210128214534986.png)

AQS维护了一个`volatile int state`和一个FIFO线程等待队列，多线程争用资源被阻塞的时候就会进入这个队列。AQS类使用单个int(32位)来保存同步状态，并暴露出`getState`、`setState`以及`compareAndSet`操作来读取和更新同步状态。其中属性state被声明为volatile，并且通过使用CAS指令来实现`compareAndSet`，使得当且仅当同步状态拥有一个一致性期望值的出后，才会被原子地设置成新值，这样就达到了同步状态的原子性管理，确保了同步状态的原子性、可见性和有序性。

`AQS`定义了两种资源共享方式：

- `Exclusive`：独占，只有一个线程能执行，如`ReentrantLock`
- `Share`：共享，多个线程可以同时执行，如`Semaphore、CountDownLatch、ReadWrtieLock、CyclicBarrier`

```java
//  AQS 源代码，使用了大量的CAS方法
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
    //节点：包装了线程，双向链表的前后引用，节点的几种状态和当前节点的状态。
    static final class Node {
        //共享式
        static final Node SHARED = new Node();
        //独占式
        static final Node EXCLUSIVE = null;
        
		//表示获取锁的请求已被取消
        static final int CANCELLED =  1;
        
   //表示当前节点的后继节点的线程需要被唤醒
   //当前节点的后继节点已经 (或即将)被阻塞（通过park）,所以当当前节点释放或则被取消时，一定要unpark它的后继节点。为了避免竞争，获取方法一定要首先设置node为signal，然后再次重新调用获取方法，如果失败，则阻塞。
        static final int SIGNAL    = -1;

        //表明该线程被处于条件队列，就是因为调用了>- Condition.await而被阻塞
        static final int CONDITION = -2;

        //共享模式下的释放操作应该被传播到其他节点。
        static final int PROPAGATE = -3;

        //当前Node所代表的线程的等待锁的状态
        volatile int waitStatus;

        //前驱和后继
        volatile Node prev;
        volatile Node next;
	
        // 节点所代表的线程
        volatile Thread thread;

        //用于条件队列和共享锁
        Node nextWaiter;
		// 返回前一节点
        final Node predecessor() throws NullPointerException {
            ...
        }
		// Node 的构造方法
        ...
    }
    // 等待队列的头尾节点
    private transient volatile Node head;
    private transient volatile Node tail;
    // 同步状态
    private volatile int state;
    
    protected final int getState() {
        return state;
    }

    protected final void setState(int newState) {
        state = newState;
    }
    //自动将共享资源更新为给定值，CAS
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    // 独占模式下线程获取共享资源的顶层入口
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    //将当前线程加入到队列的队尾
    private Node addWaiter(Node mode) {
        ...
    }
    //入队 自旋
    private Node enq(final Node node) {
        ...
    }
    //把已经追加到队列的线程节点（addWaiter方法返回值）进行阻塞
    final boolean acquireQueued(final Node node, int arg) {
        ...
    }
    //当获取锁失败后，进入park状态
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        ...
    }
    private final boolean parkAndCheckInterrupt() {
        ...
    }
    // 释放锁
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;//找到头结点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);//唤醒等待队列里的下一个线程
            return true;
        }
        return false;
	}
    // 唤醒等待队列中下一个线程
    private void unparkSuccessor(Node node) {
        ...
    }
    // 共享模式下线程获取共享资源的顶层入口
    public final void acquireShared(int arg) {
       if (tryAcquireShared(arg) < 0)
             doAcquireShared(arg);
     }
    // 当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己
    private void doAcquireShared(int arg) {
        ...
    }
    
    //  Unsafe 中的CAS()方法
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;


    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }

    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }

    private static final boolean compareAndSetWaitStatus(Node node,
                                                         int expect,
                                                         int update) {
        return unsafe.compareAndSwapInt(node, waitStatusOffset,
                                        expect, update);
    }


    private static final boolean compareAndSetNext(Node node,
                                                   Node expect,
                                                   Node update) {
        return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
    }
    
}
```



**Acquire(int arg)** 是独占模式下线程获取共享资源的顶层入口：

1. 调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；
2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断`selfInterrupt()`，将中断补上。

![image-20210128215037934](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210128215037934.png)

| 属性、方法                        | 作用                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| boolean tryAcquire(int arg)       | 独占式。尝试获取资源。成功返回true，失败返回false            |
| boolean tryRelease(int arg)       | 独占式。尝试获取资源。成功返回true，失败返回false            |
| int tryAcquireShared(int arg)     | 共享式。尝试获取资源。负数表示失败，0表示成功，但没有剩余可用资源，正数表示成功且有剩余可用资源。 |
| boolean tryReleaseShared(int arg) | 共享式。尝试释放资源，如果释放后允许唤醒后续等待线程，则返回true |
| void acquire(int arg)             | 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则会进入同步队列等待。 |
| void acquireShared(int arg)       | 共享式的获取同步状态，如果当前系统未获取到同步状态，将会进入同步队列等待 |
| boolean releaseShared(int arg)    | 共享式的释放同步状态                                         |

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源 state 的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。

同步器的设计是基于**模板方法**模式的，如果需要自定义同步器一般的方式是这样（模板方法模式很经典的一个应用）：

1. 使用者继承`AbstractQueuedSynchronizer`并重写指定的方法。（这些重写方法很简单，无非是对于共享资源state的获取和释放）
2. 将`AQS`组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。

**`ReentrantLock`：**state初始化为0，表示未锁定状态。A线程lock()时，会调用`tryAcquire()`独占该锁并将state+1。此后，其他线程再`tryAcquire()`时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

![img](https://p0.meituan.net/travelcube/f30c631c8ebbf820d3e8fcb6eee3c0ef18748.png)

**`ReentrantReadWriteLock`：**使用`AQS`同步状态中的16位来保存写锁持有的次数，剩下的16位用来保存读锁的持有次数。`WriteLock`的构建方式同`ReentrantLock`。`ReadLock`则通过使用`acquireShared`方法来支持同时允许多个读线程。

**`Semaphore`：**使用`AQS`同步状态来保存信号量的当前计数。它里面定义的`acquireShared`方法会减少计数，或当计数为非正值时阻塞线程；`tryRelease`方法会增加计数，在计数为正值时还要解除线程的阻塞。

`CountDownLatch`：任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后`countDown()`一次，state会`CAS`(Compare and Swap)减1。等到所有子线程都执行完后(即state=0)，会`unpark()`主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

**`FutureTask`：**使用`AQS`同步状态来表示某个异步计算任务的运行状态（初始化、运行中、被取消和完成）。设置（`FutureTask`的set方法）或取消（`FutureTask`的cancel方法）一个`FutureTask`时会调用`AQS`的release操作，等待计算结果的线程的阻塞解除是通过`AQS`的acquire操作实现的。

**`SynchronousQueues`：**使用了内部的等待节点，这些节点可以用于协调生产者和消费者。同时，它使用AQS同步状态来控制当某个消费者消费当前一项时，允许一个生产者继续生产，反之亦然。

### 自定义同步器

不同的自定义同步器争用共享资源的方式也不同。**自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可**，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

Mutex(互斥锁)是一个不可重入的互斥锁实现。锁资源（AQS里的state）只有两种状态：0表示未锁定，1表示锁定。下边是Mutex的核心源码：

```java
class Mutex implements Lock, java.io.Serializable {
    // 自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 判断是否锁定状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 尝试获取资源，立即返回。成功则返回true，否则false。
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // 这里限定只能为1个量
            if (compareAndSetState(0, 1)) {//state为0才设置为1，不可重入！
                setExclusiveOwnerThread(Thread.currentThread());//设置为当前线程独占资源
                return true;
            }
            return false;
        }

        // 尝试释放资源，立即返回。成功则为true，否则false。
        protected boolean tryRelease(int releases) {
            assert releases == 1; // 限定为1个量
            if (getState() == 0)//既然来释放，那肯定就是已占有状态了。只是为了保险，多层判断！
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);//释放资源，放弃占有状态
            return true;
        }
    }

    // 真正同步类的实现都依赖继承于AQS的自定义同步器！
    private final Sync sync = new Sync();

    //lock<-->acquire。两者语义一样：获取资源，即便等待，直到成功才返回。
    public void lock() {
        sync.acquire(1);
    }

    //tryLock<-->tryAcquire。两者语义一样：尝试获取资源，要求立即返回。成功则为true，失败则为false。
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    //unlock<-->release。两者语文一样：释放资源。
    public void unlock() {
        sync.release(1);
    }

    //锁是否占有状态
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }
}
```











