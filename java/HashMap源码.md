# 文章总览

1. JDK1.7和JDK1.8的区别
2. 属性解释
3. `put()`过程解析
4. 计算`threshold`
5. JDK1.8扩容优化
6. JDK1.7死循环图解

# 版本区别

## JDK1.7

- 数组+链表，即使哈希函数取得再好，也很难达到元素百分百均匀分布。
- 当 HashMap 中有大量的元素都存放到同一个桶中时，这个桶下有一条长长的链表，极端情况HashMap 就相当于一个单链表，假如单链表有 n 个元素，遍历的时间复杂度就是 **O(n)**，完全失去了它的优势。
- 数组形成链表，新的节点添加在头节点（会有死循环）
- `resize()`**需要**重新`rehash()`寻址

## JDK1.8

- JDK7与JDK8中HashMap实现的最大区别就是对于冲突的处理方法。JDK 1.8 中引入了红黑树（查找时间复杂度为 **O(logn)**）,用数组+链表+红黑树的结构来优化这个问题。
- 数组形成链表，新的节点添加在尾节点
- `resize()`**不需要**重新rehash()寻址
- 解决了`resize()`时多线程死循环问题，但仍是非线程安全的

# 数据结构

- 数组`Node<K,V>[] table`

- 节点 `Node`

  ```java
  class Node<K,V> {
      final int hash;
      final K key;
      V value;
      Node<K,V> next
  }
  ```

  

# 属性解释

## 属性定义

- `transient Node<K,V>[] table` HashMap的哈希桶数组，非常重要的存储结构，用于存放表示键值对数据的Node元素

- `transient Set<Map.Entry<K,V>> entrySet` HashMap将数据转换成set的另一种存储形式，这个变量主要用于迭代功能

- `transient int size` HashMap中实际存在的Node数量

- `transient int modCount` HashMap的数据被修改的次数，这个变量用于迭代过程中的Fail-Fast机制，其存在的意义在于保证发生了线程安全问题时，能及时的发现（操作前备份的count和当前modCount不相等）并抛出异常终止操

- `final float loadFactor` 也是加载因子，衡量HashMap满的程度，当实际大小超过临界值时，会进行扩容，默认0.75

- `int threshold` 达到临界值，当元素达到临界值会进行扩容2倍，threshold = 加载因子*容量

  

## 默认属性

- `static final int DEFAULT_INITIAL_CAPACITY = 1 << 4` 默认初始大小16
- `static final float DEFAULT_LOAD_FACTOR = 0.75f` 实际存储达到容量的0.75会进行扩容
- `static final int TREEIFY_THRESHOLD = 8` 当某个桶节点大于8，且总数超过64，转化为红黑树，否则扩容
- `static final int UNTREEIFY_THRESHOLD = 6` 当某个桶节点小于6时，会转化为链表，前提它是红黑树
- `static final int MIN_TREEIFY_CAPACITY = 64` 当整个HashMap中的元素数量大于64时，且某个桶节点大于8，也会转化为红黑树结构
- `static final int MAXIMUM_CAPACITY = 1 << 30` 最大容量

# 源码解析：

## 添加元素：

put()过程

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

发生冲突时，链表中新节点jdk1.7中是放在首位，jdk1.8是放在尾节点

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,  //这里onlyIfAbsent表示只有在该key对应原来的value为null的时候才插入，也就是说如果value之前存在了，就不会被新put的元素覆盖。
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;                     //定义变量tab是将要操作的Node数组引用，p表示tab上的某Node节点，n为tab的长度，i为tab的下标。
    // 将成员变量 table 赋值给本地变量 tab，并且将tab的长度赋值给本地变量 n
    // 如果tab为空或者 数组长度为0，进行初始化，调用 resize()方法，并且获取赋值后的数组长度
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
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
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

修改注释版

