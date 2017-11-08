# ConcurrentHashMap简介
ConcurrentHashMaps是HashMap的线程安全实现，在java8中，为了最大限度的优化其性能，ConcurrentHashMap采用了数组+链表+红黑树的数据结构，同时，抛弃了java7中分段锁（Segment）的概念，对数组的每个index区域加锁，减少锁竞争，并尽量使用CAS来对对应值进行修改。在链表的长度大于8（默认值）时，将之转换为红黑树，将查询节点的算法复杂度从O(n)下降为O(lgn)。

# ConcurrentHashMap的put方法
put方法涉及到ConcurrentHashMap的很多设计方法，因此，我们以这个方法做为切入点。以下是put方法的源码：

```
    public V put(K key, V value) {
        return putVal(key, value, false);
    }
```

其实调用的是putVal方法，那么就看其源码：

```
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //ConcurrentHashMap的key和value都不能为空
        if (key == null || value == null) throw new NullPointerException();
        //计算key的hash值，这里的计算方法兼顾了效率和hash的分布效果，同时，也可以知道我们尽量不要使用对象作为key值
        int hash = spread(key.hashCode());
        int binCount = 0;
        //自旋table数组
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                //如果table为空，就对table进行初始化
                tab = initTable();
            //计算key的hash值对应的index位置，(n-1)&hash等同于hash%n,使用位运算更为高效，这里也是ConcurrentHashMap的容量设置为2的倍数的部分原因
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //如果对应index位的Node为空，那么使用CAS操作设置当前传入的值，CAS成功，退出当前循环，CAS失败的话继续自旋，此处是无锁操作
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //此处锁定了当前的index位置
                synchronized (f) {
                    //因为之前的操作存在多线程的并发问题，此处需要判断当前index位置的Node是否还是原值，如果是原值进行后续操作，否则继续自旋
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {//表示是链表,fh是hash值，如果是红黑树，应该是TREEBIN=-2
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //如果key相同，则覆盖原来的value值
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //插入链表尾部
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //红黑树调用putTreeVal方法
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
                    //链表的长度大于TREEIFY_THRESHOLD的值，则将链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    //如果是替换value值的情况，则返回原值return
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        //
        addCount(1L, binCount);
        return null;
    }
```

上述对源码进行了注释，基本讲了每一步做的事情，总结一下：
ConcurrentHashMap不在构造函数中进行初始化，而是放在put方法中，先判断table数组是否为空，为空就进行初始化，然后根据table的容量计算出当前key的index位置，如果该位置为null，就进行CAS操作赋值，否则对当前位置就行加锁，并判断当前是链表or红黑树，进行相应的操作。java8加锁的位置是对应的index位的Node，因此，加锁的粒度要细于java7中的Segment。

