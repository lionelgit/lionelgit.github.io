---
layout: 'n'
title: 数据结构——ConcurrentHashMap
date: 2018-11-05 22:58:01
categories:
- 数据结构
tags:
- 数据结构
- 乐观锁
- 悲观锁
- CAS算法
- volatile
- 线程安全
---

## 技术点：

### 1.悲观锁和乐观锁

悲观锁是指如果一个进程占用了一个锁，而导致其他需要这个锁的线程进入等待，一直到该锁被释放，换句话说就是这个锁被独占，比如典型的synchronized；

乐观锁是指操作并不加锁，而是抱着尝试的心态去执行某项操作，如果操作失败或者操作冲突，那么就进入重试，一直到执行成功为止。

<!--more-->

### 2.原子性，指令有序性和线程可见性

这三个性质是多线程编程中核心的问题。

原子性和事务的原子性一样，对于一个或多个操作，要么都执行，要么都不执行。指令有序性是指，在我们编写的代码中，**上下两个互不关联的语句不会被指令重排序**。

**指令重排序**是指处理器为了性能优化，在无关联的代码的执行是可能会和代码顺序不一致。比如说int i=1;int j=2;那么这两条语句的执行顺序可能会先执行int j=2；

**线程可见性**是指一个线程修改了某个变量，其他线程马上能知道。

### 3.无锁算法

使用低层原子化的机器指令，保证并发情况下数据的完整性，典型的如CAS算法。

### 4.内存屏障

在《深入理解JVM》中解释：它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障之后；即在执行到内存屏障这句指令时，它前面的操作已经全部完成；它会强制对缓存的修改操作立即写入主存；如果是写操作，它会导致其他CPU中对应的缓存行无效。在使用volatile修饰的变量会产生内存屏障。

## Java内存模型

![1536022937215](/pic/1536022937215.png)

### 1.解释：

每个线程会从主内存中读取操作，所有的变量都存储在主内存中，每个线程都需要从主内存中获得变量的值。

然后如图可见，每个线程获得数据之后会放入自己的工作内存，这就是java内存模型的规定之二，保证每个线程操作的都是从主内存拷贝的副本，也就是说线程不能直接操作主内存中的变量，需要把主内存的变量值读取之后放入自己的工作内存中的变量副本中，然后操作这个副本。

最后线程和线程之间无法直接访问对方工作内存中的变量。最后需要解释一下这个访问规则局限于对象实例字段，静态字段等，局部变量不包括在内，因为局部变量不存在竞争问题。

### 2.基本执行步骤：

a.lock(锁定)：在某一个线程在读取主内存的时候需要把变量锁定

b.unlock（解锁）：在某一个线程读取完变量值之后会释放锁定，别的线程就可以进入操作

c.read（读取）：从主内存中读取变量的值并放入工作内存中

d.load（加载）：从read操作中得到的值放入工作内存变量副本中

e.use（使用）：把工作内存中的一个变量值传递给执行引擎

f.assign（赋值）：它把一个从执行引擎接收到的值赋值给工作内存的变量

g.store（存储）：把工作内存中的一个变量的值传送到主内存中

h.write（写入）：把store操作从工作内存中的一个变量的值传送到主内存的变量中

![1536023910091](/pic/1536023910091.png)

## volatile关键字

在concurrentHashMap中，有很多成员变量都是用volatile修饰的，被volatile修饰的变量有如下特性：

1.使得变量更新变得具有可见性，只有被volatile修饰的变量的赋值一旦变化就会通知到其他线程，如果其他线程的工作内存中存在这个变量的拷贝副本，那么其他线程就会放弃这个副本中变量的值，重新去主内存中获取

2.产生了内存屏障，防止指令进行了重排序，关于这点的解释，见如下代码：

```java
public class VolatileTest {

    int a = 0;                 //1
    int b = 1;                 //2
    volatile int c = 2;        //3
    int d = 3;                 //4
    int e = 4;                 //5

}
```

如上所示c变量是被volatile修饰的，那么就会被该段代码产生了一个内存屏障，可以保证在执行语句3的时候，语句1,2是绝对执行完毕的，语句4,5肯定没有执行。上述语句中虽然保证了语句3的执行顺序不可变换，但1,2；4,5语句有可能发生指令重排序。

总结：volatile修饰的变量具有可见性和有序性

