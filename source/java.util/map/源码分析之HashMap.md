# 源码分析之HashMap

> 众所周知，HashMap是一个健值都可为null的线程不安全的Map。抱着学习的态度，通过源码的方式，看能不能了解到更多。
>
> 坚持，时间会给出答案。

## 静态字段

```java
// 默认初始容量16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;// aka 16

// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表的长度大于8，去执行树化。但要与 MIN_TREEIFY_CAPACITY 共同判断。
static final int TREEIFY_THRESHOLD = 8;

// 树转化为链表的最小长度
static final int UNTREEIFY_THRESHOLD = 6;

// 当table的长度大于等于64时，才执行树化。由下图可以看到，小于64时，会对HashMap进行扩容。
static final int MIN_TREEIFY_CAPACITY = 64;
```

![image-20201208093313731](/Users/xyh/Desktop/my-note/source/java.util/map/源码分析之HashMap.assets/image-20201208093313731.png)

## 成员变量

```java
// 存放数据的Node数组
transient Node<K,V>[] table;

// 
transient Set<Map.Entry<K,V>> entrySet;

// 键值对的数量
transient int size;

// 好像没啥用，先不深究此字段 todo
transient int modCount;

// capacity * load factor，简单理解就是数组中可以真正可用的长度
// 扩容的标识
int threshold;

// 


```






## 构造器

```java
/**
 * 
 * 自定义初始容量和加载因子，并计算threshold
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    // 计算threshold
    this.threshold = tableSizeFor(initialCapacity);
}

// 指定初始容量，使用默认加载因子 0.75f
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 默认的构造器，只指定了加载因子 = 0.75f
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```



## 重要的方法

```java
// 根据key计算hash值，算法的精妙之处暂不分析 todo
static final int hash(Object key) {
    int h;
    // HashMap的key一定要重写hashCode() 函数
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 根据传入的初始容量计算 threahold ，todo
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}


```



## CRUD的执行流程



### HashMap的get()方法

```java
public V get(Object key) {
    Node<K,V> e;
    // 实质上是调用了内部方法getNode(int hash, Object key);
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    // 初始化各种变量
    // Node[] tab = 成员变量table
    // Node first = 链表的第一个值
    // Node e = 中间值Node
    // int n = 数组的长度，
    // K k = 可能是第一个Node的key，也可能是每个Node的Key，只是个中间值
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    // ① table不为null并且 数组长度大于0 并且通过hash函数计算得到的槽位不是null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) { // tab[(n - 1) & hash] 这句话很妙，等待以后分析 todo
        
        // ② 判断链表的第一个Node的key是否与传入的key相等
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        
        // ③ 判断链表有没有下一个值
        if ((e = first.next) != null) {
            
            // ④ 先判断这个链表是不是一棵树，如果是用树的形式去获取value，暂时不管树的方式 todo
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            
            // ⑤ do {} while() 循环 遍历链表的每一个节点
            do {
                
                // 于①的比较方式相等
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```



① 第一步通过判断 tab不为null、长度大于0和链表的第一个值不为null来确定是否要继续

② 第二部通过判断第一个链表头结点是否与传入的key相等，相等则直接返回。通过这个判断得知，**HashMap的key必须要重写hashCode() 函数和equals() 函数**  

③ 判断链表的长度是否大于1，如果等于1并且传入的key，与链表头结点的key不相等则返回null

④ 如果该链表是树，则用树的方式获取。等我学完红黑树再展开讨论， **TODO**

⑤ do {} while() 循环并比较key



### HashMap的put()方法

```java
 public V put(K key, V value) {
     // 实质上是调用了内部方法putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict);
     return putVal(hash(key), key, value, false, true);
 }

/**
 * @param onlyIfAbsent 如果是true，不替换已存在的value
 * @param evict 如果为false，则HashMap处于创建模式。
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    // 初始化各种变量
    // Node[] tab = 成员变量table
    // Node p = 当前槽位的第一个值
    // int n = 数组的长度
    // int i = 当前槽位的数组下标
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // ①
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // ②
    if ((p = tab[i = (n - 1) & hash]) == null) // (n - 1) & hash] 又是这个神奇的算法
        tab[i] = newNode(hash, key, value, null);
    else {
        
        // Node e = 临时变量
        // K k = 当前key
        Node<K,V> e; K k;
        
        // ③
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // ④
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            
            // ⑤ 为啥不成写while(true){} 呢？哈哈哈哈
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // ⑥
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    
    // ⑦
    if (++size > threshold)
        resize();
    
    // ⑧
    afterNodeInsertion(evict);
    return null;
}
```

① 判断当前table是否初始化，如果未初始化，则调用resize() 进行初始化后再执行后续操作  
② 判断当前槽位是否为null，为null说明没有发生过hash碰撞，直接newNode()赋值。  
③ 进入else分支，说明发生了hash碰撞，需要进一步比较，首先依据传入的key与链表第一个节点key是否相等，相等赋值给临时变量，直接进入第⑥步  
④ 其次判断是否是树，是则用树的方式插入，然后返回当前节点 **TODO** ，进入第⑥步  
⑤ 循环链表，找到相等的key，则结束循环进入第⑥步。没有则在链表末端插入，然后判断链表是否可以转化为树**TODO ** 
⑥ 是否替换旧值为新值，默认替换掉  
⑦ 判断是否达到扩容条件  
⑧ 不知道啥意思，哈哈哈**TODO**  



#### 假如容量为16并且key的分布均匀，不会发生碰撞，每个key都只占一个槽位。分析下put()函数

第一步只会在第一次put时进行初始化。  

第二步每次都进入，并不会进入else分支。  

最后直接来到第七步，判断是否达到扩容条件，此时的HashMap就好像是一个数组而已，对内存的利用率不高，会导致频繁扩容，但这只是我们的假如，哈哈哈哈。  

#### 假如初始容量很小，导致一直发生碰撞，分析下put()函数

第一步只会在第一次put时进行初始化。  

第二步每次都进入else分支  

第三步不会进入if判断，因为我们模拟的数据hashCode是相等的，value是不相等的。  

第四步在一开始的时候不会进入，因为刚开始的时候，槽位的类型是Node型，而不是TreeNode型  

第五步会进入for循环，判断每一个Node节点的key、key的hashCode与传入的是否相等，相等退出循环，不相等插入到链表的末尾，并退出循环。就是在此处，判断链表是否满足树化条件（TREEIFY_THRESHOLD > 8 && MIN_TREEIFY_CAPACITY >= 64）  

此时的HashMap对数组的利用率极低，像一个链表或树，只用到了HashMap的一个槽位。这也间接说明了hash函数的重要性。

#### hash函数

先看源码  

![image-20201208100121052](/Users/xyh/Desktop/my-note/source/java.util/map/源码分析之HashMap.assets/image-20201208100121052.png)

再看引用  

![image-20201208100057689](/Users/xyh/Desktop/my-note/source/java.util/map/源码分析之HashMap.assets/image-20201208100057689.png)

![image-20201208100034207](/Users/xyh/Desktop/my-note/source/java.util/map/源码分析之HashMap.assets/image-20201208100034207.png)



学习位运算看这篇文章[位运算](./././other/位运算.md)





#### 模拟一组不会发生hash碰撞Key，进行debug



### HashMap的remove()方法



## 内部类







