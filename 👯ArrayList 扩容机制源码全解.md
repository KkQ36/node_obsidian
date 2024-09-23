### 01.整体把握
这里首先列出 ArrayList 扩容的几个特点，看完这些特点再去阅读体验会比较好
1）ArrayList 中维护了一个 Object 类型的数组，elementData。
2）当每次创建 ArrayList 对象的时候，如果使用的是无参构造器，则初始的 elementData 的容量为 0，第一次添加的时候则扩容 elementData 的容量为 10，如果需要再次扩容，则扩容为原来的 1.5 倍
3）如果使用的是指定大小的构造器，则初始的 elementData 的容量就是指定的大小，如果需要扩容，也是直接扩容为 elementData 的 1.5 倍。
### 02.无参构造方法
下面来看具体的源码，首先就是 ArrayList 的无参构造方法：
```java
    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
解析：可以看到，它将 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 赋值给了 `elementData`；`ArrayList` 的内部实现就是基于这个 `elementData` 数组，它是 `ArrayList` 存放元素的位置，之后的拿取、扩容之类的操作本质上都是在操纵它。
而 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 是一个常量，来看一下它的定义：
```java
	/**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```
这是一个共享的（`static`）空数组实例，被用作使用**无参构造方法**时，`elementData` 的默认值；
它的作用是与另一个共享的空数组实例来做区分，那另一个共享数组为 `EMPTY_ELEMENTDATA`，它在接下来要讲的有参构造方法中具有很重要的作用
简单来说 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 是标识着通过无参构造形成的空 ArrayList，而 `EMPTY_ELEMENTDATA` 是通过有参构造形成的空 `ArrayList`，它们在后续的扩容策略中会有所不同。
除了这个标志作用以外，它们的作用都是相同的，都是在内存中开辟了一个共享空间用来存放空数组，可以避免过早的为新创建的 ArrayList 实例分配内存，达到节省内存的作用。
### 03.有参构造方法
除了默认的无参构造方法，ArrayList 还提供了两种有参构造方法
```java
public ArrayList(Collection<? extends E> c)

public ArrayList(int initialCapacity)
```
有参构造的第一种方式就是提供一个 `int` 类型的 `initialCapacity`，也就是初始的容量。第二种方法是提供一个集合类，构造方法会将这个集合类中的元素添加到新构建的 `ArrayList` 中。
`public ArrayList(int initialCapacity)`
```java
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
        }
    }
    
    // 无参构造方法
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
如果指定的初始容量大于零，就**直接构造一个容量为初始容量的 Object 数组**，然后将其赋值给 elementData。
(`HashMap` 采取的方式是将初始容量转化为 2 的倍数，作为下次扩容的大小，在第一次添加元素的时候构建的)
否则，当初始的容量等于 0 的时候，就指定其为 `EMPTY_ELEMENTDATA`；这里就能看出与无参构造的区别了，`ArrayList` 会根据是用户指定的容量为 0 或者无参构造方法的懒汉式加载导致的容量为 0 做一个区分；
如果是其他的数字，比如负数，就抛出一个异常，这也是一个常用的 Runtime 异常，`IllegalArgumentException` ，它代表的是方法入参异常。
```java
public class IllegalArgumentException extends RuntimeException{}
```

`public ArrayList(Collection<? extends E> c)`
```java
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a;
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        // replace with empty array.
        elementData = EMPTY_ELEMENTDATA;
    }
}
```
首先通过 `Collction` 接口中定义的 `toArray()` 方法，将集合类转为一个数组，方便后续的处理
然后在 `if` 的条件中去指定 `size`（当前 `ArrayList` 实例中存放元素的个数）为数组的长度，然后判断这个长度是否为 0
如果 `c.getClass()` 为 `ArrayList` 的话，就直接将数组 a 赋值给 `elementData` 
否则，需要再做一次转化，通过 `Arrays.copyOf()` 方法将数组复制为对象数组，这样做的原因有两个
- 一是为了保证类型安全问题
- 二是为了防止那个自定义的集合（为什么一定是自定义的？看看 Collection 接口的规定就知道了）头铁把自己内部的数组的引用传过来，导致操控的是他的数组。
```java
    /**
	 * Returns an array containing all of the elements in this collection.
	 * If this collection makes any guarantees as to what order its elements
	 * are returned by its iterator, this method must return the elements in
	 * the same order.
	 *
	 * <p>The returned array will be "safe" in that no references to it are
	 * maintained by this collection.  (In other words, this method must
	 * allocate a new array even if this collection is backed by an array).
	 * The caller is thus free to modify the returned array.
	 *
	 * <p>This method acts as bridge between array-based and collection-based
	 * APIs.
	 *
	 * @return an array containing all of the elements in this collection
	 */
    Object[] toArray();
