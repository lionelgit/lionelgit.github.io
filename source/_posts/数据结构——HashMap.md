---
title: 数据结构——HashMap
date: 2018-11-05 22:57:25
categories:
- 数据结构
tags:
- 数据结构
- 数组
- 红黑树
---

# HashMap底层

1.数组和链表

数组

存储空间连续，占用内存严重，连续的大内存进入老年代的可能性也会变大，但是正因如此，寻址就显得简单，但是增删时则需要把数据整体往前或往后移动。

<!--more-->

链表

存储空间不连续，占用内存较宽松，它的基本结构是一个节点（node）都会包含下一个节点的信息（如果双向链表会存在两个信息一个指向上一个，一个指向下一个），正因为如此寻址就会变得比较困难，插入和删除就显得容易，链表插入和删除的时候只需要修改节点指向信息就可以了。

2.哈希表/散列表（Hash Table非线程安全）

在哈希表的结构中就融入了数组和链表的结构，从而产生了一种寻址容易，插入删除也容易的新存储结构

![1532587148462](/pic/1532587148462.png)

为了解释方便，我们定义两个东西：String[] arr; 和 List list；  那么，上图左边那一列就是arr, 就是整个的arr[0]~arr[15]，且arr.length() = 16。上图每一行就是一个list，这里理论来说应该最大存储16个List。每个数组存存放的应该是某一个链表的头，也就是arr[0] == list.get(0)。不知道我这样的描述是否清楚。 

那么如何确定某一个对象是属于数组的某个下标呢？一般算法就是 **下标 = hash(key)%length**。算式中的key是存放的对象，hash这个对象会得到一个int值，这个int值就是在上图中所体现的数字，length就是这个数组的长度。我们用上图中的arr[1]打个比方，1%16 =1，337%16 = 1， 353%16 =1。大家就存储在arr[1]中。 

## HashMap详解

### 1.主要成员变量和方法

- loadFactor：称为**装载因子**，主要控制空间利用率和冲突。大致记住装载因子越大空间利用率更高，但是冲突可能也会变大，反之则相反。源码中默认0.75f。

```java
   /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

- DEFAULT_INITIAL_CAPACITY与MAXIMUM_CAPACITY：称为**容量**，用于控制HashMap大小的，下面是源码中的解释：

```java
    /**
     * The default initial初始 capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly隐式 specified指定
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;
```

- THRESHOLD: 这个字段主要是用于当HashMap的size大于它的时候，需要触发resize()方法进行扩容。下面是源码：

```java
    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.收缩
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection发现 under removal移动.
     */
    static final int UNTREEIFY_THRESHOLD = 6;
```

看到这里，也许一部分人心中会有一个疑问，为什么前面赋值要用移位的方式，而这里就直接赋值8而不用1<<3呢？注意一个点，在上面有一句解释：“ MUST be a power of two.”，意思就是说这个变量如果发生变化将会是以2的幂次扩容的。比如说1<<4进行扩容一位的话就是1<<4<1那结果就是32啦。

> 移位算法：
>
> - 左移位：（低位补0）
>
> 例如：5<<2,表示5左移两位
>
> 101→10100
>
> - 右移位：高位负数补1，正数补0
> - 无符号右移：无论正负高位都补0
> - 无论怎么移动，如果移动位数超过规定的bit数，都会与最大移位数取模之后进行计算
>
> 例如：int类型是32位的，如果5<<33,其实等价于5<<1

### 2.红黑树

红黑树是自平衡的二叉查找树(有序二叉树)，一般的二叉查找树的时间复杂度为O(lgn)，当一般的二叉查找树退化之后会变成一个线性的列表，这个时候它的时间复杂度就变为了O(n)(这个O(n)其实就是for循环/遍历一次)，但是**红黑树它所独有的着色和自平衡的一些性质使得时间复杂度最坏为O(logn)，而不是更低的O(n)**，这就是等下我们要说的为什么HashMap在**JDK1.8的时候引入冲突解决方案要用红黑树**。

![1532828155919](/pic/1532828155919.png)

从图中就可以看到：①对于任意一个节点，它的左子节点比它小，右子节点比它大， 其实它还有更多的性质。我们就不一一研究了。**我们只需要知道这一条性质和红黑树的优势就是可以自平衡**。注意，本博客中提到的树性质就是①。

节点插入:遍历树，根据上面描述的性质，只要根据key值的大小可以很快定位到新要插入的节点位置。如果插入破坏了红黑树本身的平衡，红黑树会进行旋转，重新着色进行调整。

节点删除：分为3种情况。①第一要删除的节点处于最外层，那么就可以直接删除。②如果它存在一个子节点，这个子节点直接顶替要删除的节点后并不会破坏整棵树的性质，③如果它存在两个子节点，就先需要拿后继节点来顶替它的位置，在把该节点删除。那小伙伴说什么是后继节点？就上图而言，根节点13的后继节点就是11和15，也就是说节点左边最靠右的，和右边最靠左的。我这么说不知道能否理解呐？删除之后也可能会重新调整树本身的平衡。

### 3、&与%：

因为在HashMap中并不会用%来表达取模运算，而是要用&来运算。也就是一定要记住如下等式：

A%B = A&(B-1)，源码中计算桶的位置都是用&来计算的，记住这个，看源码就会轻松一些。

### 4.红黑树源码

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  //父节点
    TreeNode<K,V> left;//左子节点
    TreeNode<K,V> right;//右子节点
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red; //节点颜色
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
}
```

