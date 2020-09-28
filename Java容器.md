## Java容器

## 一、概览

容器主要包括 Collection 和 Map 两种，Collection 存储着对象的集合，而 Map 存储着键值对（两个对象）的映射表。

### Collection

 [![img](https://camo.githubusercontent.com/d1efb1abc3173aa2a607316dda79bea560fe333f/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f696d6167652d32303139313230383232303934383038342e706e67)](https://camo.githubusercontent.com/d1efb1abc3173aa2a607316dda79bea560fe333f/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f696d6167652d32303139313230383232303934383038342e706e67) 

### 1. Set

- TreeSet：基于红黑树实现，支持有序性操作，例如根据一个范围查找元素的操作。但是查找效率不如 HashSet，HashSet 查找的时间复杂度为 O(1)，TreeSet 则为 O(logN)。
- HashSet：基于哈希表实现，支持快速查找，但不支持有序性操作。并且失去了元素的插入顺序信息，也就是说使用 Iterator 遍历 HashSet 得到的结果是不确定的。
- LinkedHashSet：具有 HashSet 的查找效率，并且内部使用双向链表维护元素的插入顺序。

### 2. List

- ArrayList：基于动态数组实现，支持随机访问。
- Vector：和 ArrayList 类似，但它是线程安全的。
- LinkedList：基于双向链表实现，只能顺序访问，但是可以快速地在链表中间插入和删除元素。不仅如此，LinkedList 还可以用作栈、队列和双向队列。

### 3. Queue

- LinkedList：可以用它来实现双向队列。
- PriorityQueue：基于堆结构实现，可以用它来实现优先队列。

### Map



## 二、容器中的设计模式

### 迭代器模式

Collection 继承了 Iterable 接口，其中的 iterator() 方法能够产生一个 Iterator 对象，通过这个对象就可以迭代遍历 Collection 中的元素。

从 JDK 1.5 之后可以使用 foreach 方法来遍历实现了 Iterable 接口的聚合对象。

```java
List<String> list = new ArrayList<>();
list.add("a");
list.add("b");
for (String item : list) {
    System.out.println(item);
}
```

### 适配器模式

java.util.Arrays#asList() 可以把数组类型转换为 List 类型。

```java
@SafeVarargs
public static <T> List<T> asList(T... a)
```

应该注意的是 asList() 的参数为泛型的变长参数，不能使用基本类型数组作为参数，只能使用相应的包装类型数组。

```java
Integer[] arr = {1, 2, 3};
List list = Arrays.asList(arr);
```

也可以使用以下方式调用 asList()：

```java
List list = Arrays.asList(1, 2, 3);
```

## 三、源码分析

**源码皆以 jdk1.8 为主**

### HashMap专题

#### 重要常量

```java
	/* 默认初始化容量 */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

	/* 最大容量值，当构造方法中指定的容量值超过该值时使用 */
	static final int MAXIMUM_CAPACITY = 1 << 30;

	/** 默认加载因子。
	 *  加载因子是在判断是否需要扩容时使用，如果当前元素数量>=当前容量*加载因子，则需要扩容	   
	 *  所以，加载因子影响扩容次数，间接影响效率。
	 */
	static final float DEFAULT_LOAD_FACTOR = 0.75f;

	/** 树化阈值。
     *  当链表新增元素后，链表中数量大于该阈值时需要转换成红黑树（详见putVal方法）。
     *  该值必须大于2，并且至少应该是8。
	 */
	static final int TREEIFY_THRESHOLD = 8;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
	/* 容器可能被treeified的最小表容量。(否则，如果容器中有太多节点，就会调整表的大小。)应至少为4 * TREEIFY阈值，以避免大小调整和treeification阈值之间的冲突。 */
	/* 当数组容量还小于64且链表长度达到8个以上时，不会将链表转为红黑树，而是以扩容的形式减少链表长度 */
    static final int MIN_TREEIFY_CAPACITY = 64;
```

#### 重要字段

```java
	/** 在第一次put值时初始化，因此狭义的扩容次数=广义的扩容次数+1。
	 *  容量总是2的幂,table的长度就是容量 
	 */
    transient Node<K,V>[] table;
	/* 存放键值对的Set结构 */
    transient Set<Map.Entry<K,V>> entrySet;

	/* 键值对的数量 */
    transient int size;

	/* HashMap修改次数。 putVal和removeNode等方法会增加该值，迭代中的fail-fast受该值影响 */
    transient int modCount;

	/* 扩容阈值：是容量与加载因子的乘积 */
    int threshold;

    final float loadFactor;
```

#### 存储结构

**数组+链表+红黑树的形式**

##### Node

​	hashMap内包含Node类型的数组 **table**。Node主要用于存储键值对。查看代码可知Node类型包含四个字段：**hash（哈希值）、key（键）、value（值）和 next（链表中下一个节点）。**

​	根据next字段可以知道，在table数组中存放的是链表，主要用于**解决冲突**（拉链法/链地址法）。

​	PS：当链表过长会影响查找效率，因此当链表大于某个阈值时会转换成红黑树。

```java
    /* 省略部分方法 */
	static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        /* 对节点中的 key 和 value计算哈希，并将结果异或。 */    
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        /* 判断两个节点是否相等 */
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

##### TreeNode

```java
    /* 方法代码过多，只贴出字段 */
	static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
    }

	/* 只贴出部分方法的签名和注释 */
	/* Ensures that the given root is the first node of its bin. */
	static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root);
	
	/**
     * Finds the node starting at root p with the given hash and key.
     * The kc argument caches comparableClassFor(key) upon first use
     * comparing keys.
     */
	final TreeNode<K,V> find(int h, Object k, Class<?> kc);
	
    /* Forms tree of the nodes linked from this node. */
	final void treeify(Node<K,V>[] tab);

    /**
     * Returns a list of non-TreeNodes replacing those linked from
     * this node.
     */
    final Node<K,V> untreeify(HashMap<K,V> map);

	/**
     * Removes the given node, that must be present before this call.
     * This is messier than typical red-black deletion code because we
     * cannot swap the contents of an interior node with a leaf
     * successor that is pinned by "next" pointers that are accessible
     * independently during traversal. So instead we swap the tree
     * linkages. If the current tree appears to have too few nodes,
     * the bin is converted back to a plain bin. (The test triggers
     * somewhere between 2 and 6 nodes, depending on tree structure).
     */
	/* 给定的节点必须存在，当树中的节点过少时，会转换成链表的存储结构 */
    final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable);
