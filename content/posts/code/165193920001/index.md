---
title: "生产者与消费者问题"
date: "2022-05-08"
tags: ["多线程"]
categories: ["编程知识"]
---

生产者消费者问题（Producer-consumer problem），也称有限缓冲问题（Bounded-buffer problem），是一个多线程同步问题的经典案例。生产者生成一定量的数据放到缓冲区中，然后重复此过程；与此同时，消费者也在缓冲区消耗这些数据。生产者和消费者之间必须保持同步，要保证生产者不会在缓冲区满时放入数据，消费者也不会在缓冲区空时消耗数据。不够完善的解决方法容易出现死锁的情况，此时进程都在等待唤醒。

下图为生产者和消费者的示意图：
![](images/1.png)

## 解决思想

保证同一资源被多个线程并发访问时的完整性。常用的同步方法是采用信号或加锁机制，保证资源在任意时刻至多被一个线程访问

### Java实现的几种方式

- wait() / notify() 方法
- await() / signal() 方法(可重入锁ReentrantLock)
- BlockingQueue 阻塞队列方法
- 信号量方法
- 管道方法

## 代码实现

### wait() / notify() 方法

首先，先介绍一下 `Thread.sleep()`和 `Object.wait()、Object.notify()`的区别。

1. `sleep()`是Thread类的方法；而 `wait()`，`notify()`，`notifyAll()`是Object类中定义的方法；尽管这两个方法都会影响线程的执行行为，但是本质上是有区别的
2. `Thread.sleep()`不会导致锁行为的改变，如果当前线程是拥有锁的，那么 `Thread.sleep()`不会让线程释放锁。如果能够帮助你记忆的话，可以简单认为和锁相关的方法都定义在Object类中，因此调用 `Thread.sleep()`是不会影响锁的相关行为
3. `Thread.sleep`和 `Object.wait`都会暂停当前的线程，对于CPU资源来说，不管是哪种方式暂停的线程，都表示它暂时不再需要CPU的执行时间。OS会将执行时间分配给其它线程。区别是调用wait后，需要别的线程执行notify/notifyAll才能够重新获得CPU执行时间
4. `Thread.sleep()`让线程从 【running】 -> 【阻塞态】 时间结束/interrupt -> 【runnable】;`Object.wait()`让线程从 【running】 -> 【等待队列】notify -> 【锁池】 -> 【runnable】

wait()和notify()方法的实现，缓冲区满和为空时都调用wait()方法等待，当生产者生产了一个数据或者消费者消费了一个数据之后会通过notify()唤醒所有线程。

```java
import java.util.*;

public class Test {
    // 缓冲区最大容量
    private static final int MAX_SIZE = 100;
    // 计数
    private static int count = 0;
    // 缓冲区
    private static LinkedList<Integer> list = new LinkedList<Integer>();

    public static void main(String[] args) {
        Thread producer = new Thread(new Producer());
        Thread consumer = new Thread(new Consumer());
        producer.start();
        consumer.start();

    }

    // 生产者
    public static class Producer implements Runnable {
        @Override
        public void run() {
            while (true) {
                // 为list上锁
                synchronized (list) {
                    // 缓冲区满时，等待
                    while (list.size() == MAX_SIZE) {
                        try {
                            System.out.println("list is full, Producer waiting");
                            list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    // 生产数据
                    count++;
                    System.out.println("Producer produce " + count);
                    list.add(count);
                    // 唤醒消费者
                    list.notifyAll();

                    // 等待一段时间再生产
                    try {
                        Thread.sleep(new Random().nextInt(1000));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                // 当生产100个数就结束
                if (count == 100) {
                    break;
                }
            }
        }
    }

    // 消费者
    public static class Consumer implements Runnable {
        @Override
        public void run() {
            while (true) {
                // 为list上锁
                synchronized (list) {
                    // 缓冲区空时，等待
                    while (list.isEmpty()) {
                        try {
                            System.out.println("list is empty, Consumer waiting");
                            list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费数据
                    int temp = list.poll();
                    System.out.println("Consumer consume " + temp);
                    // 唤醒生产者
                    list.notifyAll();
                }
                // 当消费100个数就结束
                if (count == 100 && list.isEmpty()) {
                    break;
                }
            }
        }
    }
}
```

执行结果如下：
![](images/2.png)

**注意：**

