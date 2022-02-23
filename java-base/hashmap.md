## HashMap

Java 集合容器主要包含三个接口：List、Map、Set。其中 List 和 Set 实现了 Collection 接口，Map 是单独的接口。

List 是有序的，有序是指插入元素的顺序，其中可以存放重复的值和 null 值。
Set 是无序的，里边的值是各不相同的，所以只能存放一个 null 值。
Map 存放的是 Entry 键值对(key-value)，key 是不能重复的，所以只能存放一个 key 为 null 的键值对。

HashMap 是通过 key 的 hashCode 来进行存放的。
HashMap 的数据结构分为三种，分别是数组、链表、红黑树。

首先对要存入元素的 key 进行 hash 操作，然后对得出来的值进行取模，取模后的值就是要存放数组的下标。
hash 算法很重要，hash 算法做的好的话，要存入的这些元素就会均匀分布在数组中，这个时候的查询时间复杂度是 O(1)；
如果通过 hash 算法对不同的 key 得出来的值是相同的，这个就是哈希碰撞，那么不同的 key 就会存在同一个数组下标的位置，就会想成一个链表，
这个时候的查询时间复杂度是 O(N)；当链表的长度超过 8，就会转换为红黑树，红黑树的查询时间复杂度是 O(log(n))。

### 泊松分布

为什么链表长度超过 8 时才转变为红黑树，这个是根据泊松分布来的。

泊松分布适合于描述单位时间内随机事件发生的次数的概率分布。如某一服务设施在一定时间内受到的服务请求的次数，
汽车站台的候客人数、机器出现的故障数、自然灾害发生的次数、DNA序列的变异数、放射性原子核的衰变数、激光的光子数分布等等。
总结一下就是：**泊松分布就是描述单位时间内，独立事件发生的次数。**

HashMap 的 key 碰撞问题，每次 key 的碰撞，都可以认为是一次独立事件。与上次或下次是否发生 key 碰撞，都无关系。
当 k=9 时，也就是发生的碰撞次数为 9 次时，概率为亿分之三，碰撞的概率已经无限接近为0。

选择超过 8 时才转变为红黑树，也是开发者在时间开销和空间开销上的一个折中取舍。

### HashMap 源码分析

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {

    // HashMap 默认长度 16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

    // HashMap 最大长度
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /*
     * HashMap 默认负载因子 0.75
     * 当 table 中元素被使用了 75% 以上时，就会进行扩容，threshold 就是这个临界值
     * 每次扩容都是原来大小的两倍，也就是说 table 的长度永远是 2 的次幂
     * 扩容是非常耗资源的操作，因为要分配更多的空间
     * 但是扩容之后数组变的更长，所以 hash 算法的结果可以更加的均匀，哈希碰撞的几率更小
     * 根据泊松分布发现在 0.75 时，碰撞的几率是最小的，所以默认的负载因子是 0.75
     * 可以通过构造器控制负载因子，但是不建议
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /*
     * 实际存放元素的数组，transient 修饰符表明这个属性不需要序列化
     * 为什么 table 的长度一定要是 2 的次幂，是为了在取模的时候做优化
     * 判断一个元素进去那个桶（元素存入数组的哪个位置）的时候需要对 key 的 hashCode 取模
     * 取模是一个比较重的操作，需要转换为十进制求余，比较耗性能
     * 当数组的长度总是 2 的 n 次方时，用 hash 值和 (length - 1) 进行位与运算的结果和对 length 取模的结果是一样的
     * 但是位与运算比取模运算的效率要高很多，所以 table 长度总设置为 2 的 n 次方
     */
    transient Node<K,V>[] table;

    // 存放 Map 中所有 Entry，便于进行遍历等操作
    transient Set<Map.Entry<K,V>> entrySet;
    // table 中实际元素的数量
    transient int size;
    // 记录 HashMap 结构发生变化的次数，比如 put 一个 key、发生了扩容等等
    transient int modCount;
    // threshold = size * loadFactor; 表示 HashMap 需要进行扩容的临界值
    int threshold;
    // HashMap 扩容的负载因子，默认值是 0.75
    final float loadFactor;

    /*
     * 求 key 的 hash 值
     */
    static final int hash(Object key) {
        int h;

        /*
         * 将 key 的 hash 值右移 16 位，然后再进行异或操作，主要是为了保留高位和低位的信息
         * 这样就能够表现目标值的特征，从而减少碰撞。
         */
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /*
     * 如果 table 为空或者 length 为 0，将会进行扩容，此时扩容后的大小为默认大小：16，
     * 其余情况都是在元素插入之后再扩容
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
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

    /*
     * 扩容，将旧数组元素转换后填充到新数组中
     * 多线程进行扩容的时候，可能会产生 Node 间的回环，所以是线程不安全的
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
                        // 可以看到扩容时并没有直接取模，而是通过 hash 值和 (length - 1) 进行位与操作
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
    
    /*
     * HashMap 的存储方式是数组 + 链表/红黑树，数组存放的元素就是 Node
     * Node 是 Entry 的实现类
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        // 当前 Node 的 hash 值
        final int hash;
        final K key;
        V value;
        // 指向链表中的下一个节点
        Node<K,V> next;

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

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
}
```