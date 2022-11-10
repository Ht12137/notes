# Juc高级（二）

## 从乐观锁和悲观锁开讲

如果对这俩不熟悉的同学：https://juejin.cn/post/7158337554075369503

### **悲观锁**

如Lock，Sychronized，的实现类都是悲观锁。他们认为自己在使用数据的时候，一定会有其他线程跟自己争夺资源，因此在获得数据的时候，一定会加锁，确保数据不会被其他线程修改

#### 适用场景

适合写操作比较多的场景，先加锁可以保证写操作时候的数据准确。

> 它会显式锁定之后再操作同步资源

### 乐观锁

认为自己在操作临界资源的时候，不会有其他线程跑过来跟我抢资源，所以不会添加锁。

在Java编程中是通过无锁编程来实现的，只是在更新操作之前会判断，之前是否有其他线程更新了这个数据。

>  如果这个数据被判断后，没有进行更新，那么就进行数据的修改

>  如果这个数据被判断后，进行了更新，那么就放弃修改，重试枪锁等等。

#### 判断规则

1. 版本号机制
2. 最常采用CAS算法

想看实战：https://juejin.cn/post/7158337554075369503

#### 适用场景

适合读操作多的场景，不加锁的特点让读操作的性能大幅提升。

## 通过8种demo，看看锁的是啥？

我们在阿里巴巴开发手册里发现：

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107123528658.png)

也就是说，如果要用锁，粒度一定要小！

那么什么是对象锁，什么是类锁？什么是锁区块？

**通过8种demo，看看锁的是啥。**

#### 第一种

```java
class Phone{
    public synchronized void sendEmail(){
        System.out.println("-----sendEmail");
    }

    public synchronized void sendSMS(){
        System.out.println("-----sendSMS");
    }
}
public class demo8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        new Thread(()-> {
            phone.sendEmail();
            },"a").start();

        Thread.sleep(200);
        
        new Thread(()->{
            phone.sendSMS();
        },"b").start();
    }
}
```

资源类是Phone，手机里有俩方法，发邮件和发短信。

我们在操作资源类的主方法里面，让俩线程之间间隔2s，肯定是先邮件后短信。



#### 第二种

我们让sendEmail方法，沉睡3s，其他不变。先打印邮件还是短信？

```java
    public synchronized void sendEmail() throws InterruptedException {
        Thread.sleep(3000);
        System.out.println("-----sendEmail");
    }
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107125354845.png)

#### 第三种

添加一个普通的hello方法（不加锁）。b线程执行hello（）。先打印邮件还是hello？

```java
   public void hello(){
        System.out.println("-----hello");
    }
```

```java
        new Thread(()->{
            phone.hello();
        },"b").start();
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107125538287.png)

#### 第四种

两部手机，先打印邮件，还是短信？

```java
   public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        Phone phone2 = new Phone();
        new Thread(()-> {
            try {
                phone.sendEmail();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"a").start();

        Thread.sleep(2000);

        new Thread(()->{
            phone2.sendSMS();
        },"b").start();
    }
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107125837150.png)

#### 第五种

俩静态同步方法，一部手机。先打印邮件，还是短信？

```java
class Phone{
    public static synchronized void sendEmail() throws InterruptedException {
        Thread.sleep(3000);
        System.out.println("-----sendEmail");
    }

    public static synchronized void sendSMS(){
        System.out.println("-----sendSMS");
    }

    public void hello(){
        System.out.println("-----hello");
    }
}


public class demo8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
//        Phone phone2 = new Phone();
        new Thread(()-> {
            try {
                phone.sendEmail();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"a").start();

        Thread.sleep(2000);

        new Thread(()->{
            phone.sendSMS();
        },"b").start();
    }
}
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107130341775.png)

#### 第六种

俩静态同步方法，两部手机。先打印邮件，还是短信？

```java
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        Phone phone2 = new Phone();
        new Thread(()-> {
            try {
                phone.sendEmail();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"a").start();

        Thread.sleep(2000);

        new Thread(()->{
            phone2.sendSMS();
        },"b").start();
    }
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107130503944.png)

#### 第7种

一部手机，一个静态同步方法，一个普通同步方法。先打印邮件，还是短信？

```java
class Phone{
    public static synchronized void sendEmail() throws InterruptedException {
        Thread.sleep(3000);
        System.out.println("-----sendEmail");
    }

    public  synchronized void sendSMS(){
        System.out.println("-----sendSMS");
    }

    public void hello(){
        System.out.println("-----hello");
    }
}


public class demo8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        Phone phone2 = new Phone();
        new Thread(()-> {
            try {
                phone.sendEmail();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"a").start();

        Thread.sleep(2000);

        new Thread(()->{
            phone.sendSMS();
        },"b").start();
    }
}