```java
public class VolatileTest {

//  int a = 0;                 //1
//  int b = 1;                 //2
    public static volatile int c = 0;        //3
//  int d = 3;                 //4
//  int e = 4;                 //5


    public static void increase(){
        c++;
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 1000; i++) {
            new Thread(new Runnable() {
                public void run() {
                    increase();
                }
            }
            ).start();
        }
        Thread.sleep(5000);
        System.out.println(c);
    }
}

//运行3次结果分别是：997，995，989
```

这个就是典型的volatile操作与原子性的概念。执行结果是小于等于1000的，为什么会这样呢？不是说volatile修饰的变量是具有原子性的么？是的，volatile修饰的变量的确具有原子性，也就是c是具有原子性的（直接赋值是原子性的），但是c++不具有原子性，c++其实就是c = c +1，已经存在了多步操作。所以c具有原子性，但是c++这个操作不具有原子性。

根据前面介绍的java内存模型，当有一个线程去读取主内存的过程中获取c的值，并拷贝一份放入自己的工作内存中，在对c进行+1操作的时候线程阻塞了（各种阻塞情况），那么这个时候有别的线程进入读取c的值，因为有一个线程阻塞就导致该线程无法体现出可见性，导致别的线程的工作内存不会失效，那么它还是从主内存中读取c的值，也会正常的+1操作。如此便导致了结果是小于等于1000的。

**注意**，这里笔者也有个没有深刻理解的问题，首先在java内存模型中规定了：在对主内存的unlock操作之前必须要执行write操作，那意思就是c在写回之前别的线程是无法读取c的。然而结果却并非如此。如果哪位朋友能理解其中的原委，请与我联系，大家一起讨论研究。

## CAS算法

CAS的全称叫“Compare And Swap”，也就是比较与交换，他主要的操作思想是：

首先它具有三个操作数，内存位置V，预期值A和新值B，如果在执行过程中，发现内存中的值V与预期值A相匹配，那么他会将V更新为新值A。如果预期值A和内存中的值V不相匹配，那么处理器就不会执行任何操作。CAS算法就是技术点中说的“无锁定算法”，因为线程不必再等待锁定，只要执行CAS操作就可以，会在预期中完成。



# **如何实现线程安全：**  

我们都知道ConcurrentHashMap核心是线程安全的，那么它又是用什么来实现线程安全的呢？在jdk1.8中主要是采用了CAS算法实现线程安全的。在上一篇博文中已经介绍了CAS的无锁操作，这里不再赘述。同时它通过CAS算法又实现了3种原子操作（线程安全的保障就是操作具有原子性），下面我赋值了源码分别表示哪些成员变量采用了CAS算法，然后又是哪些方法实现了操作的原子性:

```java
 // Unsafe mechanics  CAS保障了哪些成员变量操作是原子性的

    private static final sun.misc.Unsafe U;
    private static final long LOCKSTATE;
      static {
            try {
                U = sun.misc.Unsafe.getUnsafe();
                Class<?> k = TreeBin.class; //操作TreeBin,后面会介绍这个类
                LOCKSTATE = U.objectFieldOffset
                    (k.getDeclaredField("lockState"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
--------------------------------------------------------------------------------------
    private static final sun.misc.Unsafe U;
    private static final long SIZECTL;
    private static final long TRANSFERINDEX;
    private static final long BASECOUNT;
    private static final long CELLSBUSY;
    private static final long CELLVALUE;
    private static final long ABASE;
    private static final int ASHIFT;

    static {
        try {
        //以下变量会在下面介绍到
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentHashMap.class;
            SIZECTL = U.objectFieldOffset
                (k.getDeclaredField("sizeCtl"));
            TRANSFERINDEX = U.objectFieldOffset
                (k.getDeclaredField("transferIndex"));
            BASECOUNT = U.objectFieldOffset
                (k.getDeclaredField("baseCount"));
            CELLSBUSY = U.objectFieldOffset
                (k.getDeclaredField("cellsBusy"));
            Class<?> ck = CounterCell.class;
            CELLVALUE = U.objectFieldOffset
                (ck.getDeclaredField("value"));
            Class<?> ak = Node[].class;
            ABASE = U.arrayBaseOffset(ak);
            int scale = U.arrayIndexScale(ak);
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
//3个原子性操作方法：

    /* ---------------- Table element access -------------- */

    /*
     * Volatile access methods are used for table elements as well as
     * elements of in-progress next table while resizing.  All uses of
     * the tab arguments must be null checked by callers.  All callers
     * also paranoically precheck that tab's length is not zero (or an
     * equivalent check), thus ensuring that any index argument taking
     * the form of a hash value anded with (length - 1) is a valid
     * index.  Note that, to be correct wrt arbitrary concurrency
     * errors by users, these checks must operate on local variables,
     * which accounts for some odd-looking inline assignments below.
     * Note that calls to setTabAt always occur within locked regions,
     * and so in principle require only release ordering, not
     * full volatile semantics, but are currently coded as volatile
     * writes to be conservative.
     */

    @SuppressWarnings("unchecked")
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

以上这些基本实现了线程安全，还有一点是jdk1.8优化的结果，在以前的ConcurrentHashMap中是锁定了Segment，而在jdk1.8被移除，现在锁定的是一个Node头节点（注意，synchronized锁定的是头结点，这一点从下面的源码中就可以看出来），减小了锁的粒度，性能和冲突都会减少，以下是源码中的体现：

```java
//这段代码其实是在扩容阶段对头节点的锁定，其实还有很多地方不一一列举。
               synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                        .....
                   }
                }
