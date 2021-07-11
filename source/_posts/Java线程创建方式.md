---
title: Java线程创建方式
date: 2021-07-11 08:11:24
tags: [Java, 多线程]
banner_img: /img/01.png
---

一般来说我们比较常用的有以下三种方式，下面介绍它们的使用方法。

# 1 继承Thread类

通过继承 Thread 类，并重写它的 run 方法，就可以创建一个线程。    

```java
public class ExtendThread extends Thread {

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "is running...");
    }

    public static void main(String[] args) {
        // 创建线程
        ExtendThread thread = new ExtendThread();
        // 设置线程名称
        thread.setName("my-thread");
        // 运行线程
        thread.start();
    }
}
```

使用继承方式的好处是，在`run()`方法内获取当前线程直接使用this就可以了，无须使用`Thread.currentThread()`方法；不好的地方是Java不支持多继承，如果继承了`Thread`类，那么就不能再继承其他类。另外任务与代码没有分离，当多个线程执行一样的任务时需要多份任务代码

# 2 实现 Runnable 接口

实现`Runnable`类，并实现其`run()`方法，也可以创建一个线程。

```java
public class ImplRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "is running...");
    }

    public static void main(String[] args) {
        ImplRunnable thread = new ImplRunnable();
        new Thread(thread, "my-thread").start();
    }
}
```

`Runnable`接口是一个被`@FunctionalInterface`注解修饰，因此可以通过lambda表达式进行创建，因此使用`Runnable`方式创建线程也可以通过下列方式进行简化：

```java
public class ImplRunnableLambda {
    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "is running...");
        }, "my-thread").start();
    }
```

# 3 实现 Callable 接口，并结合 Future 实现

首先，要定义一个`Callable`实现类，并实现`call`方法；其次，通过`Future`的构造方法传入`Callable`实现类的实例；然后，把`FutureTask`作为`Thread`类的 target ，创建 `Thread `线程对象；最后，可以通过`Future`的`get`方法获取线程的运行结果。

```java
public class UseFuture {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<String> futureTask = new FutureTask<>(new ImplCallable());
        Thread thread = new Thread(futureTask);
        thread.start();
        System.out.println(futureTask.get()); // 获得线程运行后的返回值，阻塞式
    }
}

class ImplCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        return "thread execute finished";
    }
}
```

# 总结

- 相比于第一种方式，更推荐第二种方式。因为继承继承`Thread`类往往不符合里氏代换原则，而实现`Runnable`接口可以使编程更加灵活，对外暴露的细节较少，使用者只需要关注`run()`方法的实现上；

- `Runnable`和`Callable`接口的定义如下：

  ```java
  // Runnable接口
  @FunctionalInterface
  public interface Runnable {
      public abstract void run();
  }
  
  // Callable接口
  @FunctionalInterface
  public interface Callable<V> {
      V call() throws Exception;
  }
  ```

  通过对比两个接口定义可知`Runnable`和`Callable`有两点不同：（1）通过`call`方法可以获取返回值。前两种方式在任务结束后，无法直接获取执行结果，只能通过共享变量获取，而第三种则可解决这一问题；（2）`call`可以抛出要异常，`Runnable`则需要通过`setDefaultUncaughtExceptionHandler()`方法才能在主线程中获取子线程中的异常。

