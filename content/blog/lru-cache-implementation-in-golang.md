---
title: "[GO]golang如何实现lru缓存"
date: 2020-02-19T14:07:36+08:00
draft: true
---
lru缓存，在当今依然非常热门的面试问题，笔者曾经面试【某蜓，音频领域的top公司】就问到了这个问题，当时回答的并不是好。那我们一起来看看如何从0-1实现一个完美的lru缓存。
<!--more-->


## 简介
缓存，在日常的使用中帮助我们降低接口的响应速度，常常是将一些数据加载到**内存**当中，调用方的请求进来之后服务可以直接从内存中拿到数据，然后直接返回，减少了代码里面的逻辑处理，如果有数据查询，或者网络调用的话，这些步骤都可以省去，这些时间是非常可观的。所以非时间敏感的数据是可以加载到内存中去的，但是主机的资源是有限的，这其中内存资源尤其的紧张，所以缓存策略会**限制使用的内存大小**，当缓存满了之后，新进来的元素就需要踢掉旧有的某个元素，这就涉及到**缓存淘汰**。



## 缓存淘汰算法介绍
#### FIFO(first input first output)
先进先出，符合人的直观感受，当超过设定的内存size之后淘汰掉先进的元素，如何设计这种数据结构，很容易联想起数组【array】和队列【Queue】，

#### 数组的实现
当然这是很不灵活的方法，但是将其写下来，再跟其他方式对比，才能更加深入肌理。使用go的数组实现FIFO不复杂，如下图所示：淘汰e1的时候，数组中的元素需要往后移动，e2占了A[0]的位置，e3占了A[1]的位置，以此类推，直到空出的位置插入e6。

![image](/img/FIFO-flow-chart.png)

##### 队列的实现
简单的话,使用golang的`container/list`包双向链表，不考虑时间的话，可以自己实现一个单链表，满足fifo的要求。

如图所示：
![image](/img/single-list.jpg)

链表的好处在于，删除头部元素以及插入尾部元素的时间复杂度分别是O(1)，和O(n), 相较于array紧密的内存分布，list是通过指针相连，内存较为分散。但是array需要遍历才能移动元素，最终才能插入一个数据的方式，如果数量较大的情况下，计算的时间非常可观，list找到节点后添加/或者淘汰即可。

#### LFU
LFU  == least frequently used，最少使用的元素需要被淘汰，最少就意味着，每个元素都需要记录使用的次数，并且是有序的，同理也可以指定需要删除的数量即可，因为被标记为使用次数最少的元素会从缓存中淘汰点。在这，我们会一步一步实现lfu缓存。


##### 数据结构

实际上hashtable和双链表就够了，不需要其他复杂的算法。

hashtable对应go里面的map，链表的话，可以使用`container/list`， 实现lfu的流程如下图：
![image](/img/lfu-flow.jpg)


* 缓存元素的次数按照顺序排列，每一个element里面包含了缓存的数据。
* 缓存数据指向对应的list的node，图中A/B执向了node1，node1出现的次数是1次。

 第一个数据结构， 这是实际存储缓存的数据，key和value，frequencyParent代表这个数据出现的频次。
```go
type CacheItem struct {
	key             string        // Key of item
	value           interface{}   // Value of item
	frequencyParent *list.Element // Pointer to parent in frequency list
}
```

第二个数据结构，这是数据出现频次和当前频次下有多少元素，比如A/B属于出现一次的。
```go
type FrequencyItem struct {
	entries map[*CacheItem]byte // Set of entries
	freq    int                 // Access frequency
}
```

第三个数据结构，cache对象，保存了lfu的数据，通过bykey这个map可以直接找到缓存的数据，freqs是lfu缓存的频次分布，capacity容量，size当前使用的 情况。
```go
type Cache struct {
	bykey    map[string]*CacheItem // Hashmap containing for O(1) access
	freqs    *list.List            // Linked list of frequencies
	capacity int                   // Max number of items
	size     int                   // Current size of cache
}
```


##### 初始化/获取/设置

首先需要构建cache

```go
func New() *Cache {
	cache := new(Cache)
	cache.bykey = make(map[string]*CacheItem)
	cache.freqs = list.New()
	cache.size = 0
	cache.capacity = 100

	return cache
}
```

这个函数将构建一个新的Cache, 初始化了使用了默认的参数，并且直接使用的
list包，其提供了完整的double list功能。


第二个函数，实现Cache的Set方法
```go
func (cache *Cache) Set(key string, value interface{}) {
	if item, ok := cache.bykey[key]; ok {
		item.value = value
		cache.Increment(item)
	} else {
		item := new(CacheItem)
		item.key = key
		item.value = value
		cache.bykey[key] = item
		cache.size++
		if cache.atCapacity() {
			cache.Evict(e.capacity - 10)
		}
		cache.Increment(item)
	}
}
```

