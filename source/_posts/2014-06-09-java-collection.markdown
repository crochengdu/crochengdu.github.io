---
layout: post
title: "关于Java集合的那些事儿"
date: 2014-06-09 10:32:40 +0800
comments: true
categories: Java
---
#一、集合的体系结构
{% img center /images/collection.png 380 320 %}

Collection是集合的基类，List和Set均继承自该类。在使用上，List：有序(元素存入集合的顺序和取出的顺序一致)，元素都有索引，元素可以重复，而Set中元素要保证不能重复。Map集合存储和Collection有着很大不同：
(1)Collection一次存一个元素；Map一次存一对元素。

(2)Collection是单列集合；Map是双列集合。

(3)Map中的存储的一对元素：一个是键，一个是值，键与值之间有对应(映射)关系。

#二、Set和Map的关系，以及Map和List的关系
以HashMap为例，k-v键值对以Entry的方式存入了它的数据成员table，这个table我们可以认为它就是一个Set，因为这个table中的元素是彼此不相等；
反过来说，Set中的元素有一个索引与之对应，所以从这个角度看Set也是Map，而且我们可以从HashSet的实现看到，HashSet其实是通过HashMap来实现的；
Map和List的关系，与Map和Set的关系类似。

#三、ArrayList
(1)使用场景

继承自List，内用Object[]数组存放数据，封装对数组的操作，动态扩容，频繁的增（超出capacity则需要System.arraycopy，指定插入位置必执行System.arraycopy）删（每次删除都要System.arraycopy）影响性能，查询优势。当你需要一个数据结构存储数据，少增删多查询，可以使用ArrayList。

(2)源码解读

ArrayList的add/remove/subList代码解读：
```java
public boolean add(E e) {
/*内部方法，计数并判断是否需要扩容
如果elementData已满，增加其大小至elementData.length + DEFAULT_CAPACITY
		*/
        ensureCapacityInternal(size + 1);
elementData[size++] = e;
	//永远返回true，对于错误情况总是抛出异常
        return true;
    }

删除：
public E remove(int index) {
	//如果传入的index大于elementData存储总数，抛IndexOutOfBoundsException
        rangeCheck(index);
	//修改记录数，为做同步检查
        modCount++;
        E oldValue = elementData(index);
	//需要移动的数量。影响的是index之后的所有数据
        int numMoved = size - index - 1;
	//如果移除的是最后一位，就没有必要重新拷贝。
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,numMoved);
	//把多余的最后一位清空，GC会回收
        elementData[--size] = null;
        return oldValue;
}

SubList（代码比较长，摘取部分）
  //constructor
SubList(AbstractList<E> parent,
                int offset, int fromIndex, int toIndex) {
      this.parent = parent; //将当前arraylist以参数形式传给subList
      this.parentOffset = fromIndex;
      this.offset = offset + fromIndex;
      this.size = toIndex - fromIndex; //记录sublist的size，与parent分开处理
      this.modCount = ArrayList.this.modCount; //防控并发修改
}
public E set(int index, E e) {
      rangeCheck(index); //sublist自身的方法，将index与sublist的size做比较
      checkForComodification(); //只检查了ModCount，没有变更。Set不是结构性变更，所以不用更改modCount。
      E oldValue = ArrayList.this.elementData(offset + index);
      ArrayList.this.elementData[offset + index] = e; //直接修改父类的数据
      return oldValue;
}
//对于sublist的add方法，必须指定index
  public void add(int index, E e) {
        rangeCheckForAdd(index);
        checkForComodification();
        parent.add(parentOffset + index, e);//调用parent对象的add方法完成此操作,因此父类数据也会被修改。
        this.modCount = parent.modCount;//add是结构性的更变，需要修改modCount。
        this.size++;
    }
    
```
初始capacity为10, 视应用情况使用ensureCapacity为数组扩容，减少System.arraycopy的次数。

#四、HashMap
HashMap是线程不安全的
HashTable是线程安全的，但是粒度很大
ConcurrentHashMap, 线程安全，粒度比较小
利用segment缩小锁的范围

(1)、从HashMap的结构可以看出，HashMap的设计初衷是想结合链表和线性表的优势而达到较平衡的查询、添加和删除性能。

(2)、源码分析

Hash方法中，hashcode 的高位参加 hash计算，这样计算出来的hash值可以最大限度的体现对象之间的差异，目的是将对象放在不同的数组位置上。

Index方法中，对hash取模操作，这里的magic是length的长度是2的幂数，length-1后二进制都是01111…1, 从而可以快速取模。
```java
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

static int indexFor(int h, int length) {
    return h & (length-1);
}
```

