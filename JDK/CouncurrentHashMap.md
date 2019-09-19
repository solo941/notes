# ConcurrentHashMap

HashMap线程不安全，体现在发生扩容时会出现环形链表，需要线程安全的并发容器ConcurrentHashMap。在JDK1.7中，ConcurrentHashMap基于Segment+HashEntry+ReentrantLock实现；JDK1.8中采用Node+CAS+Synchronized实现。

## JDK1.7

如图所示，在1.7中，ConcurrentHashMap 采用了分段锁（Segment），每个分段锁维护着几个桶（HashEntry），多个线程可以同时访问不同分段锁上的桶，从而使其并发度更高（并发度就是 Segment 的个数），默认的并发级别为 16。

不像HashTable，不管put还是get都需要做同步处理，每当一个线程访问一个Segment时，不会影响其他的Segment。

### get

```java

public V get(Object key) {
	Segment<K,V> s;
	HashEntry<K,V>[] tab;
	int h = hash(key);//计算key的hash值
	long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
	//通过key的Hash值找到Segment及其中的HashEntry[]
	if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
		(tab = s.table) != null) {
		//通过key找到链表并循环
		for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
				 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
			 e != null; e = e.next) {
			K k;
			//如果key相等就返回value
			if ((k = e.key) == key || (e.hash == h && key.equals(k)))
				return e.value;
		}
	}
	return null;
}

```

读的过程没有加锁，使用getObjectVolatile获取Segment和table，并且HashEntry中的value属性使用volatile修饰，保证可见性。

### put

```java

//Segment中的put()方法，参数：要插入的key，key的hash值，值，false

final V put(K key, int hash, V value, boolean onlyIfAbsent) {
	//分段锁Segment为了保证插入数据的安全性，再插入之前要获取对象独占锁
	//Segment继承ReentrantLock，通过tryLock获取独占锁，获取锁返回null
	//没有获取到锁，通过scanAndLockForPut()获取
	HashEntry<K,V> node = tryLock() ? null :
		scanAndLockForPut(key, hash, value);
	V oldValue;
	try {
		HashEntry<K,V>[] tab = table;//获取其中的HashEntry[]
		int index = (tab.length - 1) & hash;//通过Key的hash值计算需要存储在HashEntry的位置
		//获取HashEntry[index]位置的链表的第一个元素
		HashEntry<K,V> first = entryAt(tab, index);
		for (HashEntry<K,V> e = first;;) {
			if (e != null) {//当前循环到的链表节点不等于null
				K k;
				//节点中的key与当前要存储的key相等
				if ((k = e.key) == key ||
					(e.hash == hash && key.equals(k))) {
					oldValue = e.value;
					if (!onlyIfAbsent) {//替换存储的值
						e.value = value;
						++modCount;
					}
					break;
				}
				e = e.next;//赋值e，用于下次循环
			}
			else {//当前循环到的链表节点等于null
				if (node != null)// 如果node不为 null，那就直接将它设置为链表表头
					node.setNext(first);
				else
					node = new HashEntry<K,V>(hash, key, value, first);//新建
				int c = count + 1;
				if (c > threshold && tab.length < MAXIMUM_CAPACITY)//如果HashEntry[]中存储的键值个数大于阈值，并且小于最大值
					rehash(node);//扩容
				else
					setEntryAt(tab, index, node);//将节点加入到index链表中，设置为链表表头
				++modCount;
				count = c;
				oldValue = null;
				break;
			}
		}
	} finally {
		unlock();//释放锁
	}
	return oldValue;
}
```

这里在第一步会尝试获取锁，如果没有获取到就通过scanAndLockForPut()获取，scanAndLockForPut()执行完毕肯定获取到了Segment锁，所以put方法是线程安全的，scanAndLockForPut()是控制写锁的关键,它通过一个while循环尝试获取Segment锁，并循环计数，如果retries>1/64(单核1,多核64)，则通过lock()进入AQS阻塞队列，直到获取锁后，才会返回，跳出循环。

### remove

remove()方法为保证线程安全，也会上锁。

