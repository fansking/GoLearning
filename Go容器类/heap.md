

[toc]

# heap

在go中，并不像java提供一个可以直接使用的容器类，而是提供一个接口，需要你实现这个接口里的方法来自定义一个堆/集合，这也虽然提高了灵活性，但是也让增加了程序员的负担（而且不得不去阅读源码，不然根本不知道怎么调用）

## container/heap

### heap接口定义

在heap.go中定义接口如下：

~~~go
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}

~~~

发现除了Push和Pop方法外，还需要实现sort接口，sort接口定义如下：

~~~go
// Package sort provides primitives for sorting slices and user-defined
// collections.
package sort

// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}

~~~

由于实现heap需要实现其接口，所以接口的功能需要清楚的写在注释中，便于程序员完成，那么我们逐一分析各个函数应该如何完成。

### 接口实现

#### heap本体

~~~go
//我们选取切片作为容器，所以定义类型intHeap为整数切片
type IntHeap []int
//在操作时都使用指针类型进行操作，防止值传递
h := &IntHeap{2, 1, 5}
~~~

#### Sort

sort接口中的三个方法需要实现，具体实现如下：

~~~go
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
~~~

这里Less函数可以决定是大根堆还是小根堆，其含义是如果为真则i在j前。

#### Push

push方法，其官方注释要求实现的功能写的很简单，事实上也确实很简单。

将x添加在数组最后即可，所以实现代码如下：

~~~go
func (h *IntHeap) Push(x interface{}) {
	// Push and Pop use pointer receivers because they modify the slice's length,
	// not just its contents.
	*h = append(*h, x.(int))
}
~~~

你可能会疑惑，如果只是实现这样得Push函数，并没有保证堆的性质啊，你只是把这个数放到了数组最后。实际上这是官方为了避免我们写太多代码而做的设计，我们在push时实际上是调用**heap.Push(h, 3)** ,这是在heap.go中的一个函数，其具体实现为：

~~~go
func Push(h Interface, x interface{}) {
	h.Push(x)
	up(h, h.Len()-1)
}
func up(h Interface, j int) {
	for {
		i := (j - 1) / 2 // parent
		if i == j || !h.Less(j, i) {
			break
		}
		h.Swap(i, j)
		j = i
	}
}
~~~

上浮的操作由官方函数up完成。

#### Pop

~~~go
func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}
~~~

由官方注释可知，我们只需要将切片变为其[0:n-1]即可，那么如何保证获取的是最小/最大值呢，查看heap.go中Pop函数可知：

~~~go
func Pop(h Interface) interface{} {
	n := h.Len() - 1
	h.Swap(0, n)
	down(h, 0, n)
	return h.Pop()
}
func down(h Interface, i0, n int) bool {
	i := i0
	for {
		j1 := 2*i + 1
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
			break
		}
		j := j1 // left child
		if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
			j = j2 // = 2*i + 2  // right child
		}
		if !h.Less(j, i) {
			break
		}
		h.Swap(i, j)
		i = j
	}
	return i > i0
}
~~~

### 小根堆实现示例

其实官方在包中已经给出规范的实现方式：

~~~go
// Copyright 2012 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// This example demonstrates an integer heap built using the heap interface.
package heap_test

import (
	"container/heap"
	"fmt"
)

// An IntHeap is a min-heap of ints.
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
	// Push and Pop use pointer receivers because they modify the slice's length,
	// not just its contents.
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

// This example inserts several ints into an IntHeap, checks the minimum,
// and removes them in order of priority.
func Example_intHeap() {
	h := &IntHeap{2, 1, 5}
	heap.Init(h)
	heap.Push(h, 3)
	fmt.Printf("minimum: %d\n", (*h)[0])
	for h.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(h))
	}
	// Output:
	// minimum: 1
	// 1 2 3 5
}

~~~

### 大小根堆实现

但是如果我们同时需要大根堆和小根堆，难道要实现两遍或定义两次吗？实际上由于heap提供了足够灵活的接口，我们可以根据自己的需求做一些更改，也比如自己封装结构体实现优先队列等等。这里我们展示如何用一个数据结构根据属性值完成大根堆和小根堆的功能

~~~go
type IntHeap struct {
	heap []int
	//true是小根堆，false是大根堆
	bool
}


func (h IntHeap) Len() int           { return len(h.heap) }
func (h IntHeap) Less(i, j int) bool {
	if h.bool{
		return h.heap[i] < h.heap[j]
	}else{
		return h.heap[i] > h.heap[j]
	}

} // 小根堆  > 大根堆
func (h IntHeap) Swap(i, j int)      { h.heap[i], h.heap[j] = h.heap[j], h.heap[i] }

func (h *IntHeap) Push(x interface{}) {
	h.heap = append(h.heap, x.(int))
}

func (h *IntHeap) Pop() interface{} {
	old := h
	n := len(old.heap)
	x := old.heap[n-1]
	h.heap = old.heap[0 : n-1]
	return x

}
~~~

当bool是true时，是一个小根堆，否则是大根堆。

