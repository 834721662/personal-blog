[TOC]


####JDK9 HashMap 源码阅读



HashMap由链表+数组组成，它的底层结构是一个数组，而数组的元素是一个单向链表。默认是长度为16位的数组，每个数组储存的元素代表的是每一个链表的头结点。
我们平时常用的MD5，SSL等都属于Hash算法，通过Key进行Hash的计算，就可以获取Key对应的HashCode。
好的Hash算法可以计算出几乎出独一无二的HashCode，如果出现了重复的hashCode，就称作碰撞，就算是MD5这样优秀的算法也会发生碰撞，即两个不同的key也有可能生成相同的MD5。
正常情况下，我们通过hash算法，往HashMap的数组中插入元素。
如果发生了碰撞事件，那么意味这数组的一个位置要插入两个或者多个元素，这个时候数组上面挂的链表起作用了，链表会将数组某个节点上多出的元素按照尾插法(jdk1.7及以前为头差法)的方式添加。

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    /** 用于序列化的版本id **/
    private static final long serialVersionUID = 362498820763181265L;

    /** 初始化的大小 16 */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /** 最大容量（必须是2的幂且小于2的30次方，传入容量过大将被这个值替换） */
	static final int MAXIMUM_CAPACITY = 1 << 30;

	/** 默认的负载因子 0.75 */
	static final float DEFAULT_LOAD_FACTOR = 0.75f;

	/**  这里表示一个桶的大小: 可以理解为 当一个桶(bucket)大小超过8时，会使得 bucket 由 链表 转化为 红黑树  */
	static final int TREEIFY_THRESHOLD = 8;

	/** 与上一个相反: 当一棵树的大小 小于六的时候就会由树转化为 链表 */
	static final int UNTREEIFY_THRESHOLD = 6;

	/** 最小的树的容量， 这里应该是是 四倍的TREEIFY_THRESHOLD，避免进行扩容、树形化选择的冲突   */
	static final int MIN_TREEIFY_CAPACITY = 64;


	/** 计算某个 key 的hash值 */
	static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }


    /** 如果实现了Comparable，返回x的实际类型，也就是Class<C>，否则返回null. */
    /** 例子:public class AppVersion implements Comparable<AppVersion> */
    static Class<?> comparableClassFor(Object x) {
        if (x instanceof Comparable) {
            Class<?> c; Type[] ts, as; ParameterizedType p;
            if ((c = x.getClass()) == String.class) // bypass checks
                return c;
            if ((ts = c.getGenericInterfaces()) != null) {
                for (Type t : ts) {
                    if ((t instanceof ParameterizedType) &&
                        ((p = (ParameterizedType) t).getRawType() ==
                         Comparable.class) &&
                        (as = p.getActualTypeArguments()) != null &&
                        as.length == 1 && as[0] == c) // type arg is c
                        return c;
                }
            }
        }
        return null;
    }

    /**
     * Returns k.compareTo(x) if x matches kc (k's screened comparable
     * class), else 0.
     */
    @SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
    static int compareComparables(Class<?> kc, Object k, Object x) {
        return (x == null || x.getClass() != kc ? 0 :
                ((Comparable)k).compareTo(x));
    }



    /**
     * n与n进行或操作再复制给n，接着无符号右移(空白补零)，最后得到一个 power of two size(2的幂数，比cap大)
     */
    static final int tableSizeFor(int cap) {
    	// cap的二进制里低位全部转成1
        // 解释一个:n |= n >>> 1 ==> n = n>>>1 | n
        // 假设n= 0001 xxxx xxxx xxxx
        // 计算:0001 xxxx xxxx xxxx | 0000 1xxx xxxx xxxx => 0001 1xxx xxxx xxxx
        // 此时最高位就是两个连续的1,然后操作n |= n >>> 2,那么就变成 0001 111x xxxx xxxx
        // 所以变1的节奏个数是:1 2 4 8 16 相加 31 刚好足够把32位的一个值低位全部变成1.
        // 只不过cap最大也就是2的30次
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

    /**
     * hashMap的核心数据结构，这里用transient修饰(临时，不参与序列化) 
     * 这里明确表示 在第一次使用这个数组的时候需要进行初始化
     *
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;

    /**
     *
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /** 记录当前map的大小 */
    transient int size;

    /** 记录当前hashMap被改变的次数 */
    transient int modCount;

    /** 调整下一个大小值(容量*负载因子)。 */
    int threshold;

    /** 负载因子 */
    final float loadFactor;

    /** 
     * 构造方法
     * 可以看到这里传入了一个初始大小，和负载因子
     * 还有另外三个构造，很简单，就不聊了(就是一些带有默认值的构造)
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
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }

    /**
     * Implements Map.putAll and Map constructor
     * Map.putAll也走这个位置 evict = false -> 表示构造函数调用
     * @param m the map
     * @param evict false when initially constructing this map, else
     * true (relayed to method afterNodeInsertion).
     */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
        	//table 是 上面那个 Node<K, V> 数组
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold) 
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

    /**
     * Returns the number of key-value mappings in this map.
     *
     * @return the number of key-value mappings in this map
     */
    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }


    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     * get 方法实际上是功过 getNode来进行搜索
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Node<K,V> e;
        //返回那个对应的Object，如果原来存的东西是null，也返回了
        //如果返回null 可能里面存的是null，也可能map当中不存在那个key,value
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods
     * get方法也走这里
     * 通过key的hash值来进行节点的搜索
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab;
        Node<K,V> first, e; 
        int n; 
        K k;
        if ((tab = table) != null && (n = tab.length) > 0 && 
        	//注意:根据输入的hash值，可以直接计算出对应的下标（n - 1）& hash，缩小查询范围，如果存在结果，则必定在数组的这个位置上。
        	//这里的n是表的长度， 长度 - 1 再与 hash值进行一个与操作
        	(first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
            	//这里判断 这个节点是 红黑树还是链表，如果是红黑树，递归搜索
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //如果是一个链表，进行循环
                do {
                	//注意: 这里判断的条件是: hash值相同，并且key符合equals
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

    /**
     * 这里可以看到实际上很多地方都掉用了 getNode 方法
     * @param   key   The key whose presence in this map is to be tested
     * @return <tt>true</tt> if this map contains a mapping for the specified
     * key.
     */
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }

    /**
	 * 如果先前的映射包含键的映射，则替换旧值。
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods
     * onlyIfAbsent: 为false的时候，替换旧值
     * evict:为false的时候表示在创建模式(构造)
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    	////tab[]为数组，p是每个桶
        Node<K,V>[] tab; 
        Node<K,V> p; 
        int n, i;
        //table为空，则调用resize()函数创建一个
        //注意：其实在这里的时候才初始化整个数组
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //计算元素所要储存的位置index,并对null做出处理
        if ((p = tab[i = (n - 1) & hash]) == null)
        	//注意:，如果tab[i]==null，说明这个位置上没有元素，这个时候就创建一个新的Node元素
            tab[i] = newNode(hash, key, value, null);
        else {
        	//运行到这里，就说明这个要添加的位置上面已经有元素了，也就是发生了碰撞。这个时候就要具体情况分类讨论：
        	// 1.key值相同，直接覆盖 2.链表已经超过了8位，变成了红黑树 3.链表是正常的链表
            Node<K,V> e; 
            K k;
            //如果hash相同 key符合equals，就覆盖这个元素
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //判断是否是红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                	//链表的下一个节点为空的情况，就新生成一个，执行插入
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果链表的长度大于8，则转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果节点key存在，则覆盖原来位置的key，同时将原来位置的元素，沿着链表向后移一位
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
        //如果容量太大，扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;


	/**
     * 扩容
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;//用于保留之前的数组
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//旧数组的容量大小
        int oldThr = threshold;//旧的临界值
        int newCap, newThr = 0;
        if (oldCap > 0) {
        	//如果容量大小大于最大容量，临界值提高到正无穷
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //newCap 等于旧容量的两倍 要小于最大容量 2^30 并且原来的容量要大于初始长度 2^4  
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold 临界值
        }
        //初始化,新数组的初始容量设置为老数组扩容的临界值
        else if (oldThr > 0) 
            newCap = oldThr;
        else {               //这里是初始化整个数组，默认大小16 扩容因子 0.75 阈值 12
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //如果newThr == 0，说明为上面 else if (oldThr > 0)为true
        if (newThr == 0) { 
        	//ft为临时变量，用于判断阈值的合法性
            float ft = (float)newCap * loadFactor;
            //计算新的阈值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //调整
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];   //改变table全局变量为，扩容后的newTable
        table = newTab;
        if (oldTab != null) {
        	//遍历整个旧数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果e节点没冲突，把e放到新数组的某个位置
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e; //取模操作
                    //如果是红黑树的情况
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                    	//这里是一个链表的情况
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //这里表示位置不变，如果冲突了就在lotail后面补
                            //这里判断的依据解释一下：HanshMap中元素存入数组的下表运算为**index = hash & (n - 1) ** n为数组的长度
                            //那么换成新的数组之后，扩容一倍，在二进制当中(16)都相当于原先的15，左移一位后(后补1)，再和自己做一个异或操作
                            //这个时候，针对两个不同的hashKey，末尾分别为: 1 1101和0 1101的hash值进行与操作，就会产生两个不同的结果
                            //因此，原来由 hash & (n - 1)的值产生的冲突的key，再经过(e.hash & oldCap)可以被分到不同的桶当中去
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //位置发生了改变，如果发生冲突就在hitail后面补
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
                            //如果(e.hash & oldCap) == 0成立，索引位置不变还是j
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            //(e.hash & oldCap) != 0 ==> newTab[j + oldCap] = hiHead = e = oldTab[j]
                            //原数组[j]位置上的桶移到了新数组[j+原数组长度]的位置上
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

    /**
     *
     * 将链表转成树结构,如果table还很小,就用resize操作.
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // table 太小 直接resize一下扩容解决
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                // 先把Node链表转成TreeNode链表
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;// 头
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                // 转变操作
                hd.treeify(tab);
        }
    }

    public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }

    /** 实际调用的是 removeNode */
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }


    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; 
        Node<K,V> p; 
        int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
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


}
```

这里是中间结构，最基础的哈希节点,貌似 treeNode 和 LinkedHashMap 也是用的它
之前版本的叫法是: Entry
```java
//Node 是单向链表
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        //指向下一个节点
        Node<K,V> next;

        //构造函数
        //hash: 哈希值，k: key值， v: value值， next: 下一个节点
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        //判断两个node是否相等
        //当且仅当 key & value 都相等的情况下 返回true
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

