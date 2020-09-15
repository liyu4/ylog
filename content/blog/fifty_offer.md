---
title: "剑指offer读书笔记"
date: 2020-09-14T16:40:07+08:00
draft: true
---

# 引言

剑指offer这本书，听过很多但是一直没有机会去阅读，今日在kindle上下单买了这本书，粗看了前面的几十页之后，觉得开心和兴奋。
书中的内容并不是新的，可以认为是再加工，再发现，再凝练，里面的题目也不仅仅是题目，而是对人一些思考方式的梳理和建议，总而言之
这是一本用心的佳作，哪怕是只看懂其三分之二 ，也依然是大有益处。

数据结构，有适用于内存的，也有适用于磁盘，软盘的，或者两者兼有。挑选合适的数据结构，比如，数组，链表，队列，栈，树，图，哈希表等
将解题思路换成数据结构的表达方式。有趣的地方在于，一种题型可能有多种解法，效率和方式可能都会不同，选择效率最高，内存消耗最低的，
使我们的追求，这也是有意思的地方。

# “面试”和被面试重要的东西
* 准备好项目中可能会问到的问题
* 充满热情
* 基础扎实，算法和数据结构，项目基础等。
* 想清楚再动手编程
* 注意命名，编写可测试单元
* 写出正确，完整，鲁棒性高的代码
* 注意边界，条件，nil、null，empty string，负数，超大数等
* 从时间和空间复杂度两个方面优化代码
* 沟通，学习，散发的能力


# 为什么跳槽

现在的工作做了一段时间，已经没有太多的激情了，因此希望寻找一份更有挑战的工作。
然后具体论述为什么有些厌倦现在的职位，以及面试的职位我为什么会有兴趣。

# 单例的思考, 高并发无处不在

读过设计模式，或者有过编程经验的人，都多少听过单例模式「有些书叫做单件模式」。

他的定义是：*确保一个类只有一个实例，并且提供一种全局的访问方式。*

第一种写法，这单机单线程的环境下这个实现是没有问题，但是当前多数的应用，都是并发执行，多个线程会同时访问
临界区的代码，这个时候就需要使用编程语言使用的锁原语来保证代码的串行执行。

go
```
type Singleton struct {
}

var singleton *Singleton

func GetSingleton() *Singleton s{
    if singleton  == nil {
        singleton = new(single)
    }
    return singleton
}

```

第二种写法，加锁，确定只有一个线程可以实例化singleton, 新的问题是，锁粒度太大

```
type Singleton struct {
}

var singleton *Singleton

func GetSingleton() *Singleton {
    lock()
    if singleton  == nil {
        singleton = new(single)
    }
    unlock()
    return singleton
}

```

第三种写法，锁粒度小一点, 有那味道了，但是还是会有多线程的问题，两个线程可以同时进入if singleton == nil, 还是会加锁两次，
并且实例化两次。

```
type Singleton struct {
}

var singleton *Singleton

func GetSingleton() *Singleton {
    if singleton  == nil {
        // 第三种方法这里还是可以有多个线程blocking在这里
        lock()
        singleton = new(single)
        unlock()
    }
    return singleton
}
```

第四种写法，针对第三种再改进一下。

```
type Singleton struct {
}

var singleton *Singleton

func GetSingleton() *Singleton {
    if singleton  == nil {
        lock()
        if singleton == nil { // 双重判断一次，好处就是，blocking在上一个判断的线程，不会再去实例化
            singleton = new(single)
        }
        unlock()
    }
    return singleton
}
```

第五种，编程语言提供的特性，安全高效，可阅读性好

```
type Singleton struct {
}

var singleton *Singleton
var once sync.Once

func InitSingleton() {
    once.Do(func(){
        singleton = new(single)
    })
}

```


这个例子，算是一个引子，这个过程展现了题目是什么，结题的思路。
* 清晰的题目描述，也就是问题是什么。
* 采用循序渐进的方案，发现问题，再逐个优化。
* 最终达到比较好的方案。




# 辩证和追求

# 有意思的例子

# 总结

