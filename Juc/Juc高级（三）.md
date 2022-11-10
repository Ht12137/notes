# JUC高级（三）

### 线程中断机制

首先，一个线程不应该由其他线程强制中断或者停止，而是应该由线程自己停止，自己来决定自己的命运。

> Tread.stop Thread.suspend,Thread.resume均被废弃了

因为这些方法很不安全，导致死锁的情况也是常有发生的。

其次，虽然在java中无法立即停止一条线程，然而停止线程却尤其重要，比如取消一个耗时的操作。

> 因此，java提供了一种停止线程的协商机制——中断，即中断表示协商机制。

注意：中断只是一种协作协商机制，Java没有给中断增加任何语法，中断的过程完全需要程序员自己实现。

只是别的线程通过某种标识位过来告诉你，你该中断了。至于你接受到这个通知做出什么样的行为，完全取决于你自己。

### 中断相关API——三大方法

- public void interrupt()

  实例方法，仅仅只是设置线程中断状态为true,发起一个协商而不会立即停止线程。

- public static boolean interrupted()

  静态方法，判断线程是否被中断并且清除当前的中断状态，变成false。这个方法返回的是中断状态。
  
- public boolean isInterrupted()

  实例方法，判断当前的线程是否被中断。通常和第一个方法搭配使用。

### 大厂面试题中断机制考点

#### 如何停止中断运行中的线程？

#### 通过volatile变量实现

> volatile是常用的高并发场景下的关键字，先了解

```java
public class InterruptDemo {

    static volatile boolean isStop = false;

    public static void main(String[] args) throws InterruptedException {

        new Thread(()->{
            while (true){
                if(isStop){
                    System.out.println(Thread.currentThread().getName() + "\t isStop 变成true");
                    break;
                }

                System.out.println("-----hello volatile");
            }
        },"t1").start();

        Thread.sleep(2000);
        new Thread(()->{
            isStop = true;
        },"t2").start();
    }

}
```

#### 通过AtomicBoolean实现

主要代码还是不变，只是这个用来提醒线程是否停止的标识符从volatile变成了AtomicBoolean。

```java
    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            while (true){
                if(atomicBoolean.get()){
                    System.out.println(Thread.currentThread().getName() + "\t atomicBoolean 变成true");
                    break;
                }

                System.out.println("-----hello atomicBoolean");
            }
        },"t1").start();

        Thread.sleep(2000);
        new Thread(()->{
            atomicBoolean.set(true);
        },"t2").start();
    }
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108183145953.png)

#### 通过线程类自带的中断api实例方法实现

> 就是我们上面讲的三种

我们刚刚的两种方法volatile和atomicBoolean，其实就是在需要中断的线程中，不断监听中断状态，一旦发生中断，就执行相应的中断处理业务逻辑去stop线程。

```java
public static void main(String[] args) throws InterruptedException {

    Thread t1 = new Thread(() -> {
        while (true) {
            if (Thread.currentThread().isInterrupted()) {
                System.out.println(Thread.currentThread().getName() + "\t is interrupted");
                break;
            }
            System.out.println("-----hello interrupt Api");
        }
    }, "t1");

    t1.start();

    Thread.sleep(2000);
    new Thread(()->{
        t1.interrupt();
    },"t2").start();
    
}
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108184332778.png)

这相当于是t2对t1发起了中断请求，t1接受了中断请求，并且打印了t1 is interrupted。

> 也可以自己把自己中断。
>
> 例如：t1.interrupt();

### 线程中断API源码解读

#### interrupt方法

对于interrupt的方法，它的jdk底层源码是这样的。

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221109184436960.png)

我们发现，它调用了interrupt0()方法，而interrupt0方法是一个native方法，我们知道native方法是C++写的。

它的作用仅仅是设置一个标志位。

> 1. 如果线程处于正常活动状态，那么interrupt方法会将该线程的中断标志设置为true,仅此而已。所以该方法不能真正的中断线程，需要和被调用的线程自己进行配合。
> 2. 如果线程处于被阻塞的状态（比如sleep，wait，join）,在别的线程调用当前现成的interrupt方法，那么线程将立即退出被阻塞的状态，并且抛出InterruptException异常

对于第二种情况，我们来进行一次代码的演示