### 5.核心hashMap

```java
//reszie()方法：
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table; //定义旧表并把原本的赋值给他，从昨天的博文中可以知道new了一个HashMap对象之后其实table是为null的。
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//如果旧表容量为null就初始0
        int oldThr = threshold;//旧表的阀值
        int newCap, newThr = 0;//定义新表的容量和新表的阀值
        //进入条件：正常扩容  
       if (oldCap > 0) {如果旧表容量大于0，这个情况就是要扩容了
       //进入条件：已达到最大，无法扩容
            if (oldCap >= MAXIMUM_CAPACITY) {//如果容量已经大于等于1<<30
                threshold = Integer.MAX_VALUE;//设置阀值最大
                return oldTab;//直接返回原本的对象(无法扩大了)
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)//旧表容量左移一位<<1,且移动之后处于合法的范围之中。新表容量扩充完成
                newThr = oldThr << 1; // 新表的阀值也扩大一倍。
        }
        //进入条件：初始化的时候使用了自定义加载因子的构造函数
        else if (oldThr > 0) // 这里如果执行的情况是原表容量为0的时候，但是阀值又不为0。hashmap的构造函数不同（需要设置自己的加载因子）的时候会触发。
            newCap = oldThr;
        //进入条件：调用无参或者一个参数的构造函数进入默认初始化
        else {               // 如果HashMap默认构造就会进入下面这个初始化，我们昨天（上一篇博文）的第一次put就会进入下面这一块。
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//初始化完成。
        }
        //进入条件：初始化的时候使用了自定义加载因子的构造函数
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;//新表容量*加载因子
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;//确定新的阀值
         //开始构造新表
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //进入条件：原表存在
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {//开始遍历
                    oldTab[j] = null;//旧表原本置空
                    if (e.next == null)//不存在下个节点，也就是目前put的就是链表头
                        newTab[e.hash & (newCap - 1)] = e;把该对象赋值给新表的某一个桶中
             //进入条件：判断桶中是否已红黑树存储的。如果是红黑树存储需要宁做判断
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
               //进入条件：如果桶中的值是合法的，也就是不止存在一个，也没有触发红黑树存储
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;//获取下一个对象信息
                            //因为桶已经扩容了两倍，所以以下部分是按一定逻辑的把一个链表拆分为两个链表，放入对应桶中。具体的拆分流程，各位看官们仔细研究下。笔者已经看得头晕了。
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

//put方法：
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

//putVal()方法：
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //进入条件:HashMap初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //进入条件:如果计算得到的桶中不存在别的对象
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);//直接把目前对象赋值给当前空对象中
        else {
            Node<K,V> e; K k;
            //进入条件：判断桶中第一个是否有相同的key值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
             //进入条件：如果该桶中的对象已经由红黑树构造了就特别处理
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                //进入条件：判断当前桶中对象存储是否达到8个，如果达到了就进入treeifyBin方法，这里不再进入方法详解。大致就是进入之后再判断当前hashMap的长度，如果不足64，只进行扩容，如果达到64就需要构建红黑树了。
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //进入条件：遍历中查看是否有相同的key值。如果有直接结束遍历并赋值
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //进入条件：如果链表上有相同的key值。进行替换
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)//这句笔者理解半天也未有所获，如果谁能清楚知道可以联系笔者一起探讨
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //如果hashMap的大小大于阀值就需要扩容操作
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }






//get()方法：
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

getNode()方法：

  final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //进入条件：如果HashMap不为空，且求得的数组下标那个对象不为空（关于%和&的关系上面方法已有讲过）
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //先判断头结点是否是要找的值
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
             //然后再进入链表遍历
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)//如果头链表已经是红黑树构造就需要用红黑树的方法去遍历
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                //直到找到拥有相同key的时候返回
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