```
上面展示的是集合接口 Collection 定义的 `toArray`，它规定了这些内容
- `toArray` 方法返回一个包含集合中所有元素的数组。
- 如果集合对其迭代器返回的元素顺序有保证（例如，按插入顺序或排序顺序），那么 toArray 方法必须返回相同顺序的元素。
- 返回的数组是“安全”的，即集合不会保留对该数组的引用。换句话说，即使集合内部使用数组存储元素，`toArray` 方法也必须分配一个新的数组。这意味着调用者可以自由地修改返回的数组，而不会影响原集合。
如果是 java 中的集合类，也就是严格遵守接口定义的集合类，一般是不会出现问题的；但如果是我们自定义的集合类，既不考虑数组类型，也不考虑返回的是否的引用的情况下，就会出大问题了：
```java
    public static void main(String[] args) {
        Object[] array = getArray();
        array[0] = new Object();
    }

    public static Object[] getArray() {
        return new String[]{"1","2","3"};
    }
```
下面的方法虽然通过类型转化转为了 `Object` 数组，但其本质还是 `String` 数组，也就无法存储 `Object` 类型的元素：
```java
Exception in thread "main" java.lang.ArrayStoreException: java.lang.Object
	at MyTest.main(MyTest.java:13)
```
另外的直接操控源数组的情况更是会导致 `ArrayList` 内部的混乱，后果是可想而知的，这里就不做演示了。
再来看看 `ArrayList` 中的 `toArray` 是如何实现的，我们就知道为什么 `ArrayList` 的 `toArray` 得到的东西可以直接赋值了。
```java
public Object[] toArray() {
	return Arrays.copyOf(elementData, size);
```
自己写的东西就不需要担心上面的两个问题了，直接用！
说回刚刚的流程，如果长度为 0 的话，就将 `elementData` 赋值为 `EMPTY_ELEMENTDATA`，所以这个 `static` 属性其实就是标识有参构造方法形成的空 `elementData` 数组。
```java
else {
	// replace with empty array.
	elementData = EMPTY_ELEMENTDATA;
}
```
### 04.底层扩容机制
终于到了重中之重的扩容机制，直接来看一下源码：
```java
	/**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```
add 方法的代码其实比较简单，首先就是调用了 `ensureCapacityInternal()` 来确保存储空间（`elementData`）足够放下这个新的元素，然后再将元素放入其中：`elementData[size++] = e`

`private void ensureCapacityInternal(int minCapacity)`
```java
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
```
在这个方法中，首先是使用了 `calculateCapacity(elementData, minCapacity)` 方法去计算容量
我们首先考虑使用无参构造和初始容量为 0 的有参数构造，以及传入空集合的有参构造，但无论是哪种方法，最终形成的 `ArrayList` 实例的长度都是 0，所以此时的 `minCapacity` 就是 `1`。
下面展示的是 `calculateCapacity()` 方法，这是一个静态方法：
```java
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
```
首先看第一个 if 语句 `if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)`，如果为 `true` 就代表着，这个 `ArrayList` 是由无参构造方法创建的。
当发现 `ArrayList` 实例是通过无参构造形成的，就会去取 `minCapacity` 和 `DEFAULT_CAPACITY` 中的最大值，而 `DEFAULT_CAPACITY` 的值就是10。
这里不能直接使用 `DEFAULT_CAPACITY` 而需要取最大值，是因为这个方法不止 `add()` 这种只添加一个元素的方法会调用，而 `addAll()` 这类方法同样也会使用 `ensureCapacityInternal()` 来保证容量。
```java
    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;
```
得到了 `minCapacity`，回到上一个方法中去执行 `ensureExplicitCapacity()` 方法，扩容机制就在这个方法之中：
```java
	private void ensureCapacityInternal(int minCapacity) {
		ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
	}

	private void ensureExplicitCapacity(int minCapacity) {
		modCount++;
		// overflow-conscious code
		if (minCapacity - elementData.length > 0)
			grow(minCapacity);
	}
```
首先是对 `modCount` 做自增操作，这个变量记录的是 `ArrayList` 集合改变结构的次数。
然后去判断 `minCapacity - elementData.length > 0` 也就是最小的容量是否大于当前的 `elementData`，如果不能就进入扩容方法 `grow(minCapacity)`。
```java
/**
 * Increases the capacity to ensure that it can hold at least the
 * number of elements specified by the minimum capacity argument.
 *
 * @param minCapacity the desired minimum capacity
 */
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
首先记录下 `oldCapacity`，也就是扩容之前 elementData 的长度
然后执行 `newCapacity = oldCapacity + (oldCapacity >> 1);`，使用了右移运算符，右移运算符其实就可以看作除以 2 的操作，然后再加上原本的 `oldCapacity`，最终就是原本长度的 1.5 倍，但是因为是最后将其转换为了 int，所以其扩容效果是 小于等于 1.5 倍的；
然后将此时的 `minCapacity` 和这个由原本长度拓展 1.5 倍的长度做一个对比，取最大。
后一个 if 语句是为了处理扩容过限的问题，代码比较容易，文章结尾贴给大家看一下。
最终就是调用 `Arrays.copyOf(elementData, newCapacity);` 将原本 `elementData` 中的内容移动到新拓展的长度为 `newCapacity` 的数组中，这就完成了一个完整的扩容
作者给到的注释也很有意思：minCapacity is usually close to size, so this is a win，minCapacity 接近于 size，此时扩容是一个胜利（win）；意思就是通常情况下，minCapacity 会接近当前 ArrayList 的大小（即元素的数量）；由于 `minCapacity` 通常接近 `size`，因此在这种情况下进行扩容操作是非常高效的。
对于无参构造和初始容量为 0 的有参数构造，以及传入空集合的有参构造，可以做如下的总结：
- 此时如果是无参构造，它带进来的 `minCapacity` 就是 10，最终其会被拓展为 10
- 如果是有参构造的话，带进来的 `minCapacity` 其实就是 1，且计算得 `int newCapacity = oldCapacity + (oldCapacity >> 1);` 结果是 0，那最终 `elementData` 会被拓展成 1。
所以说第一次拓展均拓展成 10 其实是不准确的；其他长度的拓展大家顺着流程推导一下就很容易得到了，画成图表就是这样的：

| minCapacity | newCapacity | 最终值 |
| ----------- | ----------- | --- |
| 1           | 0           | 1   |
| 2           | 1.5(1)      | 3   |
| 3           | 无需扩容        | 3   |
| 4           | 4,5(4)      | 4   |
| 5           | 6           | 6   |
| 6           | 无需扩容        | 6   |
| 7           | 9           | 9   |

最后贴上上面提到的 hugeCapacity() 方法的源码：
```java
private static int hugeCapacity(int minCapacity) {
	if (minCapacity < 0) // overflow
		throw new OutOfMemoryError();
	return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}

private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```
 这个方法是用来处理新扩容的长度过大的情况：
- 如果发现 `minCapacity < 0`，就大概率是因为越界导致的了，因为当 `int` 超过 `2^31 - 1` 的时候，就会因为错位变成负数，所以此时抛出 `OutOfMemoryError` 超过内存限制错误。
- 如果还有余地的话，判断此时的 `minCapacity` 是否大于 `MAX_ARRAY_SIZE`，如果不大于就赋值成 `MAX_ARRAY_SIZE`，否则赋值成 `Integer.MAX_VALUE`，也就是 int 的最大值，如果还是不够会在后面因为越界抛出异常。