#Mybatis Cache: A Brief Analysis of Implementation
#Mybatis缓存实现简析
`Mybatis` `Cache` `缓存`

##引子
Mybatis的缓存类非常丰富，简约不简单。<br/>
之前有一篇介绍Mybatis缓存查询结果的文章（传送门：[Mybatis Update前后Select返回相同结果](https://github.com/duanjfeng/trickle/blob/master/Java/Mybatis_LocalCache_Update%E5%89%8D%E5%90%8ESelect%E8%BF%94%E5%9B%9E%E7%9B%B8%E5%90%8C%E7%BB%93%E6%9E%9C.md)）。<br/>
本文简要分析一下Mybatis缓存类的实现，基于Mybatis-3.2.8。不是Mybatis缓存使用指南哦，需要指南的请移步[Mybatis 3 Cache Reference](http://mybatis.github.io/mybatis-3/sqlmap-xml.html#cache)。<br/>
Mybatis缓存相关的类位于org.apache.ibatis.cache包，包括基础接口/类、实现和装饰器。下面分别予以介绍。<br/>

##基础接口Cache
通常缓存会提供插入元素、获取元素、清空3个方法，Mybatis还要求每个缓存有一个唯一标识。
```Java
	
	public interface Cache {
	
	  //Mybatis的Cache都有一个唯一标识
	  String getId();
	
	  void putObject(Object key, Object value);
	
	  Object getObject(Object key);
	
	  //可选
	  Object removeObject(Object key);
	
	  //清理缓存实例 
	  void clear();
	
	  //可选，返回缓存中元素的数量
	  int getSize();
	  
	  //可选，提供一个读写锁
	  ReadWriteLock getReadWriteLock();
	  
	}
```

##基础类CacheKey
缓存有key和value。key是缓存值的标识，value是缓存的值。如何定义一个恰当的key是一个需要仔细思考的问题。例如，缓存一次查询结果，那么怎么样来标识一个查询呢？是statement的id？那动态SQL怎么办，参数不同要不要考虑？<br/>
都考虑吧，可是把这么多东西放一起构造一个key还是有点小麻烦的，所以Mybatis提供了一个构造key的工具类CacheKey。<br/>
###CacheKey的属性
CacheKey包含hashcode、checksum、count和updateList四个属性。通过这四个属性来唯一标识一个CacheKey对象，详见equals方法。<br/>
```Java
	
	// hashcode
	private int hashcode;
	// 校验和，所有对象的hashcode的和。null的计为1；如果对象是一个Array，那么处理Array里的元素。
	private long checksum;
	// 参与key构造的对象的数量。如果对象是一个Array，那么统计Array里的元素。
	private int count;
	// 参与key构造的对象。如果对象是一个Array，那么处理Array里的元素。
	private List<Object> updateList;
```
###CacheKey的构造
方法一：调用默认构造函数CacheKey()，然后多次调用update(Object object)或者updateAll(Object[] objects)方法。<br/>
方法二：调用构造函数CacheKey(Object[] objects)，然后多次调用update(Object object)或者updateAll(Object[] objects)方法。<br/>
无论是哪种方法，最终都要调用到核心私有方法doUpdate(Object object)。<br/>
```Java
	
	private void doUpdate(Object object) {
		// 基础hashcode，null为1，非null为对象的hashCode
		int baseHashCode = object == null ? 1 : object.hashCode();

		// 对象数加一
		count++;

		// 校验和加基础hashcode
		checksum += baseHashCode;

		baseHashCode *= count;
		// 新的hashcode的值：上一个hashcode乘以倍乘数，然后加上基础hashcode乘以对象数量
		hashcode = multiplier * hashcode + baseHashCode;

		// 对象列表增加当前对象
		updateList.add(object);
	}
```
<br/>
##实现类PerpetualCache
org.apache.ibatis.cache.impl.PerpetualCache是Mybatis里唯一的Cache实现类。<br/>
很简单，id由用户指定，存储使用一个HashMap。<br/>
```Java
	
	private String id;

	private Map<Object, Object> cache = new HashMap<Object, Object>();
```
<br/>
##装饰器decorators
包org.apache.ibatis.cache.decorators提供了一系列的装饰器，为底层的Cache添加一些特殊的性质/功能。
<br/>
###先进先出FifoCache
先进先出Cache类FifoCache有3个属性。<br/>
```Java
	
	// 底层Cache（被装饰的对象）
	private final Cache delegate;
	// key列表（有序）
	private LinkedList<Object> keyList;
	// 缓存大小
	private int size;
```
<br/>

在放入元素时检查size刷新keyList，然后刷新Cache，实现先进先出性质。<br/>
```Java
	
	@Override
	public void putObject(Object key, Object value) {
		cycleKeyList(key);
		delegate.putObject(key, value);
	}

	private void cycleKeyList(Object key) {
	    keyList.addLast(key);
	    if (keyList.size() > size) {
	      Object oldestKey = keyList.removeFirst();
	      delegate.removeObject(oldestKey);
	    }
	}
```
<br/>
###命中日志LoggingCache
统计命中次数和命中率，需要记录请求次数和命中次数。<br/>
```Java
	
	// 日志对象
	private Log log;  
	// 底层Cache（被装饰的对象）
	private Cache delegate;
	// 请求次数
	protected int requests = 0;
	// 命中次数
	protected int hits = 0;
```
<br/>

在获取元素的时候日志记录命中率。<br/>
```Java
	
	@Override
	public Object getObject(Object key) {
		requests++;
		final Object value = delegate.getObject(key);
		if (value != null) {
		  hits++;
		}
		if (log.isDebugEnabled()) {
		  log.debug("Cache Hit Ratio [" + getId() + "]: " + getHitRatio());
		}
		return value;
	}
```
<br/>
###最近最少使用LruCache
LRU，一个缩写两种理解，Last Recently Used（最近最久未使用）和Least Recently Used（最近最少使用）。<br/>
前者移除最近最久未使用的缓存对象，后者移除最近最少使用的缓存对象。<br/>
LruCache实现了前者，即将最近最久未被访问到的对象移除。<br/>
```Java
	
	// 底层Cache（被装饰的对象）
	private final Cache delegate;
	// keyMap，与实现相关，后面会说明
	private Map<Object, Object> keyMap;
	// 最近最久未访问的key
	private Object eldestKey;
```
<br/>

使用一个LinkedHashMap来构造keyMap，原因是利用LinkedHashMap可以按照访问顺序排序的特性。<br/>
- 构造函数LinkedHashMap(int initialCapacity,float loadFactor,boolean accessOrder)的第3个参数定义了排序方式，true为按照访问顺序排序，false为按照插入顺序排序，默认为false。<br/>
- 方法removeEldestEntry(Map.Entry<K,V> eldest)返回true/false，指示是否应该移除最古老的元素。由put/putAll方法调用。典型实现不应该执行修改map的操作，该操作应该由map自身来调用。特别的，如果实现中修改了map，那么应该返回false。<br/>
利用LinkedHashMap的上述特性，我们可以判断是否超出缓存预设大小，同时将eldestKey记录下来。<br/>
```Java
	
	public LruCache(Cache delegate) {
		this.delegate = delegate;
		setSize(1024);
	}
	public void setSize(final int size) {
		keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
		  private static final long serialVersionUID = 4267176411845948333L;
		
		  protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
		    boolean tooBig = size() > size;
		    if (tooBig) {
		      eldestKey = eldest.getKey();
		    }
		    return tooBig;
		  }
		};
	}
```
<br/>

向缓存中放入一个元素。<br/>
keyMap的put动作会触发removeEldestEntry的执行。当eldestKey非空时，我们需要从缓存中移除对应的元素。<br/>
```Java
	
	@Override
	public void putObject(Object key, Object value) {
		delegate.putObject(key, value);
		cycleKeyList(key);
	}

	private void cycleKeyList(Object key) {
		keyMap.put(key, key);
		if (eldestKey != null) {
		  delegate.removeObject(eldestKey);
		  eldestKey = null;
		}
	}
```
<br/>

访问缓存的元素。<br/>
keyMap的get动作会触发keyMap中元素的重新排序。<br/>
```Java
	
	@Override
	public Object getObject(Object key) {
		keyMap.get(key); //touch
		return delegate.getObject(key);
	}
```
<br/>
###定时清理ScheduledCache
定义一个时间间隔定时清空缓存。<br/>
```Java
	
	// 底层Cache（被装饰的对象）
	private Cache delegate;
	// 时间间隔
	protected long clearInterval;
	// 最后一次清理时间
	protected long lastClear;
```
<br/>

getSize/putObject/getObject/removeObject方法都会触发清理检查，如果时间间隔超过设定值则执行清理操作。<br/>
```Java
	
	private boolean clearWhenStale() {
		if (System.currentTimeMillis() - lastClear > clearInterval) {
		  clear();
		  return true;
		}
		return false;
	}
	
	@Override
	public void clear() {
		lastClear = System.currentTimeMillis();
		delegate.clear();
	}
```
<br/>
###序列化SerializedCache
将对象序列化后存入Cache，或者从Cache获取对象后反序列化返回。<br/>
没有使用额外的属性。<br/>
```Java
	
	// 底层Cache（被装饰的对象）
	private Cache delegate;
```
<br/>

serialize和deserialize方法通过调用ObjectOutputStream的writeObject和ObjectInputStream的readObject实现。<br/>
```Java
	
	@Override
	public void putObject(Object key, Object object) {
		if (object == null || object instanceof Serializable) {
		  delegate.putObject(key, serialize((Serializable) object));
		} else {
		  throw new CacheException("SharedCache failed to make a copy of a non-serializable object: " + object);
		}
	}
	
	@Override
	public Object getObject(Object key) {
		Object object = delegate.getObject(key);
		return object == null ? null : deserialize((byte[]) object);
	}
```
<br/>
###软引用SoftCache
为了节省内存空间，缓存中保存对象的软引用。软引用请参考[SoftReference](http://docs.oracle.com/javase/7/docs/api/java/lang/ref/SoftReference.html)。<br/>
```Java
	
	// 硬连接（引用），以防止被gc
	private final LinkedList<Object> hardLinksToAvoidGarbageCollection;
	// 软引用队列，保存被gc的软引用
	private final ReferenceQueue<Object> queueOfGarbageCollectedEntries;
	// 底层Cache（被装饰的对象）
	private final Cache delegate;
	// 硬连接数量
	private int numberOfHardLinks;
```
<br/>

getSize/putObject/removeObject/clear都会调用removeGarbageCollectedItems。移除所有被gc的对象。<br/>
```Java
	
	private void removeGarbageCollectedItems() {
		SoftEntry sv;
		while ((sv = (SoftEntry) queueOfGarbageCollectedEntries.poll()) != null) {
		  delegate.removeObject(sv.key);
		}
	}
```
<br/>

putObject时，缓存中存入的是一个软引用。<br/>
```Java
	
	@Override
	public void putObject(Object key, Object value) {
		removeGarbageCollectedItems();
		delegate.putObject(key, new SoftEntry(key, value, queueOfGarbageCollectedEntries));
	}

	private static class SoftEntry extends SoftReference<Object> {
		private final Object key;
		
		private SoftEntry(Object key, Object value, ReferenceQueue<Object> garbageCollectionQueue) {
		  super(value, garbageCollectionQueue);
		  this.key = key;
		}
	}
```
<br/>

getObject时，从缓存中取出软引用。如果软引用非空，则尝试获取对象。如果对象为空，则移除软引用；否则将对象放入hardLinksToAvoidGarbageCollection中。<br/>
```Java
@Override
	
	public Object getObject(Object key) {
		Object result = null;
		@SuppressWarnings("unchecked") // assumed delegate cache is totally managed by this cache
		SoftReference<Object> softReference = (SoftReference<Object>) delegate.getObject(key);
		if (softReference != null) {
		  result = softReference.get();
		  if (result == null) {
		    delegate.removeObject(key);
		  } else {
		    // See #586 (and #335) modifications need more than a read lock 
		    synchronized (hardLinksToAvoidGarbageCollection) {
		      hardLinksToAvoidGarbageCollection.addFirst(result);
		      if (hardLinksToAvoidGarbageCollection.size() > numberOfHardLinks) {
		        hardLinksToAvoidGarbageCollection.removeLast();
		      }
		    }
		  }
		}
		return result;
	}
```
<br/>
###同步SynchronizedCache
同步Cache的实现比较简单，所有的读写操作均使用synchronized关键字。
<br/>
###支持事务TransactionalCache
支持事务的缓存，增加了commit和rollback操作，实现事务控制。<br/>
```Java
	
	// 底层Cache（被装饰的对象）
	private Cache delegate;
	// 提交时是否先清空缓存，默认为false
	private boolean clearOnCommit;
	// 提交时添加的Entry
	private Map<Object, AddEntry> entriesToAddOnCommit;
	// 提交时删除的Entry
	private Map<Object, RemoveEntry> entriesToRemoveOnCommit;
```
<br/>

putObject时需要从entriesToRemoveOnCommit中删除对应的key，添加对应的AddEntry；removeObject时需要从entriesToAddOnCommit中删除对应的key，添加对象的RemoveEntry。<br/>
```Java
	
	@Override
	public void putObject(Object key, Object object) {
		entriesToRemoveOnCommit.remove(key);
		entriesToAddOnCommit.put(key, new AddEntry(delegate, key, object));
	}
	
	@Override
	public Object removeObject(Object key) {
		entriesToAddOnCommit.remove(key);
		entriesToRemoveOnCommit.put(key, new RemoveEntry(delegate, key));
		return delegate.getObject(key);
	}
```
<br/>

- commit操作：如果clearOnCommit为true则需要先清空底层Cache，否则将entriesToRemoveOnCommit中的元素从底层Cache移除。然后将entriesToAddOnCommit中的元素添加到底层Cache中。<br/>
- rollback操作：设置clearOnCommit为false，清空entriesToRemoveOnCommit和entriesToAddOnCommit。<br/>

```Java
	
	public void commit() {
		if (clearOnCommit) {
		  delegate.clear();
		} else {
		  for (RemoveEntry entry : entriesToRemoveOnCommit.values()) {
		    entry.commit();
		  }
		}
		for (AddEntry entry : entriesToAddOnCommit.values()) {
		  entry.commit();
		}
		reset();
	}
	
	public void rollback() {
		reset();
	}
	
	private void reset() {
		clearOnCommit = false;
		entriesToRemoveOnCommit.clear();
		entriesToAddOnCommit.clear();
	}
```
<br/>
###弱引用WeakCache
为了节省内存空间，缓存中保存对象的弱引用。弱引用请参考[WeakReference](http://docs.oracle.com/javase/7/docs/api/java/lang/ref/WeakReference.html)。<br/>
实现细节请参看上面的SoftCache一节。
<br/>

##相关链接
[Mybatis 3 Cache使用指南](http://mybatis.github.io/mybatis-3/sqlmap-xml.html#cache)<br/>
<br/>
2015-4-18