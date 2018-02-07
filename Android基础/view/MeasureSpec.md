### **目的**

我在一个多月之前就说我准备开始梳理基础的事，好吧，我承认这一个月没我怎么梳理。或者梳理的不多，当我梳理到view的时候，发现需要分成绘制流程以及事件分发进行处理。一开始是想整理一般面试的概要。后来想想，还是看源码慢慢整理把。当我把整个绘制流程的源码看完之后，我突然对一个词比较陌生，就是MeasureSpec。然后就决定整理一波。

### **MeasureSpec概念**

通过源码我们可以知道MeasureSpec是View的内部类，用来控制view的尺寸。也就是view的宽高是由他决定的。MeasureSpec的源码中我们会知道它代表了32位的int值，高2位代表了specmode，低30位代表着specsize。所有博客以及书上都这么说，那么有人知道是为什么吗？我们接着看。

### **MeasureSpec源码**

```
public static class MeasureSpec {
       private static final int MODE_SHIFT = 30;
       private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

       @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
       @Retention(RetentionPolicy.SOURCE)
       public @interface MeasureSpecMode {}

       public static final int UNSPECIFIED = 0 << MODE_SHIFT;

       public static final int EXACTLY     = 1 << MODE_SHIFT;

       public static final int AT_MOST     = 2 << MODE_SHIFT;

       public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size, @MeasureSpecMode int mode) {
           if (sUseBrokenMakeMeasureSpec) {
               return size + mode;
           } else {
               return (size & ~MODE_MASK) | (mode & MODE_MASK);
           }
       }

       public static int makeSafeMeasureSpec(int size, int mode) {
           if (sUseZeroUnspecifiedMeasureSpec && mode == UNSPECIFIED) {
               return 0;
           }
           return makeMeasureSpec(size, mode);
       }

       @MeasureSpecMode
       public static int getMode(int measureSpec) {
           //noinspection ResourceType
           return (measureSpec & MODE_MASK);
       }

       public static int getSize(int measureSpec) {
           return (measureSpec & ~MODE_MASK);
       }

       static int adjust(int measureSpec, int delta) {
           final int mode = getMode(measureSpec);
           int size = getSize(measureSpec);
           if (mode == UNSPECIFIED) {
               // No need to adjust size for UNSPECIFIED mode.
               return makeMeasureSpec(size, UNSPECIFIED);
           }
           size += delta;
           if (size < 0) {
               Log.e(VIEW_LOG_TAG, "MeasureSpec.adjust: new size would be negative! (" + size +
                       ") spec: " + toString(measureSpec) + " delta: " + delta);
               size = 0;
           }
           return makeMeasureSpec(size, mode);
       }

       public static String toString(int measureSpec) {
           int mode = getMode(measureSpec);
           int size = getSize(measureSpec);

           StringBuilder sb = new StringBuilder("MeasureSpec: ");

           if (mode == UNSPECIFIED)
               sb.append("UNSPECIFIED ");
           else if (mode == EXACTLY)
               sb.append("EXACTLY ");
           else if (mode == AT_MOST)
               sb.append("AT_MOST ");
           else
               sb.append(mode).append(" ");

           sb.append(size);
           return sb.toString();
       }
   }
```

代码很简单，也很好理解，但是，我们可以发现它在定义的时候使用的位移运算符。然后我们可以看到在getMode和getSize方法中用到了与运算，位移？与？二进制？原反补码？似乎有点印象但记忆不深啊~这似乎从Android的源码跑到了计算机原理啊。有点跑偏了哈，不过，为了搞明白，我就屁颠屁颠联系了高中的计算机原理老师。问了她几个问题。然后茅塞顿开。下面我们先科普下这些计算机原理的内容。

### 原理科普

在计算机中，数据一般都是用补码形式表示的。至于为什么，我是不想深究了。下面先科普下什么是原反补移码。

- 原码

  原码就是一个数以8位二进制表示，最高位是符号位，0是正数。1是负数。 正数的原反补码一样的。0是特例因为有+0和-0之分。

- 反码

  反码就是在原码的基础上，符号位不变，其他位取反。

- 补码

  补码就是在反码的基础上+1。

- 移码

  不管正负数，只要将其补码的符号位取反即可。

#### 例子

说了一堆废话，来了小栗子，我们以负数举例，因为正数实在没啥好说的。

X=-1.