get, 通过key获取缓存元素，这里会有次数的变化，所以需要调用Increment，维护cache的变化。
```go
func (cache *Cache) Get(key string) interface{} {
	if item, ok := cache.bykey[key]; ok {
		cache.Increment(item)
		return item.value
	}

	return nil
}
```

increment，维护了缓存元素的频次，确保cache正常。
```go
func (cache *Cache) Increment(item *CacheItem) {
	currentFrequency := item.frequencyParent
	var nextFrequencyAmount int
	var nextFrequency *list.Element

	if currentFrequency == nil {
		nextFrequencyAmount = 1
		nextFrequency = cache.freqs.Front()
	} else {
		nextFrequencyAmount = currentFrequency.Value.(*FrequencyItem).freq + 1
		nextFrequency = currentFrequency.Next()
	}

	if nextFrequency == nil || nextFrequency.Value.(*FrequencyItem).freq != nextFrequencyAmount {
		newFrequencyItem := new(FrequencyItem)
		newFrequencyItem.freq = nextFrequencyAmount
		newFrequencyItem.entries = make(map[*CacheItem]byte)
		if currentFrequency == nil {
			nextFrequency = cache.freqs.PushFront(newFrequencyItem)
		} else {
			nextFrequency = cache.freqs.InsertAfter(newFrequencyItem, currentFrequency)
		}
	}

	item.frequencyParent = nextFrequency
	nextFrequency.Value.(*FrequencyItem).entries[item] = 1
	if currentFrequency != nil {
		cache.Remove(currentFrequency, item)
	}
}

```
* keyA/valueA第一次进入lfu的情况分析
通过`Set`函数写入k/v， 缓存数量+1， 缓存freqitem, 通过`Increment`构造出来
```
缓存数据keyA/valueA
CacheItemkeyA {
    keyA
    valueA
    FrequencyItemkeyA
}

缓存数据对应的freq，出现了一次
FrequencyItemkeyA {
	CacheItemkeyA
	1
}

cache.freqs的头部就是FrequencyItemkeyA
```

* keyA/valueA第二次进入lfu的情况分析

```
通过bykey，可以找到CacheItemkeyA

更新缓存数据对应的freq，freq+1
FrequencyItemkeyA {
	CacheItemkeyA
	2
}

cache.freqs的头部依然是FrequencyItemkeyA
```

删除和更新的操作基本雷同，可以参照流程图和代码来验证。如果还想更加深入的理解可以尝试与我联系。

当然这里没有考虑并发的问题，读者可以使用sync.Mutex实现。

####  LRU
设计思路是，使用 HashMap 存储 key，这样可以做到 save 和 get key的时间都是 O(1)，而 HashMap 的 Value 指向双向链表实现的 LRU 的 Node 节点，如图所示。

![image](/img/lru.jpg)

* 最后进来的元素放在dll的头部
* 如果超过了dll的size则删除尾部的元素，对应图中就是0。

##### cacheItem
```go
type cacheItem struct{
	 key string
	 value string
}
```
* 定义dll中的ele
  

##### cache
```go
type Cache struct  {
	byKey map[string]*list.Element
	ll *list.List
	cap int
	size int
}
```

* byKey只需要O(1)的查找
* ll维护缓存的前后关系
* cap容量
* size当前使用的大小

###### new， set/get
```go
func (cache *Cache) New() *Cache{
	return &Cache{
		byKey: make(map[string]*list.Element),
		ll:  list.New(),
		cap: 100,
		size: 0,
	}
}
```
* 初始化

```go
func (cache * Cache) Get(key string) (value string) {
	if item, ok:=cache.byKey[key]; ok {
		cache.ll.MoveToFront(item)
		return item.Value.(*cacheItem).value
	}
	return
}
```
* 寻找缓存数据，如果有则将其移动到ll的前面
* 返回value
* 找不到就返回空字符串

```go
func (cache * Cache) Set(key string, value string){
	if item, ok := cache.byKey[key]; ok {
		cache.ll.MoveToFront(item)
		e := item.Value.(*cacheItem)
		e.value =  value
		return
	}else{
		newCacheItem :=new(cacheItem)
		newCacheItem.key = key
		newCacheItem.key = value
		cache.size++
		if cache.size > cache.cap{
			cache.remove()
		}
		ele := cache.ll.PushFront(newCacheItem)
		cache.byKey[key] = ele
		return
	}
	return
}
```
* 对已经存在的数据做更新，并且将其移动到ll的前面
* 新增元素，size加一，如果超过了cache的cap则删除ll尾部的
* 新增元素新加到ll的头部


这里，没有考虑并发的情况，感兴趣的可以自己添加锁，并且可以实现更多的方法，比如驱逐等。另外cache cap的控制是计算的value的和，而不是其所占内存【bytes】的大小，这一点也是不够完美的，读者可以试着自己实现 ，或者跟我联系。

## 总结

* https://ieftimov.com/post/when-why-least-frequently-used-cache-implementation-golang/
* https://www.geeksforgeeks.org/doubly-linked-list/
* https://geektutu.com/post/geecache-day1.html