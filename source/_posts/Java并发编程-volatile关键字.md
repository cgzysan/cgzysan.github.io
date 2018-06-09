---
title: 'Java并发编程:volatile关键字'
date: 2018-05-26 10:33:20
tags: java并发 volatile
---

#### 前言

> valatile 关键字用到的不多，不过在源码中时常能够看到，对其既熟悉又陌生，熟悉的是这个名字，陌生的它的用意和用法，那么我们就对其一探究竟。

<!-- more -->

#### 内存模型相关概念

> 计算机在执行程序时，每条指令都是在CPU中执行的，过程中势必会涉及到数据的读取和写入，而数据却是存放在主存（物理内存）当中，CPU的速度特别快，但是内存的读取操作相对于CPU的运算速度来说很慢，如果任何时候对数据的操作都要通过和内存的交互来进行，会大大降低指令执行的速度。因此在CPU里就有了Cache高速缓存。

- CPU先把需要的数据从内存中读取复制一份到Cache
- CPU进行计算的时候直接从Cache读取数据和写入数据
- 结束后将Cache缓存中的数据刷新到主存

#### 多线程的问题

![内存模型](/Users/ysan/Nustore%20Files/cgzysan/ysan/momory_model.png)

现如今我们的手机都是多核的，也就是说同时有几颗CPU在工作，每颗CPU都有自己的Cache高速缓存，这样也就产生了一个问题：

- CPU1 和 CPU2 先将count变量读到各自的Cache中，假设都为1
- CPU1 和 CPU2 分别开启一个线程对count进行自增操作，每个线程都有一块独立内存空间存放count的副本，自增后的count副本都为2
- 自增完毕后，将结果副本写回内存，count等于2

进行了两次自增之后，写回内存的结果等于2，而自己想要的结果应该是count=3，这就是多线程并发带来的问题。

#### 并发编程的三要素

##### 1. 原子性

> 原子性，即一个操作或者多个操作，要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

一个经典的例子就是银行账户转账问题：<br>

如果从账户A向账户B转账100元，那么必然要有两个操作：

1. 从账户A减去100元
2. 往账户B加上100元

设想一个，这两个操作不具备原子性，如果从账户A减去100元之后，操作突然中止，这就会造成，账户A虽然减去了100元，但是账户B并没有收到转过来的100元。所以这两个操作必须要具备原子性。

##### 2. 可见性

> 可见性，指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看到修改的值

比如执行线程A的是CPU1，执行线程B的是CPU2，当线程A去修改一个值 value = 5，会先把value的初始值加载到CPU1的Cache中，然后赋值为10，那么在CPU1的高速缓存当中value的值变为了10，却没有立即写入到主存中。<br>而当线程B去主存获取 value 的值，并加载到线程B的高速缓存中，此时内存当中的value还是5，那么就会使得获取到的值是5，而不是修改的值10。<br>线程A对变量value修改之后，线程B没有立即看到线程A修改的值，这也就是可见性的问题。对于可见性，Java提供了 `volatile`关键字来保证可见性，被`volatile`修饰的共享变量，它将保证修改的值会立即更新到主存，当有其他线程需要读取的时候，也会去内存读取最新值。

##### 3. 有序性

> 有序性：即程序执行的顺序按照代码的先后顺序执行。

JVM在真正执行代码的时候不一定会保证按照代码的书写顺序进行，可能会发生指令重排序。

###### 指令重排序

一般来说，处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的。处理器在进行重排序时会考虑指令之间的数据依赖性，如果两个语句之间没有数据依赖性，可能就会被重排序。<br>指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性。

#### volatile关键字

1. volatile 保证可见性
2. volatile 不能确保原子性
3. volatile 保证有序性

普通的lock：

```java
public class AtomTest {
    
    int res = 0;
    boolean lock = true;
    
    private void setLock() {
        res = 130;
        lock = false;
    }
    
    private void readInfo() {
        while (lock) {
            
        };
        System.out.println("res = " + res);
    }
    
    
    public static void main(String[] args) throws InterruptedException {
        final AtomTest atom = new AtomTest();
        
        Thread readThread = new Thread(new Runnable() {
            
            @Override
            public void run() {
                atom.readInfo();
            }
        });
        
        Thread lockThread = new Thread(new Runnable() {
            
            @Override
            public void run() {
                atom.setLock();
            }
        });
           
        readThread.start();
        TimeUnit.SECONDS.sleep(1);
        lockThread.start();
        readThread.join();
        lockThread.join();
        // join 方法使等待线程结果，才结束程序
        System.out.println("end");
    }
}
```

没有被`volatile`修饰的lock，修改的仅仅是副本的值，而没有立马写入到内存中去，使readThread一直在死循环中，程序也就无法结束。<br>volatile修饰的lock：

```java
public class AtomTest {
    
    int res = 0;
    volatile boolean lock = true;
    
    private void setLock() {
        res = 130;
        lock = false;
    }
    
    private void readInfo() {
        while (lock) {
            
        };
        System.out.println("res = " + res);
    }
    
    
    public static void main(String[] args) throws InterruptedException {
        final AtomTest atom = new AtomTest();
        
        Thread readThread = new Thread(new Runnable() {
            
            @Override
            public void run() {
                atom.readInfo();
            }
        });
        
        Thread lockThread = new Thread(new Runnable() {
            
            @Override
            public void run() {
                atom.setLock();
            }
        });
           
        readThread.start();
        TimeUnit.SECONDS.sleep(1);
        lockThread.start();
        readThread.join();
        lockThread.join();
        // join 方法使等待线程结果，才结束程序
        System.out.println("end");
    }
}
```

打印结果：<br>res = 130<br>end<br>给lock加了volatile关键字修饰之后，当lockThread对lock做了修改之后，会立即更新修改到内存中，readThread发现副本已经过期，重新从内存获取lock的新值，跳出循环，执行完成。