[X]原=10000001

[X]反=11111110

[X]补=11111111

[X]移=01111111

#### 位移

我们可以看到源码中其实还用到了位移运算符，而位移运算符分成左移和右移。左移分成算术左移，逻辑左移，带进位的小循环左移以及带进位的大循环左移。右移同理。详细介绍下？不不不，我学不动了。分类给你们了。你们有兴趣可以自己看去。

### **源码解析**

#### 定义

MODE_SHIFT：30其实表示进位大小为2的30次方。因为int的大小为32位，所以进位30位就是要使用int的最高位和倒数第二位也就是32和31位做标志位。

MODE_MASK  ：0x3<<MODE_SHIFT。0x代表16进制，10进制也为3（那为什么还要加上0x，不解），二进制为11，11向左进位30，也就是11后面30个0。

> UNSPECIFIED ：同理可得，00后面30个0。
>
> EXACTLY ：同理可得，01后面30个0。
>
> AT_MOST ：同理可得，10后面30个0。

#### 方法

下面我们看一个比较核心的代码：

```
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size, @MeasureSpecMode int mode) {
           if (sUseBrokenMakeMeasureSpec) {
               return size + mode;
           } else {
               return (size & ~MODE_MASK) | (mode & MODE_MASK);
           }
       }
```

这块代码是根据测量大小的size和测量模式的mode，来生成MeasureSpec。我们会发现有一个sUseBrokenMakeMeasureSpec。我们会通过这个值来生成对应的MeasureSpec。我们来跟踪下这个值是干嘛的。

```
sUseBrokenMakeMeasureSpec = targetSdkVersion <= JELLY_BEAN_MR1;
```

可以发现如果是API17以下他才会为真。

我们接着看另外一个类似的方法：

```
public static int makeSafeMeasureSpec(int size, int mode) {
           if (sUseZeroUnspecifiedMeasureSpec && mode == UNSPECIFIED) {
               return 0;
           }
           return makeMeasureSpec(size, mode);
       }
```

这个也就是对版本型号以及他的模式对应的处理。关于mode，这边我不会多讲，下一节会说到。

我们接下来看看我们最常用到的2个方法，getMode和getSize。

```
@MeasureSpecMode
       public static int getMode(int measureSpec) {
           //noinspection ResourceType
           return (measureSpec & MODE_MASK);
       }

       public static int getSize(int measureSpec) {
           return (measureSpec & ~MODE_MASK);
       }
```

我们可以发现，mode=measureSpec & MODE_MASK。也就是说11后面30个0与MeasureSpec进行组合，相当于替换掉MeasureSpec的后30位，保留前2位的mode值。

size=measureSpec & ~MODE_MASK。原理和上面差不多，不过这次对MODE_MASK取反了，也就是说00后面30个1与MeasureSpec进行组合，相当于替换掉MeasureSpec的前2位，保留后30位的size值。

我们还有最后两个方法没有看，toString()我们可以直接忽视了，所以我们接着看另外一个：

```
static int adjust(int measureSpec, int delta) {
           final int mode = getMode(measureSpec);
           int size = getSize(measureSpec);
           if (mode == UNSPECIFIED) {
               // No need to adjust size for UNSPECIFIED mode.
               return makeMeasureSpec(size, UNSPECIFIED);
           }
           size += delta;
           if (size < 0) {
               Log.e(VIEW_LOG_TAG, "MeasureSpec.adjust: new size would be negative! (" + size +
                       ") spec: " + toString(measureSpec) + " delta: " + delta);
               size = 0;
           }
           return makeMeasureSpec(size, mode);
       }
```

这个是对MeasureSpec的一个调整。也就是给MeasureSpec的size追加一个delta值并且通过调用makeMeasureSpec来测量大小。我们可以通过注释发现当mode为UNSPECIFIED时，不需要调整大小。如果调整之后的size小于0，就设为0，然后去调用makeMeasureSpec去测量大小。

### **小结**

关于MeasureSpec的内容基本都介绍了。有人肯定要骂了，你这写的是什么，为什么看不懂？我在文章开头说了，这只是绘制流程中的一个小东西而已。如果不结合整个绘制流程，单独看这个肯定一脸蒙蔽，那么后续的绘制流程教程什么时候出？可能要过段时间了。最少也要等我搬完家之后把~嗯，差不多该说的说完了，晚安了，各位。