对于while（true）循环里，每循环一次，让它睡眠一会儿，这样在睡眠的时候，被t2线程唤醒的概率就很高了！

```java

    public static void main(String[] args) {

        Thread t1 = new Thread(() -> {
            while (true){
                System.out.println(Thread.currentThread().getName() + "正在运行中");
                if(Thread.currentThread().isInterrupted()){
                    System.out.println(Thread.currentThread().getName() +"\t 已经被中断了");
                    break;
                }
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printS
                }
            }
        }, "t1");
        t1.start();
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        new Thread(()->{
            t1.interrupt();
        },"t2").start();
    }

```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221109192215113.png)

其实呢，这个interruptException就是sleep这个方法抛出去的。并且程序没有停下来，说明在睡眠中，对线程使用interrupt方法会抛异常。

> 而这个程序不会停下来的原因就是，虽然t2线程将t1线程的中断标志位变成true了，但是由于此时抛出了异常，程序会将中断标志位清除，所以程序会一直这样打印。

**这里是面试高频考点**

对于这个现象，我们可以在catch模块中，再次调用这个interrupt方法来解决。

```java
 try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    e.printStackTrace();
                }
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221109192500640.png)

#### Thread.interrupted()方法

这是Thread类的静态方法。它的作用我们之前提过，是将中断标志位清空，并且返回当前的中断状态。

我们进入代码实例看看。

```java
public class InterruptDemo3 {
    public static void main(String[] args) {
        System.out.println("当前的线程状态\t" + Thread.interrupted());
        System.out.println("当前的线程状态\t" + Thread.interrupted());
        System.out.println("-------1");
        Thread.currentThread().interrupt();
        System.out.println("当前的线程状态\t" + Thread.interrupted());
        System.out.println("当前的线程状态\t" + Thread.interrupted());
    }
}
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221109194625467.png)

那有同学可能会好奇了，Thread.interrupt()和 实例方法的interrupt有啥区别呢？

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221109195133534.png)

实际上，静态的interrupted方法也只不过是多做了一步清空状态的操作。也就是clearInterruptEvent（），这也是个native方法

> 这些知识都是后续AQS的学习的基础！基础不牢地动山摇！

## LockSupport

> 在学习LockSupport之前，我们一定是要进行Object的类的wait()和notify（）的学习，以及可重入锁中的await（）和signal（）的学习

其实上述说的这两对方法，其实就是用的操作系统里面生产者消费者的这样的一个思想。

我们下面会通过代码实战，来演示这两对的代码实现，以及为什么我们需要用到LockSupport来替代他们

### wait()和notify

```java
@SuppressWarnings("ALL")
public class LockSupportDemo {
    public static void main(String[] args) {
        Object objectLock = new Object();

        new Thread(()->{
            synchronized (objectLock){
                System.out.println(Thread.currentThread().getName() + "开始运作,---即将进入等待状态");
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                System.out.println(Thread.currentThread().getName()+"我被唤醒啦");
            }
        },"t1").start();
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        new Thread(()->{
            synchronized (objectLock){
                System.out.println(Thread.currentThread().getName() + "我的任务是唤醒线程");
                objectLock.notify();
            }
        },"t2").start();
    }
}
```

这是对于Object类的wait（）和notify（）方法。在t1线程里，我们对它做了等待操作，此时t1会放开对象锁，然后t2线程拿到对象锁，通知t1线程别等了，让他继续执行。

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221109202622766.png)

然而，这个对于操作系统核心思想的生产者消费者的实现是有优化空间的。因为这些操作必须要在同步代码块中进行，并且notify（）方法一定得在wait（）方法的后面，否则就会出现现成的阻塞。

### await（）和signal()

```java
    public static void main(String[] args) {

        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        new Thread(()->{
                lock.lock();
                try {
                    System.out.println(Thread.currentThread().getName() + "开始运作,---即将进入等待状态");
                    condition.await();
                    System.out.println(Thread.currentThread().getName()+"我被唤醒啦");
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } finally {
                    lock.unlock();
                }
        },"t1").start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        new Thread(()->{
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + "我的任务是唤醒线程");
                condition.signal();
            }finally {
                lock.unlock();
            }
        },"t2").start();

    }
```

同样，它和上面的方法的问题一样。需要手动加锁，解锁。且signal方法必须要放在await()方法后面。

