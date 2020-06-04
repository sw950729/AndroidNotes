# LinkedList源码分析
### 概述
LinkedList弥补了ArrayList增删较慢的问题，但在查找方面又逊色于ArrayList，所以在使用时需要根据场景灵活选择。对于这两个频繁使用的集合类，掌握它们的源码并正确使用，可以让我们的代码更高效。

- LinkedList 是一个继承于AbstractSequentialList的双向循环链表。它也可以被当作堆栈、队列或双端队列进行操作。
- LinkedList 实现 List 接口，能对它进行队列操作。
- LinkedList 实现 Deque 接口，即能将LinkedList当作双端队列使用。
- LinkedList 实现了Cloneable接口，即覆盖了函数clone()，能克隆。
- LinkedList 实现java.io.Serializable接口，这意味着LinkedList支持序列化，能通过序列化去传输。
- LinkedList 是非同步的。

### LinkedList的基本属性
LinkedList的成员变量只有三个，所以很简单。

```
// 记录当前链表的长度
transient int size = 0;

// 第一个节点
transient Node<E> first;

// 最后一个节点
transient Node<E> last;
```

### 初始化
因为链表没有长度方面的问题，所以也不会涉及到扩容等问题，其构造函数也十分简洁了。

```
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```
一个默认的构造函数，什么都没有做，一个是用其他集合初始化，调用了一下addAll方法。addAll方法我们就不再分析了，它应该是和添加一个元素的方法是一致的。

### 比较重要的几个方法

LinkedList既继承了List，又继承了Deque，那它必然有一堆add、remove、addFirst、addLast等方法。这些方法的含义也相差不大，实现也是类似的，因此LinkedList又提取了新的方法，来简化这些问题。我们看看这些不对外的方法，以及它们是如何与上述函数对应的。

```java
//将一个元素链接到首位
private void linkFirst(E e) {
    //先将原链表存起来
    final Node<E> f = first;
    //定义一个新节点，其next指向原来的first
    final Node<E> newNode = new Node<>(null, e, f);
    //将first指向新建的节点
    first = newNode;
    //原链表为空表
    if (f == null)
        //把last也指向新建的节点，现在first与last都指向了它
        last = newNode;
    else
        //把原链表挂载在新建节点，也就是现在的first之后
        f.prev = newNode;
    size++;
    modCount++;
}

//与linkFirst类似
void linkLast(E e) {
    //...
}

 //在某个非空节点之前添加元素
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    //先把succ节点的前置节点存起来
    final Node<E> pred = succ.prev;
    //新节点插在pred与succ之间
    final Node<E> newNode = new Node<>(pred, e, succ);
    //succ的prev指针移到新节点
    succ.prev = newNode;
    //前置节点为空
    if (pred == null)
        //说明插入到了首位
        first = newNode;
    else
        //把前置节点的next指针也指向新建的节点
        pred.next = newNode;
    size++;
    modCount++;
}

//删除首位的元素，元素必须非空
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

private E unlinkLast(Node<E> l) {
    //...
}

//删除一个指定的节点
E unlink(Node<E> x) {
    //...
}

```
可以看到，LinkedList提供了一系列方法用来插入和删除，但是却没有再实现一个方法来进行查询，因为对链表的查询是比较慢的，所以它是通过另外的方法来实现的，我们看一下：

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

//可以说尽力了
Node<E> node(int index) {
    // assert isElementIndex(index);
    
    //size>>1就是取一半的意思
    //折半，将遍历次数减少一半
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

```
最后，我们看下它如何对应那些继承来的方法：

```java
//引用了node方法，需要遍历
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}

//也可能需要遍历
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
            linkLast(element);
    else
        linkBefore(element, node(index));
}

//也要遍历
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

public E element() {
    return getFirst();
}

public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

public E remove() {
    return removeFirst();
}

public boolean offer(E e) {
    return add(e);
}

public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

//...
```

### 总结
LinkedList非常适合大量数据的插入与删除，但其对处于中间位置的元素，无论是增删还是改查都需要折半遍历，这在数据量大时会十分影响性能。在使用时，尽量不要涉及查询与在中间插入数据，另外如果要遍历，也最好使用foreach，也就是Iterator提供的方式。




