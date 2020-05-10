### LRUCache原理

#### 概念  
LRU（Least Recently Used）最少使用原则
给LRUCache分配缓存空间，当缓存空间存满是，会优先把最近最少使用的缓存对象移除
* Constructor   
传入最大缓存数，一般为当前进程可用容量的1/8。接着内部创建一个`LinkedHaskMap`,使用的对象引用都存在这里

```java
public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

* 示例

```java
int maxMemory = (int) (Runtime.getRuntime().totalMemory()/1024);
        int cacheSize = maxMemory/8;
        mMemoryCache = new LruCache<String,Bitmap>(cacheSize){
            //计算缓存对象所用单位
            @Override
            protected int sizeOf(String key, Bitmap value) {
                return value.getRowBytes()*value.getHeight()/1024;
            }
        };
```

#### 实现原理

* 使用LinkedHashMap

    LinkedHashMap是由数组+双向链表的数据结构来实现的。其中双向链表的结构实现了访问顺序和插入顺序两种方法，使得LinkedHashMap中的<key,value>对按照一定顺序排列起来。

```java
    /**
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity 初始容量
     * @param  loadFactor      the load factor 负载系数
     * @param  accessOrder     the ordering mode - 顺序模式<tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order true访问顺序 false插入顺序
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                            float loadFactor,
                            boolean accessOrder) {
            super(initialCapacity, loadFactor);
            this.accessOrder = accessOrder;
    }
```

* put方法

```java
public final V put(K key, V value) {
         //不可为空，否则抛出异常
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }
        V previous;
        synchronized (this) {
            //插入的缓存对象值加1
            putCount++;
            //增加已有缓存的大小
            size += safeSizeOf(key, value);
           //向map中加入缓存对象
            previous = map.put(key, value);
            //如果已有缓存对象，则缓存大小恢复到之前
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }
        //entryRemoved()是个空方法，可以自行实现
        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }
        //调整缓存大小(关键方法)
        trimToSize(maxSize);
        return previous;
    }
```

添加缓存对象后，更改size大小,最后在调用trimToSize判断是否已经存满，如果满了就会执行 最近少用原则。

* trimToSize()

```java
public void trimToSize(int maxSize) {
        //死循环
        while (true) {
            K key;
            V value;
            synchronized (this) {
                //如果map为空并且缓存size不等于0或者缓存size小于0，抛出异常
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }
                //如果缓存大小size小于最大缓存，或者map为空，不需要再删除缓存对象，跳出循环
                if (size <= maxSize || map.isEmpty()) {
                    break;
                }
                //迭代器获取第一个对象，即队尾的元素，近期最少访问的元素
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                //删除该对象，并更新缓存大小
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }
            entryRemoved(true, key, value, null);
        }
    }
```

看到这里是个死循环，直到size <=maxSize或者缓存中没有对象才跳出循环。

* get方法
```java
public final V get(K key) {
        //key为空抛出异常
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            //获取对应的缓存对象
            //get()方法会实现将访问的元素更新到队列头部的功能
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }
```

其中LinkedHashMap的get()方法如下
先判断是否是顺序访问模式，接着调用afterNodeAccess()。然后会把链表重新排列，把刚才访问的元素移至末尾。

```java
public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
}
```

afterNodeAccess()方法如下：

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMapEntry<K,V> p = (LinkedHashMapEntry<K,V>)e, 
                                 b = p.before, 
                                 a = p.after;
            p.after = null;
            //该元素的上一个为null，说明该元素在队列头，将该元素的下一个赋给head。
            //该元素的上一个不为null，将该元素的上一个元素的下一个赋值为该元素的下一个。
            if (b == null)
                head = a;
            else
                b.after = a;
            //当该元素的下一个不为null，将该元素的下一个元素的上一个元素赋值为 该元素的上一个元素
            //当该元素的下一个为null，把该元素的上一个 赋值给 last
            if (a != null)
                a.before = b;
            else
                last = b;
            //当last为null，说明该元素的下一个元素为null，并且该元素的上一个也为null。head 赋值为 该元素
            //当你last不为null，有两种情况。第一种：last 是 之前的tail，第二种 last是该元素的上一个。接着把该元素放在末尾。
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```
#### 总结
1. 优先把最近最少使用的缓存对象移除
2. 使用LinkedHashMap,
