# 什么是volitile

`volatile`可以说是一个轻量级的`synchronized`，一般作用于**变量**，在多处理器开发的过程中保证了内存的可见性。相比于`synchronized`关键字，`volatile`关键字的执行成本更低，效率更高。

# volatile的作用

> 并发编程的三大特性为可见性、有序性和原子性。通常来讲`volatile`可以保证可见性和有序性。

- 可见性：`volatile`可以保证不同线程对共享变量进行操作时的可见性。即当一个线程修改了共享变量时，另一个线程可以读取到共享变量被修改后的值。
- 有序性：`volatile`会通过禁止指令重排序进而保证有序性。
- 原子性：对于单个的`volatile`修饰的变量的读写是可以保证原子性的，但对于`i++`这种复合操作并不能保证原子性。这句话的意思基本上就是说`volatile`不具备原子性了。

## Java内存的可见性问题

Java的内存模型如下图所示。

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4a6ed02692c404e86524246e8b577e4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

这里的本地内存并不是真实存在的，只是Java内存模型的一个抽象概念，它包含了控制器、运算器、缓存等。同时Java内存模型规定，线程对共享变量的操作必须在自己的本地内存中进行，不能直接在主内存中操作共享变量。这种内存模型会出现什么问题呢？，

1. 线程A获取到共享变量X的值，此时本地内存A中没有X的值，所以加载主内存中的X值并缓存到本地内存A中，线程A修改X的值为1，并将X的值刷新到主内存中，这时主内存及本地内存A中的X的值都为1。
2. 线程B需要获取共享变量X的值，此时本地内存B中没有X的值，加载主内存中的X值并缓存到本地内存B中，此时X的值为1。线程B修改X的值为2，并刷新到主内存中，此时主内存及本地内存B中的X值为2，本地内存A中的X值为1。
3. 线程A再次获取共享变量X的值，此时本地内存中存在X的值，所以直接从本地内存中A获取到了X为1的值，但此时主内存中X的值为2，到此出现了所谓内存不可见的问题。

该问题Java内存模型是通过`synchronized`关键字和`volatile`关键字就可以解决。

## 为什么代码会重排序？

计算机在执行程序的过程中，编译器和处理器通常会对指令进行重排序，这样做的目的是为了提高性能。具体可以看下面这个例子。

```java
java复制代码int a = 1;
int b = 2;
int a1 = a;
int b1 = b;
int a2 = a + a;
int b2 = b + b;
......
```

像这段代码，不断地交替读取a和b，会导致寄存器频繁交替存储a和b，使得代码性能下降，可对其进入如下重排序。

```java
java复制代码int a = 1;
int b = 2;
int a1 = a;
int a2 = a + a;
int b1 = b;
int b2 = b + b;
......
```

按照这样地顺序执行代码便可以避免交替读取a和b，这就是重排序地意义。

### 重排序在并发情况下会引发什么问题？

在单线程程序中，重排序并不会影响程序的运行结果，而在多线程场景下就不一定了。

```java
class ReorderExample{
    int a = 0;
    boolean flag = false;
    public void writer(){
        a = 1;              // 操作1
        flag = true;        // 操作2
    }
    public void reader(){
        if(flag){          // 操作3
            int i = a + a; // 操作4
        }
    }
}
```

假设线程1先执行`writer()`方法，随后线程2执行`reader()`方法，最后程序一定会得到正确的结果吗？

答案是不一定的，如果代码按照下图的执行顺序执行代码则会出现问题。

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94b71c7f9a6b4b708bb27af54359651a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

操作1和操作2进行了重排序，线程1先执行`flag=true`，然后线程2执行操作3和操作4，线程2执行操作4时不能正确读取到`a`的值，导致最终程序运行结果出问题。这也说明了在多线程代码中，重排序会破坏多线程程序的语义。

# voliatile的实现原理

## volatile实现内存可见性原理

导致内存不可见的主要原因就是Java内存模型中的本地内存和主内存之间的值不一致所导致。

`volatile`可以保证内存可见性的关键是`volatile`的读/写实现了缓存一致性，缓存一致性的主要内容为：

- 每个处理器会通过嗅探总线上的数据来查看自己的本地数据是否过期，一旦处理器发现自己缓存对应的内存地址被修改，就会将当前处理器的缓存设为无效状态。此时，如果处理器需要获取这个数据需重新从主内存将其读取到本地内存。
- 当处理器写数据时，如果发现操作的是共享变量，会通知其他处理器将该变量的缓存设为无效状态。

那缓存一致性是如何实现的呢？可以发现通过`volatile`修饰的变量，生成汇编指令时会比普通的变量多出一个`Lock`指令，这个`Lock`指令就是`volatile`关键字可以保证内存可见性的关键，它主要有两个作用：

- 将当前处理器缓存的数据刷新到主内存。
- 刷新到主内存时会使得其他处理器缓存的该内存地址的数据无效。

## volatile实现有序性原理

为了实现`volatile`的内存语义，编译器在生成字节码时会通过插入内存屏障来禁止指令重排序。

内存屏障：内存屏障是一种CPU指令，它的作用是对该指令前和指令后的一些操作产生一定的约束，保证一些操作按顺序执行。

* 写屏障会确保指令重排序时，不会将写屏障之前的代码排在写屏障之后;写屏障仅仅是保证之后的读能够读到最新的结果，但不能保证读跑到它前面去.

* 读屏障会确保指令重排序时，不会将读屏障之后的代码排在读屏障之前.保证后面读到的都是最新结果。

# volatile能使一个非原子操作变成一个原子操作吗

`volatile`只能保证可见性和有序性，对简单读写可以保证原子性。

如但可以保证64位的`long`型和`double`型变量的原子性。

`对于32位的虚拟机来说，每次原子读写都是32位的，会将`long`和`double`型变量拆分成两个32位的操作来执行，这样long和double型变量的读写就不能保证原子性了，而通过volatile修饰的long和double型变量则可以保证其原子性`。

# volatile、synchronized的区别

- `volatile`主要是保证内存的可见性和指令重排。`synchronized`主要是解决多个线程访问资源的同步性。
- `volatile`作用于变量，`synchronized`作用于代码块或者方法。
- `volatile`仅可以保证数据的可见性，不能保证数据的原子性。`synchronized`可以保证数据的可见性和原子性。
- `volatile`不会造成线程的阻塞，`synchronized`会造成线程的阻塞。