```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107130716969.png)

#### 第八种

两部部手机，一个静态同步方法，一个普通同步方法。先打印邮件，还是短信？

```java
public class demo8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        Phone phone2 = new Phone();
        new Thread(()-> {
            try {
                phone.sendEmail();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"a").start();

        Thread.sleep(2000);

        new Thread(()->{
            phone2.sendSMS();
        },"b").start();
    }
}
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107130819922.png)

#### 总结

**1.由第一种和第二种可知：**

一个对象如果持有多个sychronized方法。某一时刻内，只有唯一线程能去访问这些sychronized方法。它锁的是当前的对象this，被锁定后，其他线程都不能进入当前对象的其他sychronized方法。

**2**.**由第三种和第四种可知**：

加上普通方法后发现和同步锁无关。

换成俩对象，就用的不是一把锁了。

**3**.**由第五种和第六种可知**：

普通同步方法：锁的是实例对象，通常指this，锁的是实例对象本身

静态同步方法：static sychronized是一把**类锁**。所有类new出来的实例公用的区域。

对于同步代码块:  锁的是括号内的对象。

**4**.**由第五种和第六种可知**：

由于普通同步方法所得是实例对象this，静态同步方法锁的是Class，所以普通同步方法和静态同步方法不会有竟态条件。

但是一个静态同步方法获得锁，其他静态同步方法也必须要等待它释放锁才能获得锁。

此时，我们对于着图的理解相必更加深入了。

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107123528658.png)

## sychronized字节码分析

### 同步代码块的字节码分析

```java
public class LockSyncDemo {

    Object object = new Object();

    public void m1(){
        synchronized (object){
            System.out.println("-------hello synchronized code block");
        }
    }

    public static void main(String[] args) {

    }
}
```

我们先运行这个空main方法来帮助编译LockSyncDemo这个类

然后在target目录下，找到他

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107162659526.png)

之后，我们cd进入这个目录，通过，反编译命令，看看他的字节码

```
javap -c .\LockSyncDemo.class
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107162912103.png)

我们在m1的字节码中发现，在执行sychronized同步代码块的时候，会有moniterenter和monitorexit俩指令。

### 普通同步方法的字节码分析

```java
public class LockSyncDemo {
    public synchronized void m2() {
        System.out.println("-------hello synchronized code block");
    }

    public static void main(String[] args) {
    }
}
```

由于这次我们需要看到更加细节的信息

```
javap -v .\LockSyncDemo.class
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107163613319.png)

我们在m2的字节码中发现，在执行sychronized普通同步方法的时候，调用指令会先检查方法的ACC_SYCHRONIZED访问标志是否被设置。如果设置了，执行线程会先持有monitior锁，然后再持有方法，最后方法完成的时候，释放monitor锁。

### 静态同步方法的字节码分析

```java
    public synchronized void m2() {
        System.out.println("-------hello synchronized m2");
    }

    public static synchronized void m3() {
        System.out.println("-------hello synchronized m3");
    }
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221107164530104.png)

我们发现静态同步方法和普通同步方法的分辨是依靠ACC_STATIC来进行分辨的。

## 公平锁和非公平锁

> 公平锁：指多个线程按照申请锁的顺序来获取锁，这里类似排队买票，先来的人先买后来的人在队尾排队，这是公平的。
>
> Lock lock = new ReentrantLock(true);
>
> 非公平锁：多个线程获得的顺序并非按照申请锁的顺序，有可能后申请的线程比先申请的线程优先获取到锁，在高并发的环境下，有可能造成优先级反转或者饥饿状态（某个线程一直获取不到锁）
>
> Lock lock = new ReentrantLock(false);
>
> Lock lock = new ReentrantLock();//默认非公平锁

#### 面试题：为什么会有公平锁/非公平锁的设计？为什么默认非公平？

1. 首先，我们得知道，恢复挂起的线程到真正锁的获取是有时间差的。从CPU的角度看，这个时间差还是很明显的，所以非公平锁能更充分利用CPU的时间片，减少CPU的空闲时间（原则：搞定就行，不管分配）
2. 多线程最重要的的考量点就是线程切换的开销，当采用非公平锁的时候。当一个线程请求锁获取同步状态，然后释放同步状态，而刚释放锁的线程在此刻再次获取到的同步状态的概率变得很大，所以减少了线程的kai

#### 使用场景

如果为了更高的吞吐量，公平锁是更合适的，因为节省了很多线程切换时间。吞吐量很容易上去。

否则就使用公平锁。

## 可重入锁（递归锁）

可重入锁是个啥?

其实就是同步代码块里面包同步代码块包同步代码块.....

> 然后这些同步代码块里锁的都是一个对象。如果第一层同步代码块对这个对象上了锁，那么在进入第二层的时候，不用解锁第一层，也可以进入第二层，以此类推

然而上述的同步代码块只是举个例子。事实上，只要是synchronized修饰的无论是同步代码块还是普通同步方法都可以进行可重入锁的实践。

```java
public class ReEntryLockDemo {