notifyAll()方法可使所有正在等待队列中等待同一共享资源的“全部”线程从等待状态退出，进入可运行状态。此时，优先级最高的那个线程最先执行，但也有可能是随机执行的，这要取决于JVM虚拟机的实现。即最终也只有一个线程能被运行，上述线程优先级都相同，每次运行的线程都不确定是哪个，后来给线程设置优先级后也跟预期不一样，还是要看JVM的具体实现吧。

### await() / signal() 方法(可重入锁ReentrantLock)

在JDK5.0之后，Java提供了更加健壮的线程处理机制，包括同步、锁定、线程池等，它们可以实现更细粒度的线程控制。用ReentrantLock和Condition可以实现等待/通知模型，具有更大的灵活性。通过在Lock对象上调用newCondition()方法，将条件变量和一个锁对象进行绑定，进而控制并发程序访问竞争资源的安全。

Condition接口的await()和signal()就是其中用来做同步的两种方法，它们的功能基本上和Object的wait()/ nofity()相同，完全可以取代它们，但是它们和新引入的锁定机制Lock直接挂钩，具有更大的灵活性。通过在Lock对象上调用newCondition()方法，将条件变量和一个锁对象进行绑定，进而控制并发程序访问竞争资源的安全。

```java
import java.util.*;
import java.util.concurrent.locks.*;

public class Test1 {
    private static final int MAX_SIZE = 100;
    private static int count = 0;
    private static LinkedList<Integer> list = new LinkedList<Integer>();

    // 创建锁及其条件
    private static final Lock lock = new ReentrantLock();
    private static final Condition fullCondition = lock.newCondition();
    private static final Condition emptyCondition = lock.newCondition();

    public static void main(String[] args) {
        Thread producer = new Thread(new Producer());
        Thread consumer = new Thread(new Consumer());
        producer.start();
        consumer.start();

    }

    public static class Producer implements Runnable {
        @Override
        public void run() {
            while (true) {
                // 上锁
                lock.lock();
                try {
                    // 缓冲区满时，等待
                    while (list.size() == MAX_SIZE) {
                        try {
                            System.out.println("list is full, Producer waiting");
                            fullCondition.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    // 生产数据
                    count++;
                    System.out.println("Producer produce " + count);
                    list.add(count);
                    // 唤醒消费者
                    emptyCondition.signalAll();
                } finally {
                    lock.unlock();
                }

                try {
                    Thread.sleep(new Random().nextInt(1000));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                if (count == 100) {
                    break;
                }
            }
        }
    }

    public static class Consumer implements Runnable {
        @Override
        public void run() {
            while (true) {
                // 上锁
                lock.lock();
                try {
                    // 缓冲区空时，等待
                    while (list.isEmpty()) {
                        try {
                            System.out.println("list is empty, Consumer waiting");
                            emptyCondition.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费数据
                    int temp = list.poll();
                    System.out.println("Consumer consume " + temp);
                    // 唤醒生产者
                    fullCondition.signalAll();
                } finally {
                    lock.unlock();
                }

                if (count == 100 && list.isEmpty()) {
                    break;
                }
            }
        }
    }
}
```

运行结果如下：（与第一种方法类似）
![](images/3.png)

### BlockingQueue 阻塞队列方法

JDK 1.5 以后新增的 java.util.concurrent包新增了 BlockingQueue 接口。并提供了如下几种阻塞队列实现：

- java.util.concurrent.ArrayBlockingQueue
- java.util.concurrent.LinkedBlockingQueue
- java.util.concurrent.SynchronousQueue
- java.util.concurrent.PriorityBlockingQueue

实现生产者-消费者模型使用 ArrayBlockingQueue或者 LinkedBlockingQueue即可。

我们这里使用LinkedBlockingQueue，它是一个已经在内部实现了同步的队列，实现方式采用的是我们第2种await()/ signal()方法。它可以在生成对象时指定容量大小。它用于阻塞操作的是put()和take()方法。

put()方法：类似于我们上面的生产者线程，容量达到最大时，自动阻塞。
take()方法：类似于我们上面的消费者线程，容量为0时，自动阻塞。

```java
import java.util.*;
import java.util.concurrent.LinkedBlockingQueue;

public class Test2 {
    private static final int MAX_SIZE = 100;
    private static int count = 0;
    // 创建阻塞队列
    private static LinkedBlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<Integer>(MAX_SIZE);

    public static void main(String[] args) {
        Thread producer = new Thread(new Producer());
        Thread consumer = new Thread(new Consumer());
        producer.start();
        consumer.start();

    }

    public static class Producer implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    count++;
                    // 生产数据到阻塞队列
                    blockingQueue.put(count);
                    System.out.println("Producer produce " + count);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                try {
                    Thread.sleep(new Random().nextInt(1000));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                if (count == 100) {
                    break;
                }
            }
        }
    }

    public static class Consumer implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    // 从阻塞队列消费数据
                    int value = blockingQueue.take();
                    System.out.println("Consumer consume " + value);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                if (count == 100 && blockingQueue.isEmpty()) {
                    break;
                }
            }
        }
    }
}
```