```java

//Segmen中的remove方法，参数为：key，key的Hash，null
final V remove(Object key, int hash, Object value) {
	if (!tryLock())//先尝试获取锁
		scanAndLock(key, hash);//位获取到，通过scanAndLock()获取
	V oldValue = null;
	try {
		HashEntry<K,V>[] tab = table;
		int index = (tab.length - 1) & hash;//计算key在HashEtry[]中的存储位置
		HashEntry<K,V> e = entryAt(tab, index);//获取这个位置上的链表第一个节点
		HashEntry<K,V> pred = null;
		while (e != null) {//循环
			K k;
			HashEntry<K,V> next = e.next;//获取下一个节点
			//在链表中找到了key所在的节点
			if ((k = e.key) == key ||
				(e.hash == hash && key.equals(k))) {
				V v = e.value;
				if (value == null || value == v || value.equals(v)) {
					if (pred == null)//pred==null说明key所在节点为链表第一个节点
						setEntryAt(tab, index, next);//直接将下一个节点设置为头节点
					else
						pred.setNext(next);//否则将当前节点的上一个节点的next设置为下一个节点，这样key所在节点就移除了
					++modCount;
					--count;
					oldValue = v;
				}
				break;
			}
			pred = e;//记录当前节点，用于下次循环还是表示上一个节点
			e = next;//修改下次循环的节点
		}
	} finally {
		unlock();//释放锁
	}
	return oldValue;
}

```

### rehash

在put方法中，如果HashEntry[]中存储的键值个数大于阈值，并且小于最大值，将会触发扩容。

- 扩容后的数组长度是原数组的2倍。
- 扩容的是Segmen中的HashEntry[]，不是所有的Segment中的HashEntry[]。
- 扩容不需要考虑并发，因为到这里的时候，是持有该 segment 的独占锁的。

```java

//扩容，Segment中，只对Segment中的HashEntry[]扩容
@SuppressWarnings("unchecked")
private void rehash(HashEntry<K,V> node) {
	HashEntry<K,V>[] oldTable = table;//保存老的HashEntry[]
	int oldCapacity = oldTable.length;
	int newCapacity = oldCapacity << 1;//扩容为原来的2倍
	threshold = (int)(newCapacity * loadFactor);//计算新的阈值
	HashEntry<K,V>[] newTable =
		(HashEntry<K,V>[]) new HashEntry[newCapacity];//创建新的HashEntry[]
	int sizeMask = newCapacity - 1;
	for (int i = 0; i < oldCapacity ; i++) {//循环移动老的数组中的元素
		HashEntry<K,V> e = oldTable[i];
		if (e != null) {
			HashEntry<K,V> next = e.next;
			int idx = e.hash & sizeMask;//计算在HashEntry[]中存放的位置
			if (next == null)   //当前节点的下一个节点为null，说明当前链表就一个节点
				newTable[idx] = e;//直接赋值
			else { //存在链表
				HashEntry<K,V> lastRun = e;
				int lastIdx = idx;
				//循环找到一个 lastRun 节点，这个节点之后的所有元素是将要放到一起的
				for (HashEntry<K,V> last = next;
					 last != null;
					 last = last.next) {
					int k = last.hash & sizeMask;
					if (k != lastIdx) {
						lastIdx = k;
						lastRun = last;
					}
				}
				newTable[lastIdx] = lastRun;//复制链表
				//处理lastRun之前的节点，这些节点可能分配在另一个链表中，也可能分配到上面的那个链表中
				for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
					V v = p.value;
					int h = p.hash;
					int k = h & sizeMask;
					HashEntry<K,V> n = newTable[k];
					newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
				}
			}
		}
	}
	 // 将新加的 node 放到新数组中刚刚的两个链表之一的头部
	int nodeIndex = node.hash & sizeMask;
	node.setNext(newTable[nodeIndex]);
	newTable[nodeIndex] = node;
	table = newTable;
}

```

## JDK1.8

JDK1.8中HashMap实现当链表长度大于等于7且数组长度大于64时，会自动将链表转换为红黑树存储，这样的目的是尽量提高查询效率，而在JDK1.8中ConcurrentHashMap相对于JDK1.7的ConcurrentHashMap也做了优化，JDK1.7中ConcurrentHashMap使用分段锁的形式，比较细粒度的控制线程安全性问题，不至于像Hashteble那样，使用全表加锁，限制了并发性，但看过源码后发现，每次在get或put的时候，都会先查询到链表的第一个元素，换个思路想想，如果能就在第一个记录上加锁，这样不也可以解决线程安全性问题，所以JDK1.8就不存在Segmrnt[]了，而是使用CAS+synchronized的形式解决线程安全性问题，这样就比JDK1.7更加细粒度的加锁，并发性能更好。


### get