```

#### 重要方法

##### put 

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
	
	/**
     * 变量说明：tab--table数组; n--初始化后的tab大小; i--hash对应的索引; p--tab[i]的节点;
     *		   key--需要添加的键; value--需要添加的值; 
	 * 过程逻辑：
	 * 1.判断table变量是否为空（即未初始化），为空则初始化。
	 * 2.判断table中，该键值对的key值哈希后对应在该table的位置是否为空？
	 *  2.a. 为空则新建节点并放入table中，执行步骤4;
	 *  2.b. 不为空则判断新插入的 key 是否已存在?
	 *   2.b.1. 如果是，直接执行步骤3。
	 *   2.b.2. 如果不是，则判断节点p是否是TreeNode?
	 *    2.b.2.a. 如果是直接putTreeVal。
	 *    2.b.2.b. 如果不是，使用尾插法在链表中插入节点，并判断是否需要转换成红黑树。
	 * 3.节点不为空则根据onlyIfAbsent 或 oldValue是否为空判断是否修改旧值。
	 * 4.modCount变量+1，判断当前节点数量是否大于阈值，大于则resize(扩容)
	 */
	// @param onlyIfAbsent: if true, don't change existing value
	// @param evict: if false, the table is in creation mode.
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
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

##### resize

```java
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
	/** 
	 * 变量说明： oldXXX--旧数组的有关参数，如容量和阈值。 newXXX--新数组的有关参数。
	 *
	 * 过程逻辑：
	 * 1. 判断oldCap旧容量是否大于 0 ？
	    1.1. 如果oldCap大于0，判断是否大于等于最大容量？
	     1.1.1. 如果大于最大容量，直接返回，不作扩容
	     1.1.2. 小于则判断是否 扩容后的新容量小于最大容量且oldCap大于默认容量
	      1.1.2.1 如果是，更新阈值：newThr = oldThr * 2
	      1.1.2.2 如果不是，跳到步骤2，此时newThr还是0
	    1.2. 如果oldCap小于0且oldThr大于0，则newCap = oldThr，跳到步骤2
	    1.3. 如果oldCap小于0且oldThr小于等于0，认为未初始化，则根据默认值进行初始化，跳到步骤2
	 * 2. 判断newThr新阈值是否等于0（即未赋值）
	    2.1. 如果等于0，则根据newCap新容量、newCap与加载因子乘积是否都小于最大容量赋值，
	    	 大于则赋值最大容量，小于则赋值newCap与加载因子的乘积
	 * 3. 创建新数组newTab，将新数组相关参数赋值到全局变量中，以方便下一次扩容的判断
	 * 4. 将oldTab中的数据转移到newTab中，重新定位索引。此时变个节点的位置会发生改变
	 */
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