```

## **如何存储数据：** 

知道了ConcurrentHashMap是如何实现线程安全的同时，最起码我们还要知道ConcurrentHashMap又是怎么实现数据存储的。以下是存储的图： 

![1536648927992](/pic/1536648927992.png)



有人看了之后会想，这个不是HashMap的存储结构么？在jdk1.8中取消了segment，所以结构其实和HashMap是极其相似的，在HashMap的基础上实现了线程安全，同时在每一个“桶”中的节点会被锁定。

### **重要的成员变量**：

#### 1、capacity：

容量，表示目前map的存储大小，在源码中分为默认和最大，默认是在没有指定容量大小的时候会赋予这个值，最大表示当容量达到这个值时，不再支持扩容。

```java
    /**
     * The largest possible table capacity.  This value must be
     * exactly 1<<30 to stay within Java array allocation and indexing
     * bounds for power of two table sizes, and is further required
     * because the top two bits of 32bit hash fields are used for
     * control purposes.
     */
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The default initial table capacity.  Must be a power of 2
     * (i.e., at least 1) and at most MAXIMUM_CAPACITY.
     */
    private static final int DEFAULT_CAPACITY = 16;
```

#### 2、laodfactor：

加载因子，这个和HashMap是一样的，默认值也是0.75f。有不清楚的可以去寻找上篇介绍HashMap的博文。

```java
    /**
     * The load factor for this table. Overrides of this value in
     * constructors affect only the initial table capacity.  The
     * actual floating point value isn't normally used -- it is
     * simpler to use expressions such as {@code n - (n >>> 2)} for
     * the associated resizing threshold.
     */
    private static final float LOAD_FACTOR = 0.75f;
```

#### 3、TREEIFY_THRESHOLD与UNTREEIFY_THRESHOLD：

作为了解，这个两个主要是控制链表和红黑树转化的，前者表示大于这个值，需要把链表转换为红黑树，后者表示如果红黑树的节点小于这个值需要重新转化为链表。关于为什么要把链表转化为红黑树，在HashMap的介绍中，我已经详细解释过了。

```java
    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2, and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;
