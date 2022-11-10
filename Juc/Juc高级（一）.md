# Juc高级（一）

## CompletebleFuture简介

说到CompletebleFuture，大家也许并不陌生，它是一个处理异步多线程的java中非常好用的类。

在它诞生之前，也就是jdk1.5时期，有一个叫Future的接口，它也能处理异步多线程。

但是，它有很多缺陷。

- Future对于异步的线程，需要调用get（）方法来收集执行的结果。而在调用的时候，程序会一直处于阻塞状态，这和异步的理念相违背
- 对于异步线程是否完成的判断，需要用while（true）循环里调用isDone方法来询问是否完成，这对CPU的资源非常浪费

> 因此，对于真正的异步处理，我们希望可以传入回调函数，在任务结束的时候，自动调用回调函数。这样就不用去等待结果。

而在jdk8中设计了CompletableFuture。

它提供了一个类似于观察者模式的机制，可以让任务执行完成以后通知监听的一方。

这样的话，程序不会阻塞，并且不用做轮询，cpu资源也就不会浪费了！

### CompletableStage源码分析

在ComoletableFuture中会实现两个接口Future和CompletableStage

![](https://nika-picbed.oss-cn-hangzhou.aliyuncs.com/typoraimage-20221031203138057.png)

CompletionStage代表异步计算过程的某一个阶段，一个阶段完成后触发另一个阶段。

例如：

```java
stage.thenApply(x->square(x)).thenAccept(x->System.out.println(x)).thenRun(()->System.out.println())
```

>  因此，对于实现CompletionStage和Future的CompletableFuture，它有可能代表一个明确完成的Future，也有可能代表一个完成阶段。它支持计算完成以后触发一些函数或者执行某些动作。

### 核心的2种静态方法，来创建一个异步任务

由于官方文档的建议，它并不推荐使用new一个CompletableFuture的实例来调用方法，创建异步任务，因为这是不完全的。

#### 第一种 runAsync

我们通过CompletableFuture.runAsync（）的方式来创建。

但由于它里面只能传入runnable接口的实现和线程池，因此它没有返回值。

```java
 CompletableFuture<Void> completableFuture = CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println(Thread.currentThread().getName());
        });
        System.out.println(completableFuture.get());
```

![image-20221031130705165](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031130705165.png)

我们再试着自己传入一个线程池试试。

```java
 ExecutorService executorService = Executors.newFixedThreadPool(3);
        CompletableFuture<Void> completableFuture = CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println(Thread.currentThread().getName());
        },executorService);
        System.out.println(completableFuture.get());
```

![image-20221031131240184](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031131240184.png)

> completableFuture.get() 和 join() 的对比：
>
> get（）会产生检查型异常，编辑器报出，join是编译型异常，会在运行的时候报出

#### 第二种 supplyAsync

我们通过CompletableFuture.supplyAsync（）的方式来创建。它需要传入一个supplier和一个线程池。

Supplier是一个接口，里面只有一个get方法，我们需要实现它，并且返回一个值即可

![image-20221031131853677](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031131853677.png)

```java
        CompletableFuture<String> stringCompletableFuture = CompletableFuture.supplyAsync(()->{
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println(Thread.currentThread().getName());
            return "Hello" + "supplyAsync";
        },executorService);
        System.out.println(stringCompletableFuture.get());

        executorService.shutdown();
```

果然，他是有返回值的，我们需要用get将它获取。

![image-20221031132035159](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031132035159.png)

> 不要忘记关掉线程池噢！

### CompletableFuture--code演示，减少轮询和阻塞

我们都知道CompletableFuture是Future的增强版，减少**阻塞和轮询**，可以传入回调对象，当异步任务完成或者发生异常的时候。自动调用回调对象的回调方法。

我们刚刚的代码证明了它可以完成Future接口所完成的所有的事情，现在我们用代码看看，Future接口不能完成但是CompletableFuture可以完成的事情。

```java
 public static void main(String[] args) {
        CompletableFuture.supplyAsync(()->{
            System.out.println(Thread.currentThread().getName() + "----come in");
            int result = ThreadLocalRandom.current().nextInt(10);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return result;
        }).whenComplete((value,exception)->{
            if(exception == null){
                System.out.println("--计算完成，更新到的value是" + value);
            }
        }).exceptionally(exception -> {
            exception.printStackTrace();
            System.out.println("异常情况是" + exception.getCause() + "\t" +exception.getMessage());
            return null;
        });
        System.out.println(Thread.currentThread().getName()+ "---正在忙其他事情");
    }
```

对于第一个supplier的返回值，可以用whenComplete来接收，whenComplete接受俩参数第一个是返回值，第二个是异常。如果有异常的话，我们就会跟一个exceptionally进行异常的处理。

我们开始运行。

![image-20221031163414949](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031163414949.png)

奇怪了，为什么没有打印出值？

原来，自定义线程会自动关闭，此时的线程类似是一个守护线程，主线程执行时间过短，它还没执行完就跟着结束了，因此，我们可以让主线程沉睡3s。

或者我们自定义线程池，然后自己关闭就可。

```java
 ExecutorService threadPool = Executors.newFixedThreadPool(3);
        CompletableFuture.supplyAsync(()->{
            System.out.println(Thread.currentThread().getName() + "----come in");
            int result = ThreadLocalRandom.current().nextInt(10);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return result;
        },threadPool).whenComplete((value,exception)->{
            if(exception == null){
                System.out.println("--计算完成，更新到的value是" + value);
            }
        }).exceptionally(exception -> {
            exception.printStackTrace();
            System.out.println("异常情况是" + exception.getCause() + "\t" +exception.getMessage());
            return null;
        });
        System.out.println(Thread.currentThread().getName()+ "---正在忙其他事情");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        threadPool.shutdown();
```

![image-20221031163549149](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031163549149.png)

我们刚刚演示了没有异常出现的程序的结果，如果说有异常出现了，那又会是怎么样呢？

下面我们来模拟一个异常

```java
public static void main(String[] args) {
    ExecutorService threadPool = Executors.newFixedThreadPool(3);
    CompletableFuture.supplyAsync(()->{
        System.out.println(Thread.currentThread().getName() + "----come in");
        int result = ThreadLocalRandom.current().nextInt(10);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        if(result>5){
            int i = 10/0;
        }
        return result;
    },threadPool).whenComplete((value,exception)->{
        if(exception == null){
            System.out.println("--计算完成，更新到的value是" + value);
        }
    }).exceptionally(exception -> {
        exception.printStackTrace();
        System.out.println("异常情况是" + exception.getCause() + "\t" +exception.getMessage());
        return null;
    });
    System.out.println(Thread.currentThread().getName()+ "---正在忙其他事情");
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
    threadPool.shutdown();
}
```

![image-20221031164102584](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031164102584.png)

### CompletableFuture优点

1. 异步任务结束，会自动回调某个对象的方法（whenComplete）
2. 主线程设置好回调以后，不用关心异步任务的执行，异步任务之间可以顺序执行（supplyAsync->whenComplete->exceptionally）
3. 异步任务出错的时候，会自动回调某个对象的方法(exceptionally)

### 函数式接口名称

对于多线程里，我们时常会遇到一些函数式接口，同时，我们也需要去实现这些接口的方法。

> 比如我们之前用到的Supplier就是一个提供者，不给他东西，同样给我们返回值。Consumer则是一个贪婪的消费者，给他东西它不吐出来。Runnable很常见，无返回值无参数。BiConsumer则是binary的意思，需要俩参数，它更贪婪，之前的（value，exception）->{}用的就是biConsumer

![image-20221031165244431](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031165244431.png)

## 实战电商CompletableFuture

#### 需求

1. 需求说明

   - [x] 同一款产品，同一时候搜索出同款产品在各大电商平台的售价；
   - [x] 同一款产品，同时搜索出本产品在同一电商平台下，各个入驻卖家的售价

2. 输出返回

   - [x] 希望是一个List<String>

3. 解决方案

   **all in!!!!!**（多线程）

#### 解决

我们先设计一个电商类

这里价格我们用一个随机数来搞定

```java
class netMal{

    @Getter
    private String mallName;

    public netMal(String mallName) {
        this.mallName = mallName;
    }

    public double getPrice(String projectName){
        return ThreadLocalRandom.current().nextDouble()*2 + projectName.charAt(0);
    }
}
```

   并对他们的实例做好初始化

```java
    static List<netMal> netMals = Arrays.asList(
            new netMal("jingdong"),
            new netMal("taobao"),
            new netMal("suncong")
    );
```

我们需要获得一个盛放String的list去获取每个电商的商品的价格。

```java
    public static List<String> getPrices(String productName){
        return netMals.stream()
                .map(netMal ->
                        CompletableFuture.supplyAsync(
                                () -> String.format(productName + " in %s ,price is %.2f", netMal.getMallName(), netMal.getPrice(productName))))
                .collect(Collectors.toList())
                .stream()
                .map(task -> task.join())
                .collect(Collectors.toList());
    }
```

对于每一个电商，我们开启多线程对它进行商品的查询。

> 对于第一个collect，收集的是异步的任务（Stream<CompletableFutrue>）。第二个collect则是任务完成以后，异步获取到的结果，我们将结果进行一个收集

运行！

```java
  public static void main(String[] args) {

        List<String> prices = getPrices("mysql");

        System.out.println(prices.stream().toList());
    }
```

![image-20221031183257194](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031183257194.png)

## CompletableFuture常用Api

### 对于获得结果和触发判断的api

> 下面的completableFuture都是实例！！不是类

####  completableFuture.getNow（String value）

上demo

```java
CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return "Hello";
        });
        System.out.println(completableFuture.getNow("如果没返回值，那我就是返回值！"));
```

我们知道，在main方法里，这个线程相当于守护线程，由于休眠2000ms，肯定比主方法执行的慢。所以这个一定是没有返回值的。

那么这个getNow就是为了应对这个情况的发生而存在的。如果没有返回值，那么传入的参数就是返回值。

![image-20221031184623732](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031184623732.png)

#### completableFuture.get（）

前面一直在用，不做赘述

#### completableFuture.get(long timeout,TimeUnit unit)

这个方法可以设置过时的时间。如果调用get方法的时候，他会先等一段时间（timeout），如果超过这个时间，那他就不等候了，就会抛异常，如果没有超过这个时间，那么就无事发生，正常返回返回值

#### completableFuture.join（）

join方法和get几乎一致，但是get（）需要在方法上抛出异常，或者直接捕获。但是join不用，等真正碰到异常的时候它会自动抛出。

所以join更加方便！！

#### completableFuture.complete(String value)

对于这个方法，在执行到它的时候，如果说，任务执行完，那么就会返回true，没有就会返回false。

那有同学可能会好奇了。既然这样传入value干嘛。

其实跟上面的` completableFuture.getNow（String value）`一样，它起到一个替代返回值的效果。它只能判断是否完成了任务。因此还需要一个join（）方法来收集返回值

```java
    public static void main(String[] args) throws InterruptedException {
        CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return "Hello";
        });
//        System.out.println(completableFuture.getNow("如果没返回值，那我就是返回值！"));
        Thread.sleep(3000);
        System.out.println(completableFuture.complete("如果没返回值，那我就是返回值！") + "\t" +completableFuture.join());

```

![image-20221031190148899](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031190148899.png)

### 对于计算结果进行处理的api

#### thenApply

上demo！

```java
 public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return 1;
        },executorService).thenApply(num -> {
            System.out.println("第一次处理数据");
            return num + 1;
        }).thenApply(num2 -> {
            System.out.println("第二次处理数据");
            return num2 + 1;
        }).thenApply(num3 -> {
            System.out.println("第二次处理数据");
            return num3 + 1;
        }).whenComplete((finalNum, exception) -> {
            if (exception == null) {
                System.out.println("最终处理数据的值" + finalNum);
            }
        }).exceptionally(exception -> {
            exception.printStackTrace();
            System.out.println("exception:" + exception.getMessage());
            return null;
        });
        executorService.shutdown();
    }
```

我们会发现，这个thenApply可以无限向下延申处理。只要有一环出错，那么它就会直接跳到exceptionally去处理这个异常。

比如我们在第二次修改数据的时候加上这个代码

```
int i = 10/0
```

![image-20221031192149891](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031192149891.png)

但是如果说我们希望，某一环即使出错，也希望她继续进行下一环，只是对出错的那一环进行抛出异常并且停止这一环继续怎么办？

这个时候，就得用另一个Api了。

#### handle（value,exception）

```java
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return 1;
        },executorService).handle((num,e) -> {
            System.out.println("第一次处理数据");
            System.out.println(num);
            return num + 1;
        }).handle((num2,e) -> {
            System.out.println("第二次处理数据");
            int i = 10/0;
            System.out.println(num2);
            return num2 + 1;
        }).handle((num3,e) -> {
            System.out.println("第三次处理数据");
            System.out.println(num3);
            return num3 + 1;
        }).whenComplete((finalNum, exception) -> {
            if (exception == null) {
                System.out.println("最终处理数据的值" + finalNum);
            }
        }).exceptionally(exception -> {
            exception.printStackTrace();
            System.out.println("exception:" + exception.getMessage());
            return null;
        });
        executorService.shutdown();

    }
```

![image-20221031192451475](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031192451475.png)

我们会发现，现在虽然第二次抛异常了，但是第三次还是会去执行。

但是对于thenApply，handle需要传入的是一个biFuction

![image-20221031192600893](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031192600893.png)

也就是说，我们需要传入两个参数，但是最终只会返回一个结果。

### 对于计算结果进行消费的api

消费消费，也就是白嫖！传入参数，没有返回！

上demo！

```java
 public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        CompletableFuture<Void> completableFuture = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return 1;
        }, executorService).thenApply(num1 -> {
            System.out.println("num=" + num1);
            return num1 + 1;
        }).thenAccept(num -> {
            System.out.println("最终num=" + num +"\t被消费掉了 ---- 嘤嘤嘤");
        });

        executorService.shutdown();
    }
```

![image-20221031193216508](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031193216508.png)

对比于thenAccept，thenAccept其实就是接受者，并且很贪婪，做了一个结尾的工作！

### 对于计算结果既不消费也不处理的api

####  thenRun（Runnable runnable）

任务a执行完任务b，并且b不需要a的结果

```java
        System.out.println(CompletableFuture.supplyAsync(()->"result").thenRun(()-> System.out.println("我只是在干自己的事情")).join());

```

![image-20221031195045078](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031195045078.png)

先打印了thenRun里面的方法，然后join收集了一下，返回值是null，说明确实没有返回值。

## CompletableFuture和线程池说明

### thenRun和thenRunSync的区别？

我们在使用方法thenRun，thenApply的时候，发现往往idea的提示里面都有thenRunAsync，thenApplyAsync，这俩有啥区别呢？

假设现在有两个任务需要执行。我们传入了一个自定义的线程池。

我们用thenRun执行第二个任务的时候，第一个任务和第二个任务都用的是一个线程池也就是我们之前传入的自定义线程池；如果调用thenRunAsync方法执行第二个任务，那么第一个任务用的是自己自定义的线程池，第二个则是ForkJoinPool线程池（系统默认）

>  thenApply，thenApplyAsync同理

**那为什么会这样呢？我们来看看源码**

![image-20221031200728276](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031200728276.png)

它用了defaultExecutor，defaultExecutor又是啥呢？

<img src="C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031200736989.png" alt="image-20221031200736989" style="zoom:150%;" />

![image-20221031200815188](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031200815188.png)

这里用了3元运算符。

![image-20221031200837515](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031200837515.png)

意思是如果说你的cpu核数大于1，那么就自动用forkJoinPool！！

市面上现在大部分电脑cpu都大于1，所以用async方法一定是用forkJoinPool这个线程池的！

### 比较谁是赢家——applyToEither（）

在企业里，往往会有不同方法去处理一个多线程问题。为了比较每个方法的性能进行取舍。

出现了applyToEither方法。

```java
  public static void main(String[] args) {
        CompletableFuture<String> completableFutureA = CompletableFuture.supplyAsync(() -> {
            System.out.println("A come in");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return "A";
        });
        CompletableFuture<String> completableFutureB = CompletableFuture.supplyAsync(() -> {
            System.out.println("B come in");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return "B";
        });

        CompletableFuture<String> completableFuture = completableFutureA.applyToEither(completableFutureB, f -> {
            return f + "\tis winner";
        });

        System.out.println(completableFuture.join());
    }
```

我们知道，A睡了1s，B睡了2s，所以A一定是更好性能的。我们对于获胜的completableFuture给予冠军奖励。我们看看A能否夺冠

![image-20221031201912699](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031201912699.png)

没问题！

它需要传入一个实现了CompletionStage的东西，没错！CompletableFuture正是实现了CompletionStage和Future！

![image-20221031201935055](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031201935055.png)

并且还要传入一个有进有出的Fuction。

### 不同的CompletableFuture结果合并——thenCombile（）

![image-20221031202535024](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031202535024.png)

thenCombine方法要求传入一个CompletionStage，和一个接受俩参数的fuction

上demo！

```java
public static void main(String[] args) {
        CompletableFuture<Integer> completableFuture1 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return 10;
        });
        CompletableFuture<Integer> completableFuture2 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return 20;
        });

        CompletableFuture<Object> completableFuture = completableFuture1.thenCombine(completableFuture2, (x, y) -> {
            System.out.println("最终的结果" + (x + y));
            return (x + y);
        });
        System.out.println(completableFuture.join());
    }
```

对于先执行完的线程会等待后执行完的线程进行结果的合并，成为一个新的

CompletableFuture。

![image-20221031202819144](C:/Users/ht/AppData/Roaming/Typora/typora-user-images/image-20221031202819144.png)

Over！！