###### **分析**	

​	根据上面的过程逻辑可以知道，当我们需要扩容时，需要将旧数组的所有键值对（节点）重新插入到新数组中，因此扩容是相当费时的。在日常开发中，可以大概计算需要存储多少键值对以及增长速率，在构造函数时给定初始容量和加载因子，可以减少扩容次数，增加系统效率。

### HashTable专题

#### 重要常量

```java
	/*
	 * 要分配的数组的最大大小。一些虚拟机在数组中保留一些头字。
	 * 尝试分配更大的数组可能会导致OutOfMemoryError:请求的数组大小超过VM限制。
	 */
	private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

#### 重要字段

基本和hashMap的常量一样，不过table的类型是Entry类型，是node类型的另一种释义。

```java
    private transient Entry<?,?>[] table;

    /* hashtable对象中的节点数，不是table的长度（容量） */
    private transient int count;

    private int threshold;

    private float loadFactor;

    private transient int modCount = 0;
```

#### 存储结构

**数组+链表的形式**

与Node类型一致

```java
	private static class Entry<K,V> implements Map.Entry<K,V> {
        	final int hash;
        	final K key;
      		 V value;
        	Entry<K,V> next;
	}
```

#### 重要方法

##### 构造方法

```java
    
    /* 带参构造函数：指定初始容量和加载因子。 */
    public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }

    /* 带参构造函数：指定初始容量，加载因子使用默认值0.75 */
    public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }

    /* 无参构造函数：初始容量默认值为11 */
    public Hashtable() {
        this(11, 0.75f);
    }

    /* 带参构造函数：先调用另一个带参构造函数，再将传入的map复制 */
    public Hashtable(Map<? extends K, ? extends V> t) {
        this(Math.max(2*t.size(), 11), 0.75f);
        putAll(t);
    }
```

##### put

```java
    /*
     * 使用synchronized修饰，hashTable是线程安全的，是与hashMap的主要区别。
     * 过程逻辑：
     * 1.首先判断传入方法的value是否为空，如果为空抛出异常。
     * 2.根据传入放的参数计算出键值对在table数组中的位置，然后循环判断该key是否已存在
     *  2.1.如果存在则将该key对应的旧值修改为新值，然后返回旧值
     *  2.2.如果不存在，调用addEntry方法，传入键值对、hash值和table索引。
     */
	public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }	