GET取出的过程
1 判断key是不是为空，为空的话，默认在table[0]这个链表里找;
2 key非空的话，通过对hashcode进行hash，算出最终的hash值;
3 通过indexfor方法找到对应的链表table[i];
4 在链表里，通过比较对象的hash值和key值。如果完全相同的话，返回对象。如果没有找到，返回null。
```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    int hash = hash(key.hashCode());
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
        e != null;
        e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
            return e.value;
    }
    return null;
}

static int indexFor(int h, int length) {
    return h & (length-1);
}
```
(3)、Fail-first机制

HashMap不是线程安全。HashMap中有个成员变量modCount用来计算保障当前的map数据没有其它线程在同时修改。
如果有多个线程同时修改，会抛出ConcurrentModificationException看如下代码，用hashiterator方法来遍历map的时候，
 expectedModCount会被赋予最新的modCount值，调用nextEntry时，检查modCount值相等则不会报错。
调用iterator的remove方法，会让modCount加1，同时expectedModCount也被更新到最新。
所以下次在调用nextEntry时，连个值还是相等的。
```java
			HashIterator() {
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
        }

			final Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

            if ((next = e.next) == null) {
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
            current = e;
            return e;
        }

        public void remove() {
            if (current == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Object k = current.key;
            current = null;
            HashMap.this.removeEntryForKey(k);
            expectedModCount = modCount;
        }
        
```
#五、HashSet
HashSet 不会保证iterable的顺序，LinkedHashSet会保证插入的顺序跟你的iterable的顺序一样
两条规则实现hashCode()在classes：
1.如果object1 和 object2 equals（），那么他们2个应该有相同的hashcode
2.如果object1 和 object2 有一样的hashcode,那么他们不一定会equal.

HashSet源码解读
```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    static final long serialVersionUID = -5024744406713321676L;

    //使用HashMap的key保存HashSet中的所有元素
    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    //定义一个虚拟的Object对象作为HashMap的value
    private static final Object PRESENT = new Object();

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * default initial capacity (16) and load factor (0.75).
     */
     //初始化HashSet，底层会初始化一个HashMap
    public HashSet() {
    map = new HashMap<E,Object>();
    }
//指定初始容量和loadfactor来初始化HashSet，其实就是初始化HashMap
     //loadfactor = 散列表的实际元素数目（n）/ 散列表的容量（m）
    public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<E,Object>(initialCapacity, loadFactor);
    }

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * the specified initial capacity and default load factor (0.75).
     *
     * @param      initialCapacity   the initial capacity of the hash table
     * @throws     IllegalArgumentException if the initial capacity is less
     *             than zero
     */
    public HashSet(int initialCapacity) {
    map = new HashMap<E,Object>(initialCapacity);
    }

    /**
     * Constructs a new, empty linked hash set.  (This package private
     * constructor is only used by LinkedHashSet.) The backing
     * HashMap instance is a LinkedHashMap with the specified initial
     * capacity and the specified load factor.
     *
     * @param      initialCapacity   the initial capacity of the hash map
     * @param      loadFactor        the load factor of the hash map
     * @param      dummy             ignored (distinguishes this
     *             constructor from other int, float constructor.)
     * @throws     IllegalArgumentException if the initial capacity is less
     *             than zero, or if the load factor is nonpositive
     */
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<E,Object>(initialCapacity, loadFactor);
    }

    /**
     * Returns an iterator over the elements in this set.  The elements
     * are returned in no particular order.
     *
     * @return an Iterator over the elements in this set
     * @see ConcurrentModificationException
     */
     //调用Map的keyset来返回迭代器
    public Iterator<E> iterator() {
    return map.keySet().iterator();
    }
/**
     * Adds the specified element to this set if it is not already present.
     * More formally, adds the specified element <tt>e</tt> to this set if
     * this set contains no element <tt>e2</tt> such that
     * <tt>(e==null&nbsp;?&nbsp;e2==null&nbsp;:&nbsp;e.equals(e2))</tt>.
     * If this set already contains the element, the call leaves the set
     * unchanged and returns <tt>false</tt>.
     *
     * @param e element to be added to this set
     * @return <tt>true</tt> if this set did not already contain the specified
     * element
     */
     //将e作为key插入了Hashmap
    public boolean add(E e) {
    return map.put(e, PRESENT)==null;
    }
/**
     * Returns a shallow copy of this <tt>HashSet</tt> instance: the elements
     * themselves are not cloned.
     *
     * @return a shallow copy of this set
     */
     //拷贝通过两步完成:1、通过调用super.clone完成父类的拷贝；2、通过调用map.clone()完成对成员变量map的拷贝。注意map的拷贝也是调用super.clone完成，所以拷贝都是浅拷贝。另外需要注意拷贝过程会抛出CloneNotSupportedException异常，这是一个检查异常。
    public Object clone() {
    try {
        HashSet<E> newSet = (HashSet<E>) super.clone();
        newSet.map = (HashMap<E, Object>) map.clone();
        return newSet;
    } catch (CloneNotSupportedException e) {
        throw new InternalError();
    }
    }
    
```

