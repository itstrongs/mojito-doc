# Java 集合篇
<br>
<details><summary>点击展开 Java 集合篇 - 思维导图</summary><img src="http://cdn.liufq.com/Fr52hn9iNMOKpfN0u-HynzwdOhBa"  alt="Java队列" /></details>

## ArrayList 源码分析

```
# 默认初始容量
private static final int DEFAULT_CAPACITY = 10;

# 列表数据容器
transient Object[] elementData;
# 列表长度
private int size;
```

继承 AbstractList，实现 List 接口。

**扩容**：首先计算容量，第一次 add 数据时默认为 10，之后为列表长度 +1；计算容量如果大于数据长度时发生扩容；使用 Arrays.copyOf(T[] original, int newLength) 实现扩容；大小为之前的 1.5 倍。

remote 方法容量不会变回去，会把 remote 位置后的所有数据复制到前一位，最后一位置 null；

通过统计修改次数（modCount）来快速失败并发修改。

## LinkedHashMap 源码分析
Tip：和 LRUCache 实现原理几乎一样，LRU 相关知识都可以套用。

```
// 继承自 HashMap，实现 Map 接口
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {

    /**
     * 定义了一个链表结构存储数据
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

    // 头结点
    transient LinkedHashMap.Entry<K,V> head;
    // 尾结点
    transient LinkedHashMap.Entry<K,V> tail;

    // 排序方式：true 代表按访问顺序，false 代表按插入顺序；默认按插入顺序，LRU 最近最久未使用应该用访问顺序
    final boolean accessOrder;

    /**
     * HashMap 每次访问数据都会调这个方法，LinkedHashMap 作为子类重写了这个方法
     */
    void afterNodeAccess(Node<K,V> e) {
        LinkedHashMap.Entry<K,V> last;
        // 如果是按访问顺序排序且尾结点不是当前访问的节点
        if (accessOrder && (last = tail) != e) {
            // p 是当前访问节点，b 是当前访问节点的上一个节点，a 是当前访问节点的下一个节点
            LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            // 当前访问节点指向空，即移动到尾部
            p.after = null;
            if (b == null)
                head = a;
            else
                // 让当前访问节点的上一个节点指向下一个节点
                b.after = a;
            if (a != null)
                // 让当前访问节点的下一个节点的前驱指针指向上一个节点
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                // 当前节点的前驱指针指向之前的尾结点，即把当前节点放到链表尾端
                p.before = last;
                last.after = p;
            }
            // 指向新的尾结点
            tail = p;
            ++modCount;
        }
    }
}
```

## HashMap 源码分析
### HashMap 的扩容因子为什么是 0.75 ？
因为实际存储的大小越接近容器的大小发生冲突的概率越高，链表或红黑树查询很耗性能，所以在还没满的时候就扩容。

典型的空间换时间，浪费 25% 的内存换取更快的查询速度，具体为什么是 0.75 是不断尝试后得出的结果。

### HashMap 为什么每次扩容都是 2 倍扩容？
一是为了**方便位运算**，只有 2 的倍数才能通过 & 运算快速取模，位运算效率高。

二是为了**降低冲突**，减少数据位置移动，比如 513 对 16 和 32 取模都是 1，只有超过 16 的一半数据位置会变，如果不是 2 倍，所有位置都变了，可能会造成冲突更严重。

### 和 JDK1.7 实现有什么区别？

1. jdk 1.7 用头插法，1.8 以后用尾插法
1. 当冲突过长时链表转化为红黑树；
1. 1.7 先扩容再插入，1.8 先插入再扩容
2. 增加 TREEIFY_THRESHOLD = 8 判断是否需要将链表转化为红黑树的阈值；
3. HashEntry 修改为 Node

## 参考
* [HashMap? ConcurrentHashMap? 相信看完这篇没人能难住你！](https://crossoverjie.top/2018/07/23/java-senior/ConcurrentHashMap/)