```

##### addEntry

```java
	/*
	 * 该方法是私有的，是真正的添加键值对方法。
	 * 逻辑过程：
	 * 1.修改次数+1，判断当前节点数量是否大于等于阈值（容量*加载因子）
	    1.1.如果是，调用rehash方法扩容（新容量=旧容量×2+1），并重新计算hash值和table索引，继续执行
	 * 2.创建新节点，并使用头插法插入到table数组中
	 * 3.节点数量+1
	 */
    private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry<?,?> tab[] = table;
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```

##### rehash

```java
    /*
     * 逻辑过程：
     * 1.将旧容量变量×2+1后复制给新容量变量，然后判断新容量是否超过规定的最大容量
        1.1.如果旧容量已经是最大容量，直接返回，无法扩容。
        1.2.将新容量设为规定的最大容量 （Integer.MAX_VALUE-8）
     * 2.创建新数组并更新全局变量：修改次数+1、更新阈值、更新table数组
     * 3.需要两重循环（数组+链表），将旧table中的数据转移到新table中，索引需要重新计算
     */
	protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
```



### ConcurrentHashMap专题

​	JDK 1.7 使用分段锁机制来实现并发更新操作，核心类为 Segment，它继承自重入锁 ReentrantLock，并发度与 Segment 数量相等。

​	JDK 1.8 使用了 CAS 操作来支持更高的并发度，在 CAS 操作失败时使用内置锁 synchronized。

#### 重要常量

默认容量、默认加载因子、最大容量、树化阈值、链化阈值与hashMap相同，下不列出。

```java
	/* 可能的最大数组大小(非2的幂)。toArray方法和相关方法所需要的。 */
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

   	/* 最小树化容量 */
    static final int MIN_TREEIFY_CAPACITY = 64;

	static final int MOVED     = -1; // 表示正在转移
	static final int TREEBIN   = -2; // 表示已经转换成树
	static final int RESERVED  = -3; // hash for transient reservations
	static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

```

#### 重要字段

```java
    /* 存放桶的数组，第一次插入值时才初始化。大小总是2的幂 */
    transient volatile Node<K,V>[] table;

    /* 转移时使用的数组 */
    private transient volatile Node<K,V>[] nextTable;

	/* 基本计数器值，主要在没有争用时使用，但也用作表初始化竞争期间的回退。通过CAS更新。 */
    private transient volatile long baseCount;

	/**
	 * 用来控制表初始化和扩容的，默认值为0，当在初始化的时候指定了大小，
	 		这会将这个大小保存在sizeCtl中，大小为数组的0.75。
	 * 当为负的时候，说明表正在初始化或扩张，
     *    -1表示初始化
     *    -(1+n) n:表示活动的扩张线程
     * 初始化之后，保存要调整表大小的下一个元素计数值。
	 */
    private transient volatile int sizeCtl;

    /* 转移数组的索引 */
    private transient volatile int transferIndex;

```

#### 存储结构

与hashMap相同，是**数组+链表+红黑树的形式**

##### Node

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
     }
```

##### TreeNode

```java
	static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
    }
```

##### TreeBin

​	用作树的头结点，只存储root和first节点，不存储节点的key、value值。

```java
	static final class TreeBin<K,V> extends Node<K,V> {
        TreeNode<K,V> root;
        volatile TreeNode<K,V> first;
        volatile Thread waiter;
        volatile int lockState;
        // values for lockState
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock
	}
```

##### ForwardingNode

​	在转移的时候放在头部的节点，是一个空节点

```java
	static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
    }
```

#### 重要方法

##### 原子操作方法

​	在ConcurrentHashMap中使用了unSafe方法，通过直接操作内存的方式来保证并发处理的安全性，使用的是硬件的安全机制。