#六、ArrayList vs LinkedList
ArrayList，实现是一个数组，动态扩容，每次当超出capacity, 就把数组的size扩大，然后把以前的内容copy到新的数组里面去。

ArrayList 如何实现的fail-fast策略，是在每个对于ArrayList结构发生变化的方法里面(比如：add, remove)，都有个modCount这样的变量来记录，然后在iterator里面，构造iterator时候，把modcount 赋值给 exceptCount, 然后check是否相等这2个变量。

LinkedList, 实现是一个双向链表。

         		ArrayList, 		LinkedList
Get(index)		O(1)			O(n)
Add    			O(1)			O(1)
Remove 		O(n - index)          O(1)
Fail-fast的实现，modifyCount != exceptiedCount.
它们2个都是线程不安全的。

#七、好的实践
##使用LinkedHashMap构建LRU缓存
(1) LRU缓存
我们用缓存来存放以前读取的数据，而不是直接丢掉，这样，再次读取的时候，可以直接在缓存里面取，而不用再重新查找一遍，这样系统的反应能力会有很大提高。但是，当我们读取的个数特别大的时候，我们不可能把所有已经读取的数据都放在缓存里，毕竟内存大小是一定的，我们一般把最近常读取的放在缓存里。
LRU缓存利用了这样的一种思想。LRU是Least Recently Used 的缩写，翻译过来就是“最近最少使用”，也就是说，LRU缓存把最近最少使用的数据移除，让给最新读取的数据。而往往最常读取的，也是读取次数最多的，所以，利用LRU缓存，我们能够提高系统的性能。
(2) 为什么是LinkedHashMap
第一，它的数据结构包含一个双向链表，其源代码片段如下：
```java
public class LinkedHashMap<K,V>  extends HashMap<K,V>  implements Map<K,V> {
    /**
     * The head of the doubly linked list.
     */
    private transient Entry<K,V> header;
其中Entry的定义的代码片段如下：
/**                                                                       
 * LinkedHashMap entry.                                                   
 */                                                                       
private static class Entry<K,V> extends HashMap.Entry<K,V> {              
    // These fields comprise the doubly linked list used for iteration.   
    Entry<K,V> before, after;
```
可见这个双向链表就具备了实现LRU的基础，每次访问LinkedHashMap里的元素或者向LinkedHashMap里添加元素时，都会更新这个双向链表，让其header永远指向最少使用的元素，其代码实现可以参考LinkedHashMap的源代码。
第二，LinkedHashMap本身有一个方法用于判断是否需要移除最少使用的元素，但是，原始方法默认不需要移除，该方法实现的源代码如下：
```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
}  
```                 
实际使用中，可以将header指向的最少使用的元素作为该方法的入参，然后重写方法  removeEldestEntry，可以根据自己的条件来决定是否删除该元素。
(3) 实现样例
基于以上的分析，下面给出一个简单的LUR缓存的实现样例，代码如下：
```java
public class LRULinkedHashMap<K, V> extends LinkedHashMap<K, V> {
	/** serialVersionUID */
	private static final long serialVersionUID = -5933045562735378538L;
	/** 最大数据存储容量 */
	private static final int LRU_MAX_CAPACITY = 1024;
	/** 剩余内存空间的阈值，设置的为50M */
	private static final int SPACE_THRESHOLD = 50 * 1024 * 1024; 
/** 存储数据容量 */
	private int capacity;
	public LRULinkedHashMap() {
		super();
	}

	/**
     * 带参数构造方法
     * 
     * @param initialCapacity 容量
     * @param loadFactor 装载因子
     * @param isLRU 是否使用lru算法，true：使用（按方案顺序排序）;false：不使用（按存储顺序排序）
     */
	public LRULinkedHashMap(int initialCapacity, float loadFactor, boolean isLRU) {
		super(initialCapacity, loadFactor, true);
		capacity = LRU_MAX_CAPACITY;
	}

	/**
     * 带参数构造方法
     * 
     * @param initialCapacity 容量
     * @param loadFactor 装载因子
     * @param isLRU 是否使用lru算法，true：使用（按方案顺序排序）;false：不使用（按存储顺序排序）
     * @param lruCapacity  lru存储数据容量
     */
	public LRULinkedHashMap(int initialCapacity, float loadFactor,
			boolean isLRU, int lruCapacity) {
		super(initialCapacity, loadFactor, true);
		this.capacity = lruCapacity;
	}

	@Override
	protected boolean removeEldestEntry(Entry<K, V> eldest) {
		if (capacity > 0 && size() > capacity) {
			return true;
		} else if (Runtime.getRuntime().freeMemory() < SPACE_THRESHOLD) {
			return true;
		}
		return false;
	}
	} 
```