```

4、下面3个参数作为了解，主要是在扩容和参与扩容（当线程进入put的时候，发现该map正在扩容，那么它会协助扩容）的时候使用，在下一篇博文中会简单介绍到。

```java
    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
     */
    private static int RESIZE_STAMP_BITS = 16;

    /**
     * The maximum number of threads that can help resize.
     * Must fit in 32 - RESIZE_STAMP_BITS bits.
     */
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    /**
     * The bit shift for recording size stamp in sizeCtl.
     */
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
```

5、下面2个字段比较重要，是线程判断map当前处于什么阶段。MOVED表示该节点是个forwarding Node，表示有线程处理过了。后者表示判断到这个节点是一个树节点。

```java
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees12
```

6、sizeCtl，标志控制符。这个参数非常重要，出现在ConcurrentHashMap的各个阶段，不同的值也表示不同情况和不同功能： 
①负数代表正在进行初始化或扩容操作 
②-N 表示有N-1个线程正在进行扩容操作 （前面已经说过了，当线程进行值添加的时候判断到正在扩容，它就会协助扩容） 
③正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，类似于扩容阈值。它的值始终是当前ConcurrentHashMap容量的0.75倍，这与loadfactor是对应的。实际容量>=sizeCtl，则扩容。

注意：在某些情况下，这个值就相当于HashMap中的threshold阀值。用于控制扩容。

### **极其重要的几个内部类：** 

如果要理解ConcurrentHashMap的底层，必须要了解它相关联的一些内部类。

#### 1、Node

```java
    /**
     * Key-value entry.  This class is never exported out as a
     * user-mutable Map.Entry (i.e., one supporting setValue; see
     * MapEntry below), but can be used for read-only traversals used
     * in bulk tasks.  Subclasses of Node with a negative hash field
     * are special, and contain null keys and values (but are never
     * exported).  Otherwise, keys and vals are never null.
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;  //用volatile修饰
        volatile Node<K,V> next;//用volatile修饰

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
        public final String toString(){ return key + "=" + val; }
        public final V setValue(V value) {
            throw new UnsupportedOperationException();  //不可以直接setValue
        }

        public final boolean equals(Object o) {
            Object k, v, u; Map.Entry<?,?> e;
            return ((o instanceof Map.Entry) &&
                    (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                    (v = e.getValue()) != null &&
                    (k == key || k.equals(key)) &&
                    (v == (u = val) || v.equals(u)));
        }

        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```

从上面的Node内部类源码可以看出，它的value 和 next是用volatile修饰的，关于volatile已经在前面一篇博文介绍过，使得value和next具有可见性和有序性，从而保证线程安全。同时大家仔细看过代码就会发现setValue（）方法访问是会抛出异常，是禁止用该方法直接设置value值的。同时它还错了一个find的方法，该方法主要是用户寻找某一个节点。

#### 2、TreeNode和TreeBin

```java
 /**
     * Nodes for use in TreeBins
     */
    static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }

        Node<K,V> find(int h, Object k) {
            return findTreeNode(h, k, null);
        }

        /**
         * Returns the TreeNode (or null if not found) for the given key
         * starting at given root.
         */
        final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
            if (k != null) {
                TreeNode<K,V> p = this;
                do  {
                    int ph, dir; K pk; TreeNode<K,V> q;
                    TreeNode<K,V> pl = p.left, pr = p.right;
                    if ((ph = p.hash) > h)
                        p = pl;
                    else if (ph < h)
                        p = pr;
                    else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                        return p;
                    else if (pl == null)
                        p = pr;
                    else if (pr == null)
                        p = pl;
                    else if ((kc != null ||
                              (kc = comparableClassFor(k)) != null) &&
                             (dir = compareComparables(kc, k, pk)) != 0)
                        p = (dir < 0) ? pl : pr;
                    else if ((q = pr.findTreeNode(h, k, kc)) != null)
                        return q;
                    else
                        p = pl;
                } while (p != null);
            }
            return null;
        }
    }


//TreeBin太长，笔者截取了它的构造方法：

 TreeBin(TreeNode<K,V> b) {
            super(TREEBIN, null, null, null);
            this.first = b;
            TreeNode<K,V> r = null;
            for (TreeNode<K,V> x = b, next; x != null; x = next) {
                next = (TreeNode<K,V>)x.next;
                x.left = x.right = null;
                if (r == null) {
                    x.parent = null;
                    x.red = false;
                    r = x;
                }
                else {
                    K k = x.key;
                    int h = x.hash;
                    Class<?> kc = null;
                    for (TreeNode<K,V> p = r;;) {
                        int dir, ph;
                        K pk = p.key;
                        if ((ph = p.hash) > h)
                            dir = -1;
                        else if (ph < h)
                            dir = 1;
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);
                            TreeNode<K,V> xp = p;
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            x.parent = xp;
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            r = balanceInsertion(r, x);
                            break;
                        }
                    }
                }
            }
            this.root = r;
            assert checkInvariants(root);
        }
