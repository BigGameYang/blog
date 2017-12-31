# LruCache详解(一) —— LruCache实现

## 概述 

Android SDK 与 supportV4 包中都提供了 LruCache 的实现，该实现本质为 lru-1 算法实现, 关于 lru-1 请看[LruCache详解(一)](LruCache详解(一).md)。

LruCache 内部维护了一个 LinkedHashMap, HashMap 的特性让 LruCache 具有快速访问的能力（具体 HashMap 分析查看 [HashMap详解](HashMap详解.md)），而 LinkedHashMap 的构造函数中有个 accessOrder 参数可以决定 LinkedHashMap 内部的顺序链表可以按照访问时间排序（具体 LinkedHashMap 分析查看 [LinkedHashMap详解](LinkedHashMap详解.md)），该排序正好满足 LRU 最近访问保留机制。


## 实现

LruCache 需要两个泛型参数分别对应 Key 和 Value 类型,内部维护的 LinkedHashMap 沿用该两个泛型参数。

```java
    public class LruCache<K, V> 
```

LruCache 内部属性

```java

    /**
     * 内部所维护的 LinkedHashMap 对象
     */
    private final LinkedHashMap<K, V> map;

    /**
     * 缓存值的大小，该值不一定代表元素个数，而是所有存在元素的大小之和
     */
    private int size;

    /**
     * 最大缓存大小，通过该值判断是否需要淘汰缓存
     */
    private int maxSize;



    // 以下属性都只为统计所用

    /**
     * put 缓存累计次数
     */
    private int putCount;

    /**
     * 访问某个不存在的 key 时通过 create 方法创建的缓存累计次数
     */
    private int createCount;

    /**
     * 淘汰了缓存的累计次数
     */
    private int evictionCount;

    /**
     * 缓存命中的累计次数
     */
    private int hitCount;

    /**
     * 缓存未命中的累计次数
     */
    private int missCount;
```

LruCache 构造器

```java
    /**
     * 初始化一个最大 maxSize 缓存大小的 LruCache 对象
     */
    public LruCache(int maxSize) {
        // maxSize 不能小于等于0，否则抛出非法参数异常
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        // 初始化一个默认长度为0，装载因子为0.75，并且按访问顺序排序的 LinkedHashMap
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

LruCache 单个元素占用缓存大小的计算

LruCache 通过 safeSizeOf 方法获取单个元素所占用的内存缓存大小，而 safeSizeOf 内调用了 sizeOf 方法来获取缓存大小，当 sizeOf 返回的 int 大小值小于0时将抛出错误, sizeOf 如果没有做任何实现将默认返回1，所以默认实现时， LruCache 的缓存 size 等于内部元素总个数。 

```java
    /**
     * 安全的通过 key 和 value 获得单个元素的缓存大小，当单个元素缓存大小小于0时将抛出异常
     */
    private int safeSizeOf(K key, V value) {
        int result = sizeOf(key, value);
        // 单个元素的缓存大小必须大于等于0
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }

    /**
     * 通过 key 和 value 获得单个元素的缓存大小，默认为1
     */
    protected int sizeOf(K key, V value) {
        return 1;
    }
```

LruCache 的缓存淘汰方法

```java
    /**
     * 通过传入一个缓存最大值，移除需要淘汰的缓存
     */
    public void trimToSize(int maxSize) {

        // 此处循环是为了一个个移除缓存后再计算当前缓存大小是否仍然超过最大缓存大小，如果超过则需要再次移除缓存
        while (true) {
            K key;
            V value;
            // 同步操作
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize || map.isEmpty()) {
                    // 当当前缓存大小已小于最大缓存大小或者当前维护的 map 对象内数据空了都代表没有需要再淘汰的缓存了，这时可以打断淘汰循环
                    break;
                }

                // 由于 LinkedHashMap 内数据按访问时间排序,且为从最久访问时间到最近访问时间排序的
                // 所以迭代器中获取的 next 节点就是最早需要被淘汰的数据
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                // 淘汰数据通过调用 map 的 remove 移除该元素
                map.remove(key);
                // 将当前的缓存大小减去淘汰的元素的缓存大小重新赋值
                size -= safeSizeOf(key, value);
                // 累计淘汰次数
                evictionCount++;
            }
            // entryRemoved 是个模板方法，为了子类实现时可以收到已淘汰了该 key value 元素的回调
            entryRemoved(true, key, value, null);
        }
    }
```

LruCache 的 put 方法

put 方法中考虑了两种情况，如果该 key 在散列表中产生冲突，并且与已存在一个节点的 key 值相等时，则在替换了该 value 值后，需要将缓存大小size 减去被替换的元素的缓存大小，并加上新替换的元素的缓存大小，如果没有冲突，则直接累加缓存大小。
由 HashMap 源码可知，当 map.put 方法调用返回了一个对象不为空时，就代表了该 key 产生了冲突，返回的这个对象就是之前该 key 所对应的元素对象，所以 LruCache 实现中正利用了这点。

```java
    public final V put(K key, V value) {
        // LruCache 不允许 key 或 value 为空值
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            // 累计 put 次数
            putCount++;
            // 获取该元素的大小后加到当前总缓存大小中
            size += safeSizeOf(key, value);
            // 添加该元素到 LinkedHashMap 中
            previous = map.put(key, value);
            if (previous != null) {
                // 如果 map.put 方法返回了一个非空对象，则代表之前已缓存过相同 key 的缓存，总缓存大小需要减去被替换元素的大小
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            // 因为有替换元素，所以需要通知子类之前的 previous 被移除替换为新值 value
            entryRemoved(false, key, previous, value);
        }

        // 进行一次缓存淘汰计算
        trimToSize(maxSize);
        return previous;
    }
```

LruCache 的 get 方法

get 方法在 LruCache 中也存在两种考虑，一种是简单直接从 LinkedHashMap 中命中了某个 key 的缓存元素就直接返回该缓存元素，另一种则是当没有该 key 缓存时，LruCache 提供了模板方法 create(key) 让子类创建一个缓存元素（该模板方法默认返回空值），如果子类有实现并返回了非空值，则会将该次创建的缓存元素再添加到缓存中，并返回该缓存元素。

```java
    public final V get(K key) {
        // 访问缓存的 key 不能为空
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        // 从 LinkedHashMap 中获取缓存元素
        V mapValue;
        synchronized (this) {
            // 通过 LinkedHashMap 的 get 方法获取到元素对象
            mapValue = map.get(key);
            if (mapValue != null) {
                // 获取到了缓存对象则累计命中数
                hitCount++;
                return mapValue;
            }
            // 没有获取到缓存对象则累计缓存未命中数
            missCount++;
        }

        // 当缓存未命中时，通过模板方法 create 方法获取一个新创建的缓存值对应该 key
        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        // 如果通过 create 方法创建了非空缓存对象，则将该缓存添加到 LinkedHashMap 中，并累加总缓存大小,以下实现和 put 类似
        synchronized (this) {
            // 累计新创建缓存次数
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }

    /**
     * 通过 key 创建一个新的缓存对象的模板方法，默认返回空值，可由子类实现
     */
    protected V create(K key) {
        return null;
    }
```