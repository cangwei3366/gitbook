# 关键字：volatile详解

### volatile的作用详解

#### 防重排序

以单例模式下的双重检查加锁举例。

```java
private static volatile Singleton instance;

    public static Singleton getInstance(){
        if(instance==null){
            synchronized (Singleton.class){
                if(instance==null){
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
```

实例化一个对象可以分为以下三步：

* 分配内存空间
* 初始化对象
* 将内存空间的地址赋值给对应的引用

但由于操作系统可以对指令进行重排序，所以上面的过程也可能会变成如下过程：

* 分配内存空间
* 将内存空间的地址赋值给对应的引用
* 初始化对象

为了防止这个过程的重排序，我们需要将变量设置为volatile类型的变量。

#### 实现可见性

可见性问题是指在多线程环境下，一个线程修改了共享变量，而另一个线程却看不到。引起可见性问题的主要原因每个线程拥有自己的一个高速缓冲区——线程工作内存，通过volatile可以解决这个问题。

```java
public class VolatileTest {
    int a = 1;
    int b = 2;

    public void change(){
        a = 3;
        b = a;
    }

    public void print(){
        System.out.println("b="+b+";a="+a);
    }

    public static void main(String[] args) {
        while (true){
            final VolatileTest test = new VolatileTest();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test.change();
                }
            }).start();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test.print();
                }
            }).start();
        }
    }
}
```

直观上说，这段代码的结果只可能有两种：b=3;a=3 或 b=2;a=1。不过运行上面的代码(可能时间上要长一点)，你会发现除了上两种结果之外，还出现了第三种结果： b=3,a=1; 原因就是第一个线程将值a=3修改后，但是对第二个线程是不可见的，所以才出现这一结果。

#### 保证原子性

volatile不能保证完全的原子性，但保证单次的读/写操作具有原子性。

注：共享的long和double变量的为什么要用volatile?

因为long和double两种数据类型的操作可分为高32位和低32位两部分，因此普通的long或double类型读/写可能不是原子的。因此，鼓励大家将共享的long和double变量设置为volatile类型，这样能保证任何情况下对long和double的单次读/写操作都具有原子性。

### volatile的实现原理

#### 可见性实现

java代码

```java
instance = new Singleton(); // instance是volatile变量
```

通过 jitwatch 工具获取编译后的汇编代码

```
0x01a3de1d: movb $0×0,0×1104800(%esi);0x01a3de24: lock addl $0×0,(%esp);
```

Lock前缀的指令在多核处理器下会引发了两件事情。

1）将当前处理器缓存行的数据写回到系统内存。

2）这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存（L1，L2或其他）后

再进行操作，但操作完不知道何时会写到内存。

如果对声明了volatile的 变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的

数据 写回到系统内存。

但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操 作就会有问题。所以，在多处理器

下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来

检查自己缓存的值是不是过期了，当 处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行

设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

**缓存一致性**

#### 有序性实现

JMM内存模型的happens-before规则

#### 禁止重排序

为了实现 volatile 内存语义时，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎是不可能的，为此，JMM 采取了保守的策略。

* 在每个 volatile 写操作的前面插入一个 StoreStore 屏障。
* 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障。
* 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障。
* 在每个 volatile 读操作的后面插入一个 LoadStore 屏障。

volatile 写是在前面和后面分别插入内存屏障，而 volatile 读操作是在后面插入两个内存屏障。

| 内存屏障          | 说明                                       |
| ------------- | ---------------------------------------- |
| StoreStore 屏障 | 禁止上面的普通写和下面的 volatile 写重排序。              |
| StoreLoad 屏障  | 防止上面的 volatile 写与下面可能有的 volatile 读/写重排序。 |
| LoadLoad 屏障   | 禁止下面所有的普通读操作和上面的 volatile 读重排序。          |
| LoadStore 屏障  | 禁止下面所有的普通写操作和上面的 volatile 读重排序。          |

### 应用场景

### 使用优化&#x20;