### 信号量方法

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源，在操作系统中是一个非常重要的问题，可以用来解决哲学家就餐问题。Java中的Semaphore维护了一个许可集，一开始先设定这个许可集的数量，可以使用acquire()方法获得一个许可，当许可不足时会被阻塞，release()添加一个许可。

Semaphore可以用来构建一些对象池，资源池之类的，比如数据库连接池，我们也可以创建计数为1的Semaphore，将其作为一种类似互斥锁的机制，这也叫二元信号量，表示两种互斥状态。

在下列代码中，还加入了另外一个mutex信号量，维护生产者消费者之间的同步关系，保证生产者和消费者之间的交替进行

```java
import java.util.*;
import java.util.concurrent.Semaphore;

public class Test3 {
    private static final int MAX_SIZE = 100;
    private static int count = 0;
    private static LinkedList<Integer> list = new LinkedList<Integer>();

    // 创建信号量
    final static Semaphore notFull = new Semaphore(MAX_SIZE);
    final static Semaphore notEmpty = new Semaphore(0);
    final static Semaphore mutex = new Semaphore(1);

    public static void main(String[] args) {
        Thread producer = new Thread(new Producer());
        Thread consumer = new Thread(new Consumer());
        producer.start();
        consumer.start();

    }

    public static class Producer implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    // 获取许可
                    notFull.acquire();
                    mutex.acquire();
                    // 生产数据
                    count++;
                    list.add(count);
                    System.out.println("Producer produce " + count);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    // 释放许可
                    mutex.release();
                    notEmpty.release();
                }

                try {
                    Thread.sleep(new Random().nextInt(1000));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                if (count == 100) {
                    break;
                }
            }
        }
    }

    public static class Consumer implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    // 获取许可
                    notEmpty.acquire();
                    mutex.acquire();
                    // 消费数据
                    int value = list.poll();
                    System.out.println("Consumer consume " + value);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    // 释放许可
                    mutex.release();
                    notFull.release();
                }

                if (count == 100 && list.isEmpty()) {
                    break;
                }
            }
        }
    }
}
```

### 管道方法

在java的io包下，PipedOutputStream和PipedInputStream分别是管道输出流和管道输入流。

它们的作用是让多线程可以通过管道进行线程间的通讯。在使用管道通信时，必须将PipedOutputStream和PipedInputStream配套使用。

使用方法：先创建一个管道输入流和管道输出流，然后将输入流和输出流进行连接，用生产者线程往管道输出流中写入数据，消费者在管道输入流中读取数据，这样就可以实现了不同线程间的相互通讯。

但是这种方式在生产者和生产者、消费者和消费者之间不能保证同步，也就是说在一个生产者和一个消费者的情况下是可以生产者和消费者之间交替运行的，多个生成者和多个消费者者之间则不行。

这种方式只适用于两个线程之间通信，不适合多个线程之间通信。

```java
import java.io.IOException;
import java.io.PipedInputStream;
import java.io.PipedOutputStream;
import java.util.*;

public class Test4 {
    // 控制生产和消费个100次
    private static int countP = 0;
    private static int countS = 0;

    // 创建管道输入流和管道输出流
    final static PipedInputStream pis = new PipedInputStream();
    final static PipedOutputStream pos = new PipedOutputStream();
    // 将输入流和输出流进行连接
    static {
        try {
            pis.connect(pos);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        Thread producer = new Thread(new Producer());
        Thread consumer = new Thread(new Consumer());
        producer.start();
        consumer.start();

    }

    public static class Producer implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    // 写入数据
                    countP++;
                    pos.write(countP);
                    pos.flush();
                    System.out.println("Producer produce " + countP);

                    Thread.sleep(new Random().nextInt(1000));

                    if (countP == 100) {
                        break;
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                try {
                    // 关闭输出流
                    pos.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static class Consumer implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    // 读取数据
                    countS++;
                    int value = pis.read();
                    System.out.println("Consumer consume " + value);

                    if (countS == 100) {
                        break;
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    // 关闭输入流
                    pis.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        }
    }
}
```