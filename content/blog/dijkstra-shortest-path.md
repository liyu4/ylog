---
title: "Dijkstra Shortest Path"
date: 2020-10-20T14:26:24+08:00
draft: true
---

# 前言
图论中有两种图，一种是有向图一种是无向图，比如微博，我关注了大V, 大V可能不关注我，这两者之间的关系是有向的。
微信中添加了一个好友，那么两个人都是朋友关系，这两者的关系是最简单的无向图。

还有一种图是加权图，图的顶点之间的边带有权重，比方说可以时间，可以是距离等物理特性。
其中一个经典的算法就是求图中某一个顶点到另外一个顶点的最短距离。这其中比较有名的算法是dijkstra shortest path。

# 算法推导
由于我家在十八线小县城，自上海出发，选择火车出行没有直达的车次，所以选择一个出行距离最近的方案是符合直觉的方案「花费的钱，还有时间，这些暂时不考虑」

[![BSjumq.png](https://s1.ax1x.com/2020/10/20/BSjumq.png)](https://imgchr.com/i/BSjumq)

| 上海 | 杭州 | 丽水 | 武汉 | 南昌 | 永修 |
|  ----  | ----  |   ----  | ----  |  ----  | ----  |
|        |  150KM | 无法直接到达| 600KM | 500KM |  无法直接到达     |

从图中可以看出，从上海出发，可以直接到达杭州，武汉，南昌，其距离分别为150KM, 600KM, 500KM, 因为离杭州最近，
所以选取其为下一个出发顶点「出发城市」


| 上海 | 杭州 | 丽水 | 武汉 | 南昌 | 永修 |
|  ----  | ----  |   ----  | ----  |  ----  | ----  |
|       |  150KM | 350KM | 600KM | 500KM |  无法直接到达     |

杭州的下一个节点是丽水，所以上海从杭州出发到丽水的距离是350KN, 并且下一个顶点是丽水


| 上海 | 杭州 | 丽水 | 武汉 | 南昌 | 永修 |
|  ----  | ----  |   ----  | ----  |  ----  | ----  |
|       |  150KM | 350KM | 600KM | 500KM |  无法直接到达     |

丽水距离武汉400KM, 所以上海到杭州到丽水到武汉这一条线路是350KM+400KM = 750KM,
这大于上海直接出发到达的线路，所以上海到武汉不更新。下一个顶点武汉。

| 上海 | 杭州 | 丽水 | 武汉 | 南昌 | 永修 |
|  ----  | ----  |   ----  | ----  |  ----  | ----  |
|       |  150KM | 350KM | 600KM | 500KM |  860KM     |

武汉可以到达永修也可以到达南昌，到达南昌的距离是600KM+300KM这比上海直接到要大，所以不更新
上海到南昌的600KM, 武汉到达永修的距离是260KM, 所以上海通过这一条路径也就是上海--》杭州--》丽水--》武汉--》永修的距离是
860KM。下一个顶点是永修。

| 上海 | 杭州 | 丽水 | 武汉 | 南昌 | 永修 |
|  ----  | ----  |   ----  | ----  |  ----  | ----  |
|       |  150KM | 350KM | 600KM | 500KM |  860KM     |

永修无出发列车，真的是惨，所以这个顶点结束。回到上一个顶点南昌。

| 上海 | 杭州 | 丽水 | 武汉 | 南昌 | 永修 |
|  ----  | ----  |   ----  | ----  |  ----  | ----  |
|       |  150KM | 350KM | 600KM | 500KM |  540KM     |

南昌永修的距离是500KM+40KM = 540KM < 860KM，表格更新到540KM, 至此所有顶点都被访问了一次。
上海到各城市的最短距离也如上表所示。

# 算法实现

经过上面的分析我们发现，整个过程可以归纳为如下步骤：

* 选取当前顶点
* 寻找当前节点相接的顶点，并且记录下其权重
* 选择相接顶点权重较低的节点作为下一个当前顶点
* 重复上面三步


```go
package main

import (
	"fmt"
	"sort"
)

// 构建加权图

type City struct {
	city2Route map[string]*Route
}

type node struct {
	name     string
	distance int64
}

type Route struct {
	name    string //当前城市包含的roters
	routers []node
}

func Instace() *City {
	return &City{city2Route: make(map[string]*Route)}
}

func (c *City) addCity(name string) *Route {
	c.city2Route[name] = &Route{name: name}
	return c.city2Route[name]
}

func (r *Route) addRoute(name string, distance int64) {
	if name == "" {
		return
	}
	if r != nil {
		r.routers = append(r.routers, node{name, distance})
		return
	}
	newRoute := new(Route)
	newRoute.routers = append(newRoute.routers, node{name, distance})
	r = newRoute
	return
}

var instance = Instace()

func build() {
	shr := instance.addCity("sh")
	hzr := instance.addCity("hz")
	lsr := instance.addCity("ls")
	whr := instance.addCity("wh")
	ncr := instance.addCity("nc")
	yxr := instance.addCity("yx")

	shr.addRoute("hz", 150)
	shr.addRoute("wh", 600)
	shr.addRoute("nc", 500)

	hzr.addRoute("ls", 200)

	lsr.addRoute("wh", 400)

	whr.addRoute("nc", 300)
	whr.addRoute("yx", 260)

	ncr.addRoute("yx", 40)

	yxr.addRoute("", -1)

}

func main() {
	build()
	depart := "sh"
	startCity := instance.city2Route[depart]
	if startCity == nil {
		panic("not existed city")
	}

	// visited := make(map[string]struct{})

	from2DestDistance := make(map[string]int64)
	priority := make([]string, 0)
	priority = append(priority, startCity.name)

	for {
		// 维护一个访问队列，
		// 从优先级最高的队列开始，
		// 并且更新此队列，如果没有更新的节点，
		// 则从访问队列拿一个节点出来访问，直到所有的节点都被访问到
		// 从队列中选取一个
		len := len(priority)
		if len == 0 {
			break
		}

		p := priority[len-1]
		priority = priority[:len-1]
		currentCity := instance.city2Route[p]
		for _, node := range currentCity.routers {
			newly := from2DestDistance[currentCity.name] + node.distance
			if value, ok := from2DestDistance[node.name]; ok {
				if value > newly {
					from2DestDistance[node.name] = newly
				}
			} else {
				from2DestDistance[node.name] = newly
			}
		}

		sort.Slice(currentCity.routers, func(i, j int) bool {
			return currentCity.routers[i].distance > currentCity.routers[j].distance
		})

		for _, route := range currentCity.routers {
			priority = append(priority, route.name)
		}
	}

	// TODO  优先级队列会出现重复的情况，需要去重和优先级提升
	for key, each := range from2DestDistance {
		fmt.Printf("from: %v to dest city: %v, nearest distance:%vkm\n", depart, key, each)
	}
}

// from: sh to dest city: hz, nearest distance:150km
// from: sh to dest city: wh, nearest distance:600km
// from: sh to dest city: nc, nearest distance:500km
// from: sh to dest city: ls, nearest distance:350km
// from: sh to dest city: yx, nearest distance:540km

```

# 总结
这个算法还是比较有趣的，每一个顶点被访问之后，就说明遍历结束，有意思的是，顶点是有可能冲突的，需要注意冲突之后的upgrade。
另外算法模型也是比较清晰的，可以使用递归来实现，但是需要注意深度的问题。