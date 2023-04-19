# ThreadLocal

## 示例代码

```java
public class ThreadLocalExample implements Runnable{

    // SimpleDateFormat 不是线程安全的，所以每个线程都要有自己独立的副本
    private static final ThreadLocal<SimpleDateFormat> formatter 
        = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyyMMdd HHmm"));

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalExample obj = new ThreadLocalExample();
        for(int i=0 ; i<10; i++){
            Thread t = new Thread(obj, ""+i);
            Thread.sleep(new Random().nextInt(1000));
            t.start();
        }
    }

    @Override
    public void run() {
        sout("Thread Name= "+Thread.currentThread().getName()
             +" default Formatter = "+formatter.get().toPattern());
        try {
            Thread.sleep(new Random().nextInt(1000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        formatter.set(new SimpleDateFormat());
        sout("Thread Name= "+Thread.currentThread().getName()+" formatter = "+formatter.get().toPattern());
    }

}


```

结果：

```text
Thread Name= 0 default Formatter = yyyyMMdd HHmm
Thread Name= 0 formatter = yy-M-d ah:mm
Thread Name= 1 default Formatter = yyyyMMdd HHmm
Thread Name= 2 default Formatter = yyyyMMdd HHmm
Thread Name= 1 formatter = yy-M-d ah:mm
Thread Name= 3 default Formatter = yyyyMMdd HHmm
Thread Name= 2 formatter = yy-M-d ah:mm
Thread Name= 4 default Formatter = yyyyMMdd HHmm
Thread Name= 3 formatter = yy-M-d ah:mm
Thread Name= 4 formatter = yy-M-d ah:mm
Thread Name= 5 default Formatter = yyyyMMdd HHmm
Thread Name= 5 formatter = yy-M-d ah:mm
Thread Name= 6 default Formatter = yyyyMMdd HHmm
Thread Name= 6 formatter = yy-M-d ah:mm
Thread Name= 7 default Formatter = yyyyMMdd HHmm
Thread Name= 7 formatter = yy-M-d ah:mm
Thread Name= 8 default Formatter = yyyyMMdd HHmm
Thread Name= 9 default Formatter = yyyyMMdd HHmm
Thread Name= 8 formatter = yy-M-d ah:mm
Thread Name= 9 formatter = yy-M-d ah:mm
```

## get方法

```java
//this ->调用get方法的ThreadLocal对象

//Class:ThreadLocal
public T get() {
    Thread t = Thread.currentThread(); //获取当前线程
    ThreadLocalMap map = getMap(t);	
    //获取当前线程的ThreadLocalMap类型的threadLocals。
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        //获取调用get方法的对象存储在当前线程的值
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();	//当前线程threadLocals为null，初始化并返回null
    							//当前线程threadLcoals不为null，threadLocals里this赋值null，并返回null
}


//Class:ThreadLocal
//获取指定线程的ThreadLocalMap类型的threadLocals
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}


//Class:Thread
ThreadLocal.ThreadLocalMap threadLocals = null;


//Class:ThreadLocal.ThreadLocalMap
//获取当前线程的threadLocals中key是key的value
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}


//Class:ThreadLocal
private T setInitialValue() {
    T value = initialValue(); //结果为null
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
protected T initialValue() {
    return null;
}
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

get()方法就是返回当前线程里存储的调用get方法的ThreadLocal对象对象的值，如果没有就向线程存储这个ThreadLocal对象和null。

## set方法

```java
//this -> 调用set方法的ThreadLocal对象

//Class:ThreadLocal
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}


//Class:ThreadLocal
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

set()方法可以看出每个线程的都有一个属于自己的ThreadLocal对应的值,他们不能互相干扰,属于线程安全的

## remove方法

```java
//Class:ThreadLocal
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
    	m.remove(this);
}

//Class:ThreadLocal.ThreadLocalMap
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

## ThreadLocalMap.Entry

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

## ThreadLocalMap

### hash算法

```java
private void set(ThreadLocal<?> key, Object value) {
    ...
	int i = key.threadLocalHashCode & (len-1);
    ...
}

private final int threadLocalHashCode = nextHashCode(); //属于对象的常量

private static int nextHashCode() { 			//静态方法
    return nextHashCode.getAndAdd(HASH_INCREMENT); 
}

private static AtomicInteger nextHashCode =
        new AtomicInteger();					//静态变量

private static final int HASH_INCREMENT = 0x61c88647;
					//这个值很特殊，它是斐波那契数也叫黄金分割数。hash增量为这个数字，带来的好处就是hash分布非常均匀。
```

`new ThreadLocal()对象时，就会调用静态方法nextHashCode()方法赋值给threadLocalHashCode,这时因为threadLocalHashCode属性是final所以它将是一个常量再也不变，而在不同线程中调用这个ThreadLocal对象的get或set方法都是将这个对象作为key保存到不同线程中的threadLocals里面，因此同一个ThreadLocal对象即使在不同线程可能因为len相同使得它们计算的哈希码值都一样.`

`静态方法nextHashCode()和静态变量nextHashCode会导致不同的ThreadLocal对象的threadLocalHashCode属性不同,也会导致不同对象计算的哈希码不同`

虽然`ThreadLocalMap`中使用了**黄金分割数**来作为`hash`计算因子，大大减少了`Hash`冲突的概率，但是仍然会存在冲突。

`HashMap`中解决冲突的方法是在数组上构造一个**链表**结构，冲突的数据挂载到链表上，如果链表长度超过一定数量则会转化成**红黑树**。

而 `ThreadLocalMap` 中并没有链表结构，所以这里不能使用 `HashMap` 解决冲突的方式了。

如果我们插入一个数据，通过 `hash` 计算后应该落入槽位 4 中，而槽位 4 已经有了 `Entry` 数据。此时就会线性向后查找，一直找到 `Entry` 为 `null` 的槽位才会停止查找，将当前元素放入此槽位中。

### set方法

#### api层

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) { 		//如果当前Entry的key与key相等
            e.value = value;	//当前Entry的key对应的value替换为value
            return;
        }

        if (k == null) {		//如果当前Entry的key为null
            replaceStaleEntry(key, value, i);  //替换过期Entry
            return;
        }
    }

    //当遍历到某个Entry为null时
    tab[i] = new Entry(key, value);
    int sz = ++size;
    //未清理到任何数据且当前散列数组中Entry的数量已经达到了列表的扩容阈值(len*2/3)，就开始执行rehash()逻辑
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

#### 计算数组中前后下标

```java
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