```java
	/*
     * 用来返回节点数组的指定位置的节点的原子操作
     */
    @SuppressWarnings("unchecked")
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    /*
     * 使用cas原子操作，在指定位置设定值
     */
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    /*
     * 原子操作，在指定位置设定值
     */
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

##### 构造方法

​	在构造方法中，即使指定了初始容量，也不会立刻进行初始化操作，而是在添加元素时才会进行。

```java
    /* 无参构造函数 */
    public ConcurrentHashMap() {
    }

	/**
     *	带指定容量的构造函数，会初始化sizeCtl。
     *	tableSizeFor会以初始容量二进制最高位的全1值作为返回值。
     * （例如：初始容量为4 ，二进制为 100，则返回值为111，再加1则为8）
     */
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }

	/* 当传入一个map时，先指定sizeCtl为默认容量，再添加元素 */
    public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
    }

    /**
     * Creates a new, empty map with an initial table size based on
     * the given number of elements ({@code initialCapacity}) and
     * initial table density ({@code loadFactor}).
     *
     * @param initialCapacity the initial capacity. The implementation
     * performs internal sizing to accommodate this many elements,
     * given the specified load factor.
     * @param loadFactor the load factor (table density) for
     * establishing the initial table size
     * @throws IllegalArgumentException if the initial capacity of
     * elements is negative or the load factor is nonpositive
     *
     * @since 1.6
     */
    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

    /**
     * Creates a new, empty map with an initial table size based on
     * the given number of elements ({@code initialCapacity}), table
     * density ({@code loadFactor}), and number of concurrently
     * updating threads ({@code concurrencyLevel}).
     *
     * @param initialCapacity the initial capacity. The implementation
     * performs internal sizing to accommodate this many elements,
     * given the specified load factor.
     * @param loadFactor the load factor (table density) for
     * establishing the initial table size
     * @param concurrencyLevel the estimated number of concurrently
     * updating threads. The implementation may use this value as
     * a sizing hint.
     * @throws IllegalArgumentException if the initial capacity is
     * negative or the load factor or concurrencyLevel are
     * nonpositive
     */
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

##### initTable