```java
//这里onlyIfAbsent表示只有在该key对应原来的value为null的时候才插入，也就是说如果value之前存在了，就不会被新put的元素覆盖。
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        /* 声明本地变量 tab，p，n，i（提高性能，effective java），可以先多记两边，防止后面不知道变量怎么来的！ */
        // 定义变量tab是将要操作的Node数组引用，p表示tab上的某Node节点，n为tab的长度，i为tab的下标。
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, i;

        /* 将成员变量 table 赋值给本地变量 tab，并且将tab的长度赋值给本地变量 n */
        tab = table;
        if (tab != null) {
            n = tab.length;
        }

        /* 如果tab为空或者数组长度为0，进行初始化，调用 resize()方法，并且获取赋值后的数组长度 */
        if (tab == null || n = 0) {
            tab = resize();
            n = tab.length;
        }

        /* 根据key的hash值得到当前key在数组中的 位置，赋值给 i */
        i = (n - 1) & hash;
        /* 将i在数组中对应的key值去除赋值给p，所以p代表当前的key */
        p = tab[i];

        /* 判断当前数组中取出来的key是否为空（数组中没有），就new一个新的节点，并且放在这个索引 i的位置 */
        if (p == null) {
            tab[i] = newNode(hash, key, value, null);
            /* 如果不为空，那就表示已经有这样的hash 值已经存在了，可能存在hash冲突 或者 直接替换原来的value */    
        } else {
            /* 声明本地变量 e, k */
            Node<K, V> e;
            K k;

            /* 如果取出来的节点 hash值相等，key也和原来的一样（ == 或者 equals方法为true），直接将这个节点
            * p赋值给刚刚声明的本地变量 e （这个操作很重要，在心中记住）
            * 另外这里还将节点p的key赋值给了本地变量k
            * */
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) {
                e = p;
                
                /* 如果 hash值一样，但不是同一个 key，则表示hash冲突，接着判断这个节点是不是 红黑树的节点
                 * 如果是，则生成一个红黑树的节点然后赋值给本地变量e */
            } else if (p instanceof TreeNode) {
                e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);

                /* 不是红黑树，hash冲突了，这个时候开始扩展链表 */
            } else {
                /* 声明一个本地变量 binCount，开始遍历 p节点后面的链表 */
                for (int binCount = 0; ; ++binCount) {
                    /* 首先将p节点的 next（链表的下一个）赋值给 本地变量e */
                    e = p.next;
                    
                    /* 如果e为空，表示p指向的下一个节点不存在，这个时候直接将 新的 key，value放在链表的最末端 */
                    if (e == null) {
                        p.next = newNode(hash, key, value, null);

                        /* 放入后，还要判断下 这个链表的长度是否已经大于等于红黑树的阈值 （前面分析静态成员变量已经说明），
                        *  一旦大于，就可以变形，调用 treeifyBin方法将原来的链表转化为红黑树 ！
                        * */
                        if (binCount >= TREEIFY_THRESHOLD - 1) { // -1 for 1st
                            treeifyBin(tab, hash);
                        }
                        break;
                    }
                    /* 如果不为空，表示还没有到链表的末端，
                    将 e 赋值给 p（p的下一个节点赋值给p），开启下一次循环 */
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        break;
                    }
                    p = e;
                }
            }
            
            /* e不等于null，则表示 key值相等，替换原来的value即可，
             * 这里需要注意，这里不是表示 hash冲突（再观察下前面的分析），
             * hash冲突链表的扩展已经在最后一个 else完成了！
             * */
            if (e != null) { // existing mapping for key
                V oldValue = e.value;

                if (!onlyIfAbsent || oldValue == null) {
                    e.value = value;
                }

                /* 替换新值后，回调该方法（子类可扩展） */
                afterNodeAccess(e);
                /* 返回原来的 key对应的旧值 */
                return oldValue;
            }
        }

        /* 完成一次 put方法后，加一次 modCount，看前面成员变量分析 */
        ++modCount;

        /* 加入了新节点，把 size 自加，并且 判断是否已经大于要扩容的阈值（观察前面成员变量分析），开始扩容 */
        if (++size > threshold)
            resize();
        
        /* 插入新节点后，回调方法（子类可扩展） */
        afterNodeInsertion(evict);
        
        /* 插入的新节点，直接返回 null即可 */
        return null;
    }
```

## 红黑树化：

treeifyBin()解析

总容量大于64且单链表长度大于等于8，需要对链表进行红黑树转化

```java
/**
 * 1、如果链表大于等于8，数组总长度小于64，则扩容
 * 2、如果链表大于等于8，数组总长度大于64，则红黑树化
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果容量小于64，则扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // 容量大于64且单链表长度大于8，则树化
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

## 计算阈值：

`threshold`

1、默认初始化

```
threshold = newCap * loadFactor
```

2、自己带初始值

找到最近的2的幂次值

```
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

```
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
    this.threshold = tableSizeFor(initialCapacity);
}
```

```
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

```
threshold = newCap * loadFactor
```

## 数组扩容：

### 解析resize()

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值就不再扩充了，就只好随你碰撞去吧
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0)
        newCap = oldThr;
    else {
        // 设置默认容量和阈值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // newThr 为 0 时，按阈值计算公式进行计算 
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 如果旧的桶数组不为空，则遍历桶数组，并将键值对映射到新的桶数组中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 重新映射时，需要对红黑树进行拆分
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 低位链表，节点位置不变
                    Node<K,V> loHead = null, loTail = null;
                    // 高位链表，节点位置+oldCap
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 遍历链表，并将链表节点按原顺序进行分组
                    do {
                        next = e.next;
                        // e.hash & oldCap用来计算hash在新数组的高位是不是为 1
                        // 1、如果为 1，代表在新的数组里面需要原来的位置+旧数组长度
                        // 2、如果为 0.代表在新数组里面位置不变
                        // 在新数组位置不变
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 在新数组的位置：原来位置+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 将分组后的链表映射到新桶中
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
```

### 避免rehash()

**获取节点位置：**

HashMap节点选取数组下标

```
(n-1) & hash
```

**扩容前**：

容量 16 

节点hash值为  0110

```java
(n-1) & 10110

    0  1 1 1 1
&   1  0 1 1 0
------------------
    0  0 1 1 0
```

00110也就是第6位

**扩容后：**

此时我们扩容后容量是 32

```java
(n-1) & 10110

     1  1 1 1 1
&    1  0 1 1 0
------------------------
     1  0 1 1 0
```

10110 = 00110+10000 = 6+15=23

**结果：**

我们可以看出HashMap扩容后，位置是否改变取决于hash的高位是不是为`1`

**判断方法：**`(e.hash & oldCap) == 0`

- 为真：高位是`0`
- 为假：高位为`1`

**计算位置：**

- 如果为`1`
  新位置 =  原位置+原容量
- 如果为`0`
  新位置 =  原位置

# 红黑树

## 定义：

```
1、节点是红色或黑色。
2、根是黑色。
3、所有叶子都是黑色（叶子是NIL节点）。
4、每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5、从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点（简称黑高）。
```

待完善