#### 替换过期Entry

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    //它是用来判断当前过期槽位staleSlot之前是否还有过期元素。
    int slotToExpunge = staleSlot;
    //以当前staleSlot开始向前迭代查找，找其他过期的数据，然后更新过期数据起始扫描下标slotToExpunge
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;  //当没有遇到空槽
         i = prevIndex(i, len))
        if (e.get() == null)	//当前Entry的key过期
            slotToExpunge = i;

    //从当前桶下标之后的桶下标开始向后遍历
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;	//当没有遇到空槽
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        if (k == key) { //当遇到某个槽的key与key相等
            //更新Entry的值并交换staleSlot元素的位置
            e.value = value;
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            //如果slotToExpunge == staleSlot，这说明replaceStaleEntry()一开始向前查找过期数据时并未找到过期的Entry数据，接着向后查找过程中也未发现过期数据，修改开始探测式清理过期数据的下标为当前循环的 index，即slotToExpunge = i
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            //过期Entry的清理工作
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
		//k == null说明当前遍历的Entry是一个过期数据，slotToExpunge == staleSlot说明，一开始的向前查找数据并未找到过期的Entry。如果条件成立，则更新slotToExpunge 为当前位置，这个前提是前驱节点扫描时未发现过期数据。
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    //如果没有找到k == key的数据，且碰到Entry为null的数据，则结束当前的迭代操作。此时说明这里是一个添加的逻辑，将新的数据添加到table[staleSlot] 对应的slot中
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
	//还有其他过期Entry
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

#### 两种清理方式

```java
//探测式清理
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    //将过期数据清理
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    //往后探测,如果遇到过期数据就清空槽位数据，遇到未过期数据重新计算hash然后插入某个槽以尽量让该数据重新插入的槽接近本来应该插入的槽
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        
        if (k == null) {
            //清空槽位数据
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            //重新计算hash然后插入某个槽以尽量让该数据重新插入的槽接近本来应该插入的槽
            //经过迭代后，有过Hash冲突数据的Entry位置会更靠近正确位置，这样的话，查询的时候效率才会更高。
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;//在for循环中第一个探测到的空槽下标
}

//启发式清理
//i-已知非过期entry位置。扫描从i之后的元素开始。
//n-扫描控制:扫描log2(n)个单元格，除非发现一个过期entry，在这种情况下，扫描log2(table.length)-1个额外的单元格。当从replaceStaleEntry调用时，它是表的长度
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {//当前扫描的entry已过期
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;//是否清除过过期entry
}

```

### get方法

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)//直接找到对应的槽
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

### 扩容机制

```java
private int threshold; 
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
private void rehash() {
    //探测式清理工作,从table的起始位置往后清理
    expungeStaleEntries();

    //此时通过判断size >= threshold * 3/4 = len * 2/3 * 3/4 = len * 1/2来决定是否扩容。
    if (size >= threshold - threshold / 4)
        //扩容
        resize();
}
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
//扩容
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    //新数组长度为旧数组长度的2倍
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {	//当为过期Entry则让该entry的value指向null,强引用等待GC
                e.value = null;
            } else {			//为非过期Entry，让该Entry在新数组中尽量放在接近它的hash位置
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
	//更新ThreShold
    setThreshold(newLen);
    size = count;
    table = newTab;
}
```



## InheritableThreadLocal

在异步场景下是解决子线程共享父线程中创建的线程副本数据的。

```java
public class InheritableThreadLocalDemo {
    public static void main(String[] args) {
        ThreadLocal<String> ThreadLocal = new ThreadLocal<>();
        ThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
        ThreadLocal.set("父类数据:threadLocal");
        inheritableThreadLocal.set("父类数据:inheritableThreadLocal");

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("子线程获取父类ThreadLocal数据：" + ThreadLocal.get());
                System.out.println("子线程获取父类inheritableThreadLocal数据：" + 			inheritableThreadLocal.get());
            }
        }).start();
    }
}
```

```text
子线程获取父类ThreadLocal数据：null
子线程获取父类inheritableThreadLocal数据：父类数据:inheritableThreadLocal
```

### 实现

```java
//在一个线程中调用Thread的构造方法创建子线程
//在构造方法中有
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
    ...
	if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    ...
}
```

但`InheritableThreadLocal`仍然有缺陷，一般我们做异步化处理都是使用的线程池，而`InheritableThreadLocal`是在`new Thread`中的`init()`方法给赋值的，而线程池是线程复用的逻辑，所以这里会存在问题。

当然，有问题出现就会有解决问题的方案，阿里巴巴开源了一个`TransmittableThreadLocal`组件就可以解决这个问题。

