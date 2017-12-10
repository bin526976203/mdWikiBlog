**Java集合**-自动扩容与Hash冲突
=========

ArrayList自动扩容实现
-----------
Hint:直接上源码，JDK1.8中的ArrayList；

```
//往ArrayList中新增一个元素var1
public boolean add(E var1) {
    this.ensureCapacityInternal(this.size + 1);
    this.elementData[this.size++] = var1;
    return true;
}

//新增元素前确保集合容量足够检查
private void ensureCapacityInternal(int var1) {
	//为空时初始化默认为10
    if(this.elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        var1 = Math.max(10, var1);
    }

    this.ensureExplicitCapacity(var1);
}

//如果当前的所需的容量大于当前容量，调用grow方法
private void ensureExplicitCapacity(int var1) {
++this.modCount;
 if(var1 - this.elementData.length > 0) {
     this.grow(var1);
 }

//新的集合容量 = 当前容量 + (当前容量 >> 1);新的集合将复制获取到旧集合的顺序数据，然后旧集合将被回收
private void grow(int var1) {
    int var2 = this.elementData.length;
    int var3 = var2 + (var2 >> 1);
    if(var3 - var1 < 0) {
        var3 = var1;
    }

    if(var3 - 2147483639 > 0) {
        var3 = hugeCapacity(var1);
    }

    this.elementData = Arrays.copyOf(this.elementData, var3);
}
}
```

 1. ArrayList先检查 当前size+1，当前数组是否可以支持。

 2. 如果不支持，new新数组，新数组size = 当前容量 + (当前容量 >> 1)

 3. newArray = Arrays.copyOf(this.elementData, newElement);将旧数组与最新的元素数据置值到新数组中。旧数组完成回收
 
 * * *
 
HashMap自动扩容实现
----------

Note:JDK1.7的逻辑相对简单，JDK1.8使用了红黑树TreeMap相对复杂，现在用1.7的进行讲解。本质上其实一样。

```
void resize(int newCapacity) {   //传入新的容量  
    Entry[] oldTable = table;    //引用扩容前的Entry数组  
    int oldCapacity = oldTable.length;  
    if (oldCapacity == MAXIMUM_CAPACITY) {  //扩容前的数组大小如果已经达到最大(2^30)了  
        threshold = Integer.MAX_VALUE; //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了  
        return;  
    }  
  
    Entry[] newTable = new Entry[newCapacity];  //初始化一个新的Entry数组  
    transfer(newTable);                         //！！将数据转移到新的Entry数组里  
    table = newTable;                           //HashMap的table属性引用新的Entry数组  
    threshold = (int) (newCapacity * loadFactor);//修改阈值  
}  

void transfer(Entry[] newTable) {  
    Entry[] src = table;                   //src引用了旧的Entry数组  
    int newCapacity = newTable.length;  
    for (int j = 0; j < src.length; j++) { //遍历旧的Entry数组  
        Entry<K, V> e = src[j];             //取得旧Entry数组的每个元素  
        if (e != null) {  
            src[j] = null;//释放旧Entry数组的对象引用（for循环后，旧的Entry数组不再引用任何对象）  
            do {  
                Entry<K, V> next = e.next;  
                int i = indexFor(e.hash, newCapacity); //！！重新计算每个元素在数组中的位置  
                e.next = newTable[i]; //标记[1]  
                newTable[i] = e;      //将元素放在数组上  
                e = next;             //访问下一个Entry链上的元素  
            } while (e != null);  
        }  
    }  
}  

static int indexFor(int h, int length) {  
    return h & (length - 1);  
}  
```

扩容的方式是新建一个newTab，是oldTab的2倍。遍历oldTab，将oldTab赋值进对应位置的newTab。与ArrayList中的扩容逻辑基本一致，只不过ArrayList是当前容量+(当前容量>>1)。


**JDK1.8实现resize()方式**
```
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
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

* * *

HashMap-Hash冲突解决
-----------

背景：我们常用HashMap作为我们Java开发时的K-V数据存储结构(如id-person，这个ID对应这个人)。我们知道他们的数据结构么，它的Hash值是什么意义。Hash冲突是怎么解决的。我们带着这2个问题将HashMap做个整体剖析。（其实还有一个问题是，它怎么进行动态扩容的）

Note:一、HashMap的数据结构是什么。

下面是HashMap中的源码。其实HashMap的本质是Node数组；K-V结构对应的基础数据结构就是以下源码

```
transient HashMap.Node<K, V>[] table;

static class Node<K,V> implements Map.Entry<K,V> {
		//Hash值
        final int hash;
        //Key值
        final K key;
        //Value值
        V value;
        //当前Node对应的下个Node(用于解决Hash冲突；稍后讲解)
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
    }
```

根据上面代码我们知悉，HashMap中基本的数据结构是Node。

```
Map persons = new HashMap<String,String>();
//put方法会新建一个Node 
//hash=k.hashCode() ^ k >>> 16 ; hash是数组所在位置
//K="1",V="jack";next=null;(无Hash冲突情况)
persons.put("1","jack");
persons.put("2","john");
```

上述的Node数据结构中的hash值的意义是将Node均匀的放置在数组[]中。获得hash值后使用在table[(n - 1) & hash] 位置置值。

```
/**
* Computes key.hashCode() and spreads (XORs) higher bits of hash
* to lower.  Because the table uses power-of-two masking, sets of
* hashes that vary only in bits above the current mask will
* always collide. (Among known examples are sets of Float keys
* holding consecutive whole numbers in small tables.)  So we
* apply a transform that spreads the impact of higher bits
* downward. There is a tradeoff between speed, utility, and
* quality of bit-spreading. Because many common sets of hashes
* are already reasonably distributed (so don't benefit from
* spreading), and because we use trees to handle large sets of
* collisions in bins, we just XOR some shifted bits in the
* cheapest possible way to reduce systematic lossage, as well as
* to incorporate impact of the highest bits that would otherwise
* never be used in index calculations because of table bounds.
*/
static final int hash(Object var0) {
    int var1;
    return var0 == null?0:(var1 = var0.hashCode()) ^ var1 >>> 16;
}  
```

* * *

Note:二、HashMap怎么处理hash冲突

上述说道，Node中的hash值是用hash算法得到的，目的是均匀放置，那如果put的过于频繁，会造成不同的key-value计算出了同一个hash，这个时候hash冲突了，那么这2个Node都放置到了数组的同一个位置。HashMap是怎么处理这个问题的呢？

**解决方案：HashMap的数据结构是：数组Node[]与链表Node中有next Node.**

![HashMap中的实际数据结构](http://img.blog.csdn.net/20171022201400006?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjcxMjkwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

(1)如果上述的 persons.put("1","jack");persons.put("2","john"); 同时计算到的hash值都为123，那么jack先放在第一列的第一个位置Node-jack，persons.put("2","john");执行时会将Node-jack的next(Node) = Node(john)，Jack的下个节点将指向Node(john)。

(2)那么取的时候呢，persons.get("2")，这个时候取得的hash值是123，即table[123]，这时table[123]其实是Node-jack，Key值不相等，取Node-jack的next下个Node，即Node-John，这时Key值相等了，然后返回对应的person.

* * *

