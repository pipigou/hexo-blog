---
title: Java线程相关的常用方法
date: 2021-07-11 15:26:55
tags: [Java, 多线程]
banner_img: /img/02.jpg
---

# 1 start和run方法

new一个`Thread`，线程进入了新建状态。调用`start()`方法，会启动一个线程并使线程进入了就绪状态，当分配到时间片后就可以开始运行了。`start()`会执行线程的相应准备工作，然后自动执行`run()`方法的内容，这是真正的多线程工作。 但是，直接执行 `run()` 方法，会把 `run()` 方法当成一个 main 线程下的普通方法去执行，并不会在某个线程中执行它，所以这并不是多线程工作。

> **调用`start()`方法方可启动线程并使线程进入就绪状态，直接执行`run()`方法的话不会以多线程的方式执行。**

# 2 sleep方法

1. 调用 `sleep` 会让当前线程从 **Running 进入 Timed Waiting 状态（阻塞）**，让出指定时间的CPU执行权，该段时间内不再参与CPU调度，但线程拥有的监视器资源不会被释放，比如持有的锁；
2. 睡眠时间到了之后，会进入到就绪状态，等待CPU分配时间段执行；
3. 其它线程可以使用**`interrupt`** 方法打断正在睡眠的线程，这时 sleep 方法会抛出`InterruptedException`。

```
public class SleepTest01 {
    private static final Lock lock = new ReentrantLock();

    public static void main(String[] args) {
        // 线程A
        Thread threadA = new Thread(() -> {
            // 获取独占锁
            lock.lock();
            try {
                System.out.println("Thread A is in sleep");
                Thread.sleep(2500);
                System.out.println("Thread A is in awaked");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        });

        // 线程B
        Thread threadB = new Thread(() -> {
            // 获取独占锁
            lock.lock();
            try {
                System.out.println("Thread B is in sleep");
                Thread.sleep(2500);
                System.out.println("Thread B is in awaked");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        });

        threadA.start();
        threadB.start();
    }
}
```

以上代码无论执行多少次，总是A或者B先执行，并不会出现A和B交替运行的情况。

下面的代码演示了子线程在睡眠期间被主线程打断的情况：

```java
public class SleepTest02 {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            try {
                System.out.println("Sub thread is in sleep");
                Thread.sleep(5000);
                System.out.println("Sub thread is in awaked");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread.start();        // 子线程执行
        Thread.sleep(2000);    // 主线程睡眠2000ms
        thread.interrupt();    // 终端子线程
    }
}
```

其执行结果如下：

```bash
Sub thread is in sleep
java.lang.InterruptedException: sleep interrupted
	at java.base/java.lang.Thread.sleep(Native Method)
	at cn.tgq007.sleep.SleepTest02.lambda$main$0(SleepTest02.java:13)
	at java.base/java.lang.Thread.run(Thread.java:831)
```

# 3 yield方法

当一个线程调用yield方法时，当前线程会让出CPU使用权，然后处于就绪状态，线程调度器会从就绪队列中选取一个优先级最高的线程执行，也有可能调度到刚刚让出CPU的那个线程来获取CPU执行权。该方法一般很少用。

> `sleep`与`yield`比较：sleep与yield的区别是，当线程调用sleep是，该线程会进入阻塞状态，在睡眠期间CPU不会对该线程进行调用。而线程调用yield方法时，线程只是让出自己的CPU时间片，并未被阻塞，处于就绪状态，调度器在下一轮的调度时就有可能调到当前线程执行。

# 4 join方法

等待调用join方法的线程结束，再去执行后续任务。在项目实践中会遇到需要某几件事情完成后才继续往下执行，比如多个线程加载资源，需要等待多个线程加载完成后汇总处理。

下面的代码演示了其使用方法：

```
public class JoinTest {
    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> {
            try {
                Thread.sleep(2500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("Thread A done!");
        });

        Thread threadB = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("Thread B done!");
        });

        threadA.start();
        threadB.start();
        System.out.println("Wait for all sub thread done!");
        threadA.join();
        threadB.join();
        System.out.println("continuing");
    }
}
```

其运行结果如下：

```bash
Wait for all sub thread done!
Thread B done!
Thread A done!
continuing
```

在主线程中调子线程调用`join`方法后，则主线程将会进入阻塞状态，直到子线程运行结束。上述代码中，主线程`threadA.join()`后进入阻塞状态，当结束后，由于此时线程B已经执行完成则无需阻塞，进入后续的步骤。线程A开始到两个线程结束，大约耗时2500毫秒。

# 5 线程中断方法

Java中的线程中断时一种线程间的协作模式，通过设置线程的中断标志并不能直接将线程杀死，而是由被中断线程根据中断状态自行处理。

- `void interrupt()`方法：中断线程。当线程A运行时，线程B可以调用线程A的`interrupt`方法设置其中断标志并立即返回，再次强调：**只是设置标志**。如果线程A调用了`wait`函数、join方法或sleep方法，这时线程B再调用线程A的`interrupt`方法，则会抛出`InterruptedException`异常。

- `boolean isInterrupted()`方法：判断是当前线程是否设置了中断标志。

  ```java
   public boolean isInterrupted() {
          return interrupted;
      }
  ```

- `boolean interrupted()`方法：检测当前线程是否设置了中断标志，与`isInterrupted()`方法不同的是，其会将中断标志设置为`false`，并通过本地方法`clearInterruptEvent`清除中断事件。

  ```
  public static boolean interrupted() {
          Thread t = currentThread();
          boolean interrupted = t.interrupted;
          if (interrupted) {
              t.interrupted = false;
              clearInterruptEvent();
          }
          return interrupted;
      }
  ```
