# ArrayList源码分析
### 概述
ArrayList是日常开发中使用最频繁的集合类。首先这边简单介绍一下ArrayList：  
- ArrayList是通过动态数组去实现的
- ArrayList并不是线程安全的
- 实现了RandomAccess接口，可以通过下标序号进行快速访问
- 实现了Cloneable接口，能被克隆
- 实现了Serializable接口，支持序列化

### ArrayList基本的属性
ArrayList属性主要就是当前数组长度size，以及存放数组的对象elementData数组，除此之外还有一个经常用到的属性就是从AbstractList继承过来的modCount属性，代表ArrayList集合的修改次数。  

```
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable {
	// 序列化id
	private static final long serialVersionUID = 8683452581122892189L;
	// 默认初始的容量
	private static final int DEFAULT_CAPACITY = 10;
	// 一个空对象
	private static final Object[] EMPTY_ELEMENTDATA ={};
	// 一个空对象，如果使用默认构造函数创建，则默认对象内容默认是该值
	private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA ={};
	// 当前数据对象存放地方，当前对象不参与序列化
	transient Object[] elementData;
	// 当前数组长度
	private int size;
	// 数组最大长度
	private static final int MAX_ARRAY_SIZE =  Integer.MAX_VALUE - 8;
 
	// 省略方法。。
	}
```

### ArrayList的初始化
说到初始化，就肯定会提到构造函数，还有初始大小，那么ArrayList的初始大小是多少？我们首先看看无参的构造函数：

```
public ArrayList() {
		this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
	}
```
注意，当我们创建的时候，elementData这个数组的数组长度为0，但当我们添加数据的时候他的默认长度才会变成10。

当然除了无参的构造方法，还有有参的构造。我们可以在创建的时候直接指定一个大小，具体代码如下：

```
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```
传入的数如果大于等于0，那么就使用传入的数据作为初始化的大小，如果小于0则会抛出异常。

除了上述两种初始化，其里面还有带Collection的构造函数。代码如下：

```
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
将collection对象转换成数组，然后将数组的地址给到elementData。  
更新size的值如果为0就传空，否则通过arrays.copyof将collection的内容考本到elementData中去。  

### 比较重要的几个方法
#### Add
ArrayList已经是具体的实现类了，所以其实现了list接口中所有的定义方法。首先前面说到ArrayList的默认大小为0，只有第一次添加数据的时候才会设为10。下面我们看看add的方法。

```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
首先我们看到elementData如果为null的时候，他会讲初始大小设为10。然后在ensureExplicitCapacity方法中可以看出，如果需要扩容了，就调用了grow方法进行扩容。grow方法会将数组扩容成原来的1.5倍。最后确保新增的数据有地方存储之后，则将新元素添加到位于size的位置上。最后返回一个添加成功的boolean值。

当然了，添加处理这个还有一个方法就是在指定位置添加内容，逻辑和上面差不多，这边就不介绍了。

#### Remove

前面我们说到了add方法，与之对应的就是remove了。这边不仅可以通过坐标去移除，也可以通过object去移除，具体代码如下：

```
   public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

```
先看看根据索引去移除某个对象。我们需要判断索引是否越界，如果越界了则抛出异常。否则将指定的位置的元素保存在oldValue中。然后讲指定位置的后面元素都前移动一位。然后讲最后一位置空。最后返回oldValue。（注意，这个方法不会缩减数组的长度，只是将最后一位置空而已...）

下面是根据对象进行remove。它会循环编译所有对象，找到你要移除对象的索引位置。然后调用fastRemove进行移除，具体的逻辑和remove是一样的。但是为什么remove方法不去调用fastRemove方法....  

#### contains
然后，我们在看看contains方法，这是判断当前list中是否包涵某个内容。内部其实是通过equals去判断的：

```
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

#### 其他
其实ArrayList还有很多方法，最后我们只介绍2个方法。前面我们介绍了将数组元素置为空后，它的数组大小没变。那么通过clear方法。他的数组大小会变吗？我们继续看代码：

```
 public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```
很明显，这里也只是讲值设为null，然后等待GC去回收掉。那么什么方法是去讲空的数据去除改变数组大小呢。是trimToSize。我们具体看看代码：

```
  public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```
将elementData中空余的空间（包括null值）去除，例如：数组长度为10，其中只有前三个元素有值，其他为空，那么调用该方法之后，数组的长度变为3。


### 总结
总体而言，ArrayList还是和数组一样，更适合于数据随机访问，而不太适合于大量的插入与删除，如果一定要进行插入操作，要使用以下三种方式：

- 使用ArrayList(intinitialCapacity)这个有参构造，在创建时就声明一个较大的大小。

- 使用ensureCapacity(int minCapacity)，在插入前先扩容。

- 使用LinkedList，这个无可厚非哈，我们很快就会介绍这个适合于增删的集合类。

