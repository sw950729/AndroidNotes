# Hashmap源码分析
### 概述
hashmap应该是集合中较复杂的一个类。最早出现在1.2中，在1.8中加入了红黑树，所以1.8的改动很大。hashmap允许null值和null键，底层是通过散列算法实现的。并且hashmap不是线程安全的。  

### 源码分析
#### 成员变量  

```java
  //默认的初始化数组大小为16
  static final int DEFAULT_INITIAL_CAPACITY = 1 << 4
  //最大存储容量
  static final int MAXIMUM_CAPACITY = 1 << 30
  //默认扩容因子
  static final float DEFAULT_LOAD_FACTOR = 0.75f;
  //这是jdk1.8新加的，这是链表的最大长度，当大于这个长度时，就会将链表转成红黑树
  static final int TREEIFY_THRESHOLD = 8;

  //table就是存储Node 的数组，就是hash表中的桶位
  transient Node<K,V>[] table;
  //实际存储的数量，则HashMap的size()方法，实际返回的就是这个值，isEmpty()也是判断该值是否为0
  transient int size;
  
  transient int modCount;

  //扩容的门限值，当大于这个值时，table数组要进行扩容，一般等于（cap*loadFactor）
  int threshold;
  //装载因子
  final float loadFactor;
```
我记得hashmap的初始大小在android中是4，java中是16...不知道啥时候改了....

#### 构造方法
hashmap的构造方法有4个，具体代码如下：

```java
/** 构造方法 1 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/** 构造方法 2 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/** 构造方法 3 */
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

/** 构造方法 4 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

```
我们可以在初始化的时候传入初始大小以及扩容因子，如果不传则为默认值。当然，我们还可以把已有map直接传进去。我们可以看到在第三个构造方法中给扩展的限制进行了重新的定义，因为可能我们传的大小不是2的幂数。现在他是如何重新定义的：

```java

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
这段代码的意思就是把初始大小的后面所有数置为1，然后加一，也就是找到最近的2的次幂。

### 添加
现在我们看看hashmap是如何添加一个元素的。  

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
这边主要是通过putVal进行添加的，这边通过hash函数将key与其高16位异或。具体代码如下：

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

下面我们具体看看putVal这个方法：

```java

 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //table 没有初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            //可以看到table的初始化放在了这里，是通过resize来做的
            n = (tab = resize()).length;

        //这里就是HASH算法了，用来定位桶位的方式，可以看到是采用容量-1与键hash值进行与运算
        //n-1,的原因就是n一定是一个2的整数幂，而(n - 1) & hash其实质就是(n-1)%hash,但是取余运算的效率明显不如位运算与
        //并且(n - 1) &hash也能保证散列均匀，不会产生只有偶数位有值的现象
        if ((p = tab[i = (n - 1) & hash]) == null)
            //当这里是空桶位时，就直接构造新的Node节点，将其放入桶位中
            //newNode()方法，就是对new Node(,,,)的包装
    //同时也可以看到Node中的hash值就是重新计算的hash(key)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //键hash值相等，键相等时，这里就是发现该键已经存在于Map中
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //当该桶位是红黑树结构时，则应该按照红黑树方式插入
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //当该桶位为链表结构时，进行链表的插入操作，但是当链表长度大于TREEIFY_THRESHOLD - 1，就要将链表转换成红黑树
            else {
                //这里binCount记录链表的长度
                for (int binCount = 0; ; ++binCount) {
              //找到链表尾端
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
        //记录改变次数
        ++modCount;
        //当达到扩容上限时，进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
上面注释的已经很清楚了，这边说一下，他会通过散列算法找到对应的位置，如果是空就直接扔进去，如果是非空就判断它是链表还是红黑树，按照对应的数据结构进行插入。

如果链表太长，会进行树化操作，下面看看如何进行树化的：

```java

final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 桶数组容量小于 MIN_TREEIFY_CAPACITY，优先进行扩容而不是树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // hd 为头节点（head），tl 为尾节点（tail）
        TreeNode<K,V> hd = null, tl = null;
        do {
            // 将普通节点替换成树形节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);  // 将普通链表转成由树形节点链表
        if ((tab[index] = hd) != null)
            // 将树形链表转换成红黑树
            hd.treeify(tab);
    }
}

TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
```
这边代码不多，上面注释已经说的比较清楚了。  

无论在put还是树化中都用到了resize方法，也就是扩容，下面看看resize是如何处理扩容的。

```java

    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
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
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
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
```
从上述代码，我们可以看出，如果超过了最大值则不需要扩容了，否则将大小以及阈值扩充到原来的两倍。扩充完成后，重新对数据排列。对红黑树以及链表进行拆分。然后按照原顺序进行排列。  

#### 获取
获取其实很简单，就是先定位键所在的桶的位置，然后在对链表或者红黑树进行查找。具体代码如下：

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 1. 定位键值对所在桶的位置
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 2. 如果 first 是 TreeNode 类型，则调用黑红树查找方法
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                
            // 2. 对链表进行查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### 删除
删除操作其实也是很简单的，第一步，定位bucket的位置，第二步，遍历找到与key相同的节点，第三步，删除。具体代码如下：

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        // 1. 定位桶位置
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 如果键的值与链表第一个节点相等，则将 node 指向该节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {  
            // 如果是 TreeNode 类型，调用红黑树的查找逻辑定位待删除节点
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 2. 遍历链表，找到待删除节点
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        
        // 3. 删除节点，并修复链表或红黑树
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
### 总结
合理的使用HashMap能够在增删改查等方面都有很好的表现。在使用时我们需要注意以下几点：

- 设计的key对象一定要实现hashCode方法，并尽可能保证均匀少重复。

- 由于树化过程会依次通过hash值、比较值和对象的hash值进行排序，所以key还可以实现Comparable，以方便树化时进行比较。

- 如果可以预先估计数量级，可以指定initial capacity，以减少rehash的过程。

- 虽然HashMap引入了红黑树，但它的使用是很少的，如果大量出现红黑树，说明数据本身设计的不合理，我们应该从数据源寻找优化方案。