```

从上面的源码可以看出，在ConcurrentHashMap中不是直接存储TreeNode来实现的，而是用TreeBin来包装TreeNode来实现的。也就是说在实际的ConcurrentHashMap桶中，存放的是TreeBin对象，而不是TreeNode对象。之所以TreeNode继承自Node是为了附带next指针，而这个next指针可以在TreeBin中寻找下一个TreeNode，这里也是与HashMap之间比较大的区别。

#### 3、ForwordingNode

```java
    /**
     * A node inserted at head of bins during transfer operations.
     */
    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }

        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) {
                    int eh; K ek;
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            return e.find(h, k);
                    }
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }
```

这个静态内部内就显得独具匠心，它的使用主要是在扩容阶段，它是链接两个table的节点类，有一个next属性用于指向下一个table，注意要理解这个table，它并不是说有2个table，而是在扩容的时候当线程读取到这个地方发现这个地方为空，这会设置为forwordingNode，或者线程处理完该节点也会设置该节点为forwordingNode，别的线程发现这个forwordingNode会继续向后执行遍历，这样一来就很好的解决了多线程安全的问题。这里有小伙伴就会问，那一个线程开始处理这个节点还没处理完，别的线程进来怎么办，而且这个节点还不是forwordingNode呐？说明你前面没看详细，在处理某个节点（桶里面第一个节点）的时候会对该节点上锁，上面文章中我已经说过了。

认识阶段就写到这里，对这些东西有一定的了解，在下一篇，也就是尾篇中，我会逐字逐句来介绍transfer（）扩容，put（）添加和get（）查询三个方法。

# 引言

### **transfer方法（扩容方法）** 

再这之前，我大致描述一下扩容的过程：首先**有且只能**由一个线程构建一个nextTable，这个nextTable主要是扩容后的数组（容量已经扩大），然后把原table复制到nextTable中，这个过程可以多线程共同操作。但是一定要清楚，这个复制并不是简单的把原table的数据直接移动到nextTable中，而是需要有一定的规律和算法操控的（不然怎么把树转化为链表呢）。

再这之前，先简单说下复制的过程： 
数组中（桶中）总共分为3种存储情况：空，链表头，TreeBin头 
①遍历原来的数组（原table），如果数组中某个值为空，则直接放置一个forwordingNode（上篇博文介绍过）。 
②如果数组中某个值不为空，而是一个链表头结点，那么就对这个链表进行拆分为两个链表，存储到nextTable对应的两个位置。 
③如果数组中某个值不为空，而是一个TreeBin头结点，那么这个地方就存储的是红黑树的结构，这样一来，处理就会变得相对比较复杂，就需要先判断需不需要把树转换为链表，做完一系列的处理，然后把对应的结果存储在nextTable的对应两个位置。

在上一篇博文中介绍过，多个线程进行扩容操作的时候，会判断原table的值，如果这个值是forwordingNode就表示这个节点被处理过了，就直接继续往下找。接下来，我们针对源码逐字逐句介绍：

```java
    /**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride; //stride 主要和CPU相关
        //主要是判断CPU处理的量，如果小于16则直接赋值16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating只能有一个线程进行构造nextTable，如果别的线程进入发现不为空就不用构造nextTable了
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1]; //把新的数组变为原来的两倍，这里的n<<1就是向左移动一位，也就是乘以二。
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n; //原先扩容大小
        }
        int nextn = nextTab.length;
        //构造一个ForwardingNode用于多线程之间的共同扩容情况
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true; //遍历的确认标志
        boolean finishing = false; // to ensure sweep before committing nextTab
        //遍历每个节点
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh; //定义一个节点和一个节点状态判断标志fh
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                //下面就是一个CAS计算
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                //如果原table已经复制结束
                if (finishing) {
                    nextTable = null; //可以看出在扩容的时候nextTable只是类似于一个temp用完会丢掉
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1); //修改扩容后的阀值，应该是现在容量的0.75倍
                    return;//结束循环
                }
                //采用CAS算法更新SizeCtl。
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //CAS算法获取某一个数组的节点，为空就设为forwordingNode
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
           //如果这个节点的hash值是MOVED，就表示这个节点是forwordingNode节点，就表示这个节点已经被处理过了，直接跳过
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
            //对头节点进行加锁，禁止别的线程进入
                synchronized (f) {
                //CAS校验这个节点是否在table对应的i处
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        //如果这个节点的确是链表节点
                        //把链表拆分成两个小列表并存储到nextTable对应的两个位置
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            //CAS存储在nextTable的i位置上
                            setTabAt(nextTab, i, ln);
                            //CAS存储在nextTable的i+n位置上
                            setTabAt(nextTab, i + n, hn);
                            //CAS在原table的i处设置forwordingNode节点，表示这个这个节点已经处理完毕
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        //如果这个节点是红黑树
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            //如果拆分后的树的节点数量已经少于6个就需要重新转化为链表
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                                //CAS存储在nextTable的i位置上
                            setTabAt(nextTab, i, ln);
                              //CAS存储在nextTable的i+n位置上
                            setTabAt(nextTab, i + n, hn);
                            //CAS在原table的i处设置forwordingNode节点，表示这个这个节点已经处理完毕
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

### **PUT方法** 

再这之前，先简单说一下PUT的具体操作： 
①先传入一个k和v的键值对，不可为空（HashMap是可以为空的），如果为空就直接报错。 
②接着去判断table是否为空，如果为空就进入初始化阶段。 
③如果判断数组中某个指定的桶是空的，那就直接把键值对插入到这个桶中作为头节点，而且这个操作不用加锁。 
④如果这个要插入的桶中的hash值为-1，也就是MOVED状态（也就是这个节点是forwordingNode），那就是说明有线程正在进行扩容操作，那么当前线程就进入协助扩容阶段。 
⑤需要把数据插入到链表或者树中，如果这个节点是一个链表节点，那么就遍历这个链表，如果发现有相同的key值就更新value值，如果遍历完了都没有发现相同的key值，就需要在链表的尾部插入该数据。插入结束之后判断该链表节点个数是否大于8，如果大于就需要把链表转化为红黑树存储。 
⑥如果这个节点是一个红黑树节点，那就需要按照树的插入规则进行插入。 
⑦put结束之后，需要给map已存储的数量+1，在addCount方法中判断是否需要扩容

```java
    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key和value都不可为空，为空直接抛出错误
        if (key == null || value == null) throw new NullPointerException();
        //计算Hash值，确定数组下标，这个和HashMap是一样的，我再HashMap的第一篇有介绍过
        int hash = spread(key.hashCode());
        int binCount = 0;
        //进入无线循环，直到插入为止
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //如果table为空或者容量为0就表示没有初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();//初始化数组
             //CAS如果查询数组的某个桶是空的，就直接插入该桶中
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin这句话的意思是这个时候插入不用加锁
            }
            //如果在插入的时候，节点是一个forwordingNode状态，表示正在扩容，那么当前线程进行帮助扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //锁定头节点
                synchronized (f) {
                //确定这个节点的确是数组中的这个头结点
                    if (tabAt(tab, i) == f) {
                    //是个链表
                        if (fh >= 0) {
                            binCount = 1;
                            //遍历这个链表
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //如果遍历到一个值，这个值和当前的key是相同的，那就更改value值
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //如果遍历到结束都没有遇到相同的key，且后面没有节点了，那就直接在尾部插入一个
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //如果是红黑树存储就需要用红黑树的专门处理了，笔者不再展开。
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                //判断节点数量是否大于8，如果大于就需要把链表转化成红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        //map已存储的数量+1
        addCount(1L, binCount);
        return null;
    }
```

其实，相对于transfer来说，PUT理解起来是不是简单很多？说到transfer，咋在PUT方法中都没出现过，只有一个helpTransfer（协助扩容）方法呢？其实，transfer方法放在了addCount方法中，下面是addCount方法的源码：

```java
    private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        //是否需要进行扩容操作
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                //如果小于0就说明已经再扩容或者已经在初始化
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                        //如果是正在扩容就协助扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                //如果正在初始化就首次发起扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

### **GET方法**

Get方法不论是在HashMap和ConcurrentHashMap都是最容易理解的，它的主要步骤是： 
①先判断数组的桶中的第一个节点是否寻找的对象是为链表还是红黑树， 
②如果是红黑树另外做处理 
③如果是链表就先判断头节点是否为要查找的节点，如果不是那么就遍历这个链表查询 
④如果都不是，那就返回null值。

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        //数组已被初始化且指定桶中不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            //先判断头节点，如果头节点的hash值与入参key的hash值相同
            if ((eh = e.hash) == h) {
            //头节点的key就是传入的key
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;//返回头节点的value值
            }
            //eh<0表示这个节点是红黑树
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;//直接从树上进行查找返回结果，不存在就返回null

            //如果首节点不是查找对象且不是红黑树结构，那边就遍历这个列表
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        //都没有找到就直接返回null值
        return null;
    }
```