```java
    /**
     * 初始化table数组，根据sizeCtl判断情况。
     * 如果sizeCtl小于0，说明别的数组正在初始化，则让出执行权。
     * 否则通过CAS获取执行权，执行下面步骤。
     * 如果sizeCtl大于0，则初始化一个大小为sizeCtl的数组。
     * 如果sizeCtl等于0，则初始化大小为默认容量的数组。
     * 最后将sizeCtl设置为数组长度的 3/4 
     */
	private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { 
       //SIZECTL：表示当前对象的内存偏移量，sc表示期望值，-1表示要替换的值，设定为-1表示要初始化表 
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

##### put

```java
    /* 真正执行添加元素操作的方法是 putVal，第三个参数true表明值为空才添加，false为覆盖原来的值 */
	public V put(K key, V value) {
        return putVal(key, value, false);
    }

    
	/**
	 * 过程逻辑：
	 * 1.数组未进行初始化则需进行初始化。
	 * 2.已初始化则计算hash值，确定节点在数组的位置。
	 * 3.如果这个位置为空，则使用CAS操作添加节点并直接跳到最后第7步骤
	 * 4.如果这个位置不为空，判断当前数组是否正在扩容（hash值为-1时），是的话则当前线程也去帮助复制
	 * 5.如果这个位置既不为空，也没扩容，则通过加锁进行添加操作：
	    5.1. 判断这个位置的存储结构是红黑树还是链表（根据当前位置头节点的hash值判断）
	 	 5.1.1. 如果大于等于0，说明是链表。
	 	  5.1.1.1. 遍历链表，如果遇到key相等且对于hash值相等，说明是同一个key，
	 	  		则根据参数判断是否覆盖原来的值，否则以尾插法的形式添加元素。
	 *   5.1.2. 如果小于0，说明是红黑树，以数的形式添加元素。
	 * 6.最后在添加完成之后，会判断在该节点处共有多少个节点（注意是添加前的个数），
	 如果达到8个以上了的话，则调用treeifyBin方法来尝试将处的链表转为树，或者扩容数组。
	 * 7. 计数
	 */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        // for循环主要是为了在初始化后和帮助转移数组后能够继续进行添加操作
        for (Node<K,V>[] tab = table;;) { 
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    // 再次取出索引位置的头节点，比较是否相等，不相等说明扩容后位置改变，需要重新计算
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
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
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

##### 扩容方法

​	在put方法的详解中，我们可以看到，在同一个节点的个数超过8个的时候，会调用treeifyBin方法来看看是扩容还是转化为一棵树。同时在每次添加完元素的addCount方法中，也会判断当前数组中的元素是否达到了sizeCtl的量，如果达到了的话，则会进入transfer方法去扩容

```java
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```

​	只有在 **treeifyBin** 方法和 **putAll** 方法才会调用 **tryPresize**

```java
    /**
     * 暂时不看
     *
     *
     *
     */
	private final void tryPresize(int size) {
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```

### HashMap、HashTable与ConcurrentHashMap

三者相同点：

- 以键值对的形式存储数据
- 默认加载因子都是 0.75f （经过研究表明，是空间和时间上的权衡。0.75空间利用率比较高，避免相当多的hash冲突）

### hashMap与hashTable的不同

- 默认容量不同：map是16，table是11
- 扩容数量不同：map是旧容量×2，table是旧容量×2+1
- 继承父类不同：map继承AbstractMap，table继承Dictionary
- 线程安全问题：map是线程不安全的，table是线程安全的（方法用synchronized修饰）
- 空值容忍：map允许key和value有null，table都不允许有null
- 哈希值不同：map重新计算hash值，table直接使用对象的hash值

### ArrayList专题

**extends** AbstractList<E>  **implements** List<E>, RandomAccess, Cloneable, java.io.Serializable

#### 重要常量

```java
    /* 默认容量：10 */
    private static final int DEFAULT_CAPACITY = 10;

	/* 在有参构造函数中用到，表明刚创建的ArrayList对象中的数组是空数组 */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /* 用于默认大小的空实例的共享空数组实例。我们将其与EMPTY_ELEMENTDATA区分开来，
    	以便知道添加第一个元素时需要膨胀多少。 */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

#### 重要字段

```java
    /**
     * 存储ArrayList元素的数组缓冲区。ArrayList的容量是这个数组缓冲区的长度。
     * 任何带有elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA的空ArrayList,
     * 将在添加第一个元素时扩展为DEFAULT_CAPACITY。
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /* 记录数组含有多少个元素 */
    private int size;
```

#### 存储结构

​	由上面 elementData字段可以知道，ArrayList是基于数组实现的，所以支持快速随机访问。RandomAccess 接口标识着该类支持快速随机访问。

#### 重要方法

##### 构造方法

```java
    /* 指定初始容量的构造方法 */
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

    /* 创建空实例的构造方法 */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /* 将指定集合复制到ArrayList实例中的构造方法 */
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

##### add

```java
    /* 不指定索引，在末尾添加元素。在添加前需要确认容量是否足够，且增加修改次数 */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

	/* 在指定的索引添加元素，而索引后的元素将后移。同样需要确认容量且增加修改次数 */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

##### 扩容相关

```java
    /* 通过计算数组容量和所需容量，确认容量是否满足 minCapacity */
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    
    /* 判断数组是否为空数组，如果是则返回默认容量与minCapacity的最大值，否则直接返回minCapacity */
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

	/* 根据计算出的结果，比较所需容量是否大于现有数组容量，如果大于就扩容。 */
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
	/*
     * 新容量为旧容量/2 + 旧容量，即原来的3/2倍；
     * 如果所需容量还是大于新容量，将所需容量赋值给新容量
     * 如果新容量大于最大数组长度，就根据所需容量返回最大数组长度还是整数最大值
     * 将数组中数据复制到新容量的数组中
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

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

### LinkedList专题