```java
 public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            //头结点的hash值<0,表示正在扩容或者其节点红黑树		
            //参考 ForwardingNode.find(int h, Object k) 和 TreeBin.find(int h, Object k)			//ForwardingNode中保存了nextTable为扩容的新数组，其中的find方法会在新数组中查询对应的节点
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

get操作全程不需要加锁，Node的成员val使用volatile修饰，并且next数组用volatile修饰，保证数组扩容时的可见性。在多线程环境下，A线程修改结点的val或者新增节点，线程B可见。

### put

```java
 final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //头节点的hash==-1，表示当前有线程在扩容数组，节点为ForwardingNode类型的，       
            //hash值为-1，数组在扩容时，会将正在迁移数据原数组的链表或红黑树的头结点设置为         
            //ForwardingNode类型，表示当前链表或红黑树已经迁移完成。也就是说当前正在进行数据迁移
            //,如果迁移完毕，会将新数组覆盖掉老数组，也就不会出现ForwardingNode类型的节点
            else if ((fh = f.hash) == MOVED)
                //当前线程帮着移动数据
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
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
                                    //如果当前节点的下一个节点为null，链表中不存在当前要插入的key
									//新建一个节点，作为当前节点的下一个节点
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

JDK1.8的扩容是比较麻烦的，当一个线程发起扩容后，其他线程也发现数组需要扩容时，这个线程就会去帮忙迁移数据，而不是扩容，这样效率会更高。

### helpTransfer()

```java
 final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

### tryPresize

```java
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
                    //用CAS将sizeCtl加1
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //迁移数据，nt不等于null，说明此线程不是第一个发起迁移数据的线程
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    //第一次发起迁移数据的线程是传入null，当第二个线程再来迁移数据时，就会执行上面的迁移方法，不会传入null
                    transfer(tab, null);
            }
        }
    }
```

迁移数据机制：原数组长度为 n，所以我们有 n 个迁移任务，让每个线程每次负责一个小任务是最简单的，每做完一个任务再检测是否有其他没做完的任务，帮助迁移就可以了，而 Doug Lea 使用了一个 stride，简单理解就是步长，每个线程每次负责迁移其中的一部分，如每次迁移 16 个小任务。所以，我们就需要一个全局的调度者来安排哪个线程执行哪几个任务，这个就是属性 transferIndex 的作用。第一个发起数据迁移的线程会将 transferIndex 指向原数组最后的位置，然后从后往前的 stride 个任务属于第一个线程，然后将 transferIndex 指向新的位置，再往前的 stride 个任务属于第二个线程，依此类推。当然，这里说的第二个线程不是真的一定指代了第二个线程，也可以是同一个线程。其实就是将一个大的迁移任务分为了一个个任务包。

### remove

```java
final V replaceNode(Object key, V value, Object cv) {
        int hash = spread(key.hashCode());
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break;
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                boolean validated = false;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            validated = true;
                            for (Node<K,V> e = f, pred = null;;) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    V ev = e.val;
                                    if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                        oldVal = ev;
                                        if (value != null)
                                            e.val = value;
                                        else if (pred != null)
                                            pred.next = e.next;
                                        else
                                            setTabAt(tab, i, e.next);
                                    }
                                    break;
                                }
                                pred = e;
                                if ((e = e.next) == null)
                                    break;
                            }
                        }
                        else if (f instanceof TreeBin) {
                            validated = true;
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                V pv = p.val;
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    if (value != null)
                                        p.val = value;
                                    else if (t.removeTreeNode(p))
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                    }
                }
                if (validated) {
                    if (oldVal != null) {
                        if (value == null)
                            addCount(-1L, -1);
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }
```

核心方法是replaceNode，如果正在扩容需要帮助迁移，其他的和1.7类似，同样需要对Node加锁。

## 1.7与1.8的对比

JDK1.8的实现降低锁的粒度，JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry（首节点）。
	JDK1.8版本的数据结构变得更加简单，使得操作也更加清晰流畅，因为已经使用synchronized来进行同步，所以不需要分段锁的概念，也就不需要Segment这种数据结构了，由于粒度的降低，实现的复杂度也增加了。
	JDK1.8使用红黑树来优化链表，基于长度很长的链表的遍历是一个很漫长的过程，而红黑树的遍历效率是很快的，代替一定阈值的链表。

## 参考资料

[**JDK1.7&1.8中ConcurrentHashMap解析**](https://blog.csdn.net/qq_36625757/article/details/90074355)

[**深入浅出ConcurrentHashMap1.8**](https://www.jianshu.com/p/c0642afe03e0)