    public synchronized void m3(){
        System.out.println("asdasdas");
    }


    public synchronized void m2(){
        m3();
    }


    public synchronized void m1(){
        m2();
    }


    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getName() + "\t" + "start");
        new ReEntryLockDemo().m1();
        System.out.println(Thread.currentThread().getName() + "\t" + "end");
    }
}
```

### 原理分析

之前我们说过monitorenter指令。对于可重入锁，在执行monitorenter的时候，如果目标锁对象的计数器为0，那么说明他没有被其他线程所持有，Java虚拟机会将该锁对象的持有线程设置为当前线程，并且将计数器加一。

如果目标锁对象的计数器不为0，如果锁对象的持有线程是当前线程，那么Java虚拟机就可以将计数器加一，否则需要等待，直到持有线程释放该锁。

当执行monitorexit的时候，Java虚拟机则需要将锁对象的计数器减一。计数器为0则说明锁已经被释放了。

### 显示锁也有可重入锁

像sychronized这样的叫隐式锁。像Lock也有ReentrantLock其实也是可重入锁

```java
        lock.lock();
        try {
            System.out.println("第一层");
            lock.lock();
            try {
                System.out.println("第二层");
            }finally {
                lock.unlock();
            }
        }finally {
            lock.unlock();
        }
```

加锁和解锁必须要一一对应

## 死锁及排查

### 死锁是什么？

死锁是指两个或两个以上的线程在执行的过程中，因争夺资源二造成的一种互相等待的现象，若无外力干涉那他们就无法推进下去，如果系统资源充足，进程的资源请求都能够得到满足，死锁出现的概率就很低，否则就会因为争夺有限的资源而陷入死锁。

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108123732402.png)

> 吃着碗里，惦记着盘子里的！

### 手写死锁

```java
public class DeadLockDemo {
    public static void main(String[] args) {
        Object a = new Object();
        Object b = new Object();
        new Thread(()->{
            synchronized (a){
                System.out.println(Thread.currentThread().getName()+"\t我持有了a锁，希望获得b锁");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                synchronized (b){
                    System.out.println(Thread.currentThread().getName()+"\t成功获得b锁");
                }
            }
        },"A").start();

        new Thread(()->{
            synchronized (b){
                System.out.println(Thread.currentThread().getName()+"\t我持有了b锁，希望获得a锁");
                synchronized (a){
                    System.out.println(Thread.currentThread().getName()+"\t成功获得a锁");
                }
            }
        },"B").start();


    }
}
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108124425010.png)

此时A和B俩线程由于开始了资源的争夺，而陷入了死锁。

那如果说，这段代码，不是我们写的，而我们也看不出来这是死锁，那么我们如何进行故障的排查呢？

### 如何排查死锁

进入终端

```
jps -l
```

查看进程编号

```
jstack + 进程编号
```

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108125045709.png)

或者win + r

输入jconsole

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108131043174.png)

## ObjectMonitor

我们之前讨论过，为什么任何一个对象都可以成为一把锁。其实这和java的底层ObjectMonitor.hpp的C++代码有关

> 指针指向monitor对象（也叫管程或监视器锁），每个对象都存在一个monitor与之关联，当一个monitor被某个线程持有以后，他就会处于锁定状态。在java虚拟机中，monitor是由ObjectMonitor（C++）实现的，

下面是ObjectMonitor的主要数据结构

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108132111804.png)

重要属性

| _owner      | 指向ObjectMonitor对象的线程，写入线程Id |
| ----------- | --------------------------------------- |
| _WaitSet    | 存放于wait状态的线程队列                |
| _EntryList  | 存放于等待锁 block状态的线程队列        |
| _recursions | 锁的重入次数                            |
| _count      | 用来记录线程获取锁的次数                |

在使用sychronized（this）锁住一个对象以后，this会指向对象监视器的指针也就是ObjectMonitor。

在字节码里，在执行完sychronized（this）之后，字节码会执行

```
monitorenter
```

指令进行加锁，并判断_count是否等于0，如果是，则说明当前锁对象没有被占用，那么就执行加锁操作。如果不是就说明当前对象已经被占用。那么此时就会判断 _owner指向的是否是当前线程，如果是，则执行加锁，如果不是就把线程加入阻塞队列 _EntryList.

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108133200771.png)

在执行完加锁操作以后， _count会自增1， _owner会指向当前线程， _recursions也会自增1，然后才会执行同步代码块内部的任务。

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221108134153612.png)

在执行完以后，字节码会执行

```
monitorexit
```

指令，之后count会减1，recursions也会减1.

> 我们之前说过recursion记录可重入锁的进出次数

然后判断recursion是否为0，如果是0，就可以从多层sychronized里面退出了，如果不是，那么就不退出，保持owner不变

如果是，那就将owner擦除，表示线程完全释放了锁。

