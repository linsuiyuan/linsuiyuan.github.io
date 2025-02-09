---
title: "Go语言中的迭代器"
date: 2025-02-08

categories:
  - 编程语言
  - Golang
tags:
  - Go
  - 迭代器

summary: "简单介绍了迭代器，以及为何需要迭代器。列举了在 Go 语言中迭代器的一些实现方式，并重点谈论了 Go 1.23 版本的迭代器。"
---

## 1. 迭代器简介

### 1.1 定义：

维基百科上迭代器的定义：迭代器（英语：iterator），是使用户可在容器对象（container，例如链表或数组）上遍访的对象，设计人员使用此接口无需关心容器对象的内存分配的实现细节。

### 1.2 为何需要迭代器

理论上，迭代器的功能可以通过其他方法实现，那为何还需要迭代器呢？迭代器之所以存在，自然是有它的优点的：

1. **节省内存**: 迭代器不会一次性将所有数据都加载到内存中，而是按需生成数据。比如在逐行读取大文件时使用迭代器能显著减少内存的使用。
2. **性能优化**：迭代器仅在需要时生成数据，可以减少不必要的计算。比如，数据库查询结果集逐行返回，而不是一次性返回所有记录。
3. **解耦数据生成逻辑与遍历逻辑**：迭代器负责生成数据，其逻辑是独立的，不受遍历数据逻辑的影响。同样的 ，遍历迭代器时无需考虑数据生成的细节。

## 2. Go 1.23版本之前的迭代器

在 1.23 版本之前，Go 官方是没有提供传统意义上的迭代器的，如果要用到迭代器的功能，就需要自己实现。下面来说说两种比较有代表性的实现。

### 面向对象语言的迭代器

在面向对象语言中，Java 中的 `Iterator` 迭代器是比较有代表性的，下面使用 Go 来实现一个类似的迭代器。

```go
package main  
  
import "fmt"  
  
type Iterator struct {  
    data  []int  
    index int  
}  
  
func NewIterator(data []int) *Iterator {  
    return &Iterator{data: data, index: 0}  
}  
  
func (it *Iterator) HasNext() bool {  
    return it.index < len(it.data)  
}  
  
func (it *Iterator) Next() int {  
    if it.index >= len(it.data) {  
       panic("没有元素了")  
    }  
    item := it.data[it.index]  
    it.index++  
    return item  
}  
  
func main() {  
    it := NewIterator([]int{1, 2, 3, 4, 5})  
    for it.HasNext() {  
       fmt.Println(it.Next())  
    }  
}
```

这种面向对象实现的迭代器不够简洁，一般而言，在实际编程中很少自定义这样的迭代器。

### 函数形式的迭代器

Go 中使用函数形式实现的迭代器，比较有代表性的是使用协程和通道来实现的：

```go
package main  
  
import "fmt"  
  
func generator(num int) <-chan int {  
    ch := make(chan int)  
    go func() {  
       for i := 0; i < num; i++ {  
          ch <- i  
       }  
       close(ch)  
    }()  
    return ch  
}  
  
func main() {  
    for n := range generator(5) {  
       fmt.Println(n)  
    }  
}
```

这种形式的迭代器，相对面向对象的迭代器要简洁得多。不过该方式使用了协程来实现，其性能要差很多。

## 3. Go 1.23版本的迭代器

Go 1.23 版本提供了一个 iter 包，算是 Go 官方提供了一个标准的迭代器实现。

Go 1.23 的迭代器是一个函数类型，把另一个函数（ `yield` 函数）作为参数，定义迭代器时将元素传递给 `yield` 函数。迭代器函数会在以下两种情况停止。
- 在序列迭代结束后停止，表示此次迭代结束。
- 在 `yield` 函数返回 `false` 时停止，表示提前停止迭代。

下面使用 iter 包来实现函数形式的迭代器，`iter.Seq[V any]` 是 `func(yield func(V) bool)` 的简称：

```go
package main  
  
import (  
    "fmt"  
    "iter")  
  
func generator(num int) iter.Seq[int] {  
    return func(yield func(int) bool) {  
       for i := 0; i < num; i++ {  
          if !yield(i) {  
             return  
          }  
       }  
    }  
}  
  
func main() {  
    for n := range generator(5) {  
       fmt.Println(n)  
    }  
}
```

这种形式的迭代器，兼顾了性能和简洁性。

在性能方面，笔者测试了一下，使用 `yield`函数 实现的迭代器，比使用协程和通道实现的迭代器，提升了 200 多倍的性能！

简洁性是相对面向对象来说的。有些遗憾的是，相比其他的一些语言，上面的实现在简洁明了方面做得不够好。`generator` 需要返回一个函数，该函数的参数还是一个函数，还需要判断是否提前停止迭代。这样很不优雅，本来这些是需要隐藏到幕后，由语言本身实现而不是由用户去实现的。

其他语言中一个比较优雅的迭代器实现方式是 Python 的生成器函数，其方式如下：

```python
def generator(num: int):  
    for i in range(num):  
        yield i
```

上面的代码仅用3行就实现了同样的功能，而且该函数里面无需返回函数，无需判断是否需要提前停止迭代。用户只需关注如何生成数据，然后通过一个 `yield` 语句就可将数据传给遍历者。

## 4. 总结

Go 1.23版本之前，Go 官方没有提供迭代器，需要用户自己定义，很不方便。Go 1.23 开始，终于提供了官方的迭代器，弥补了 Go 语言在迭代器方面的缺失，方便了有相关需求的用户，这是可喜的。

Go 语言追求简单，少即是多的理念，这本无可厚非，不过之前的 Go 追求得有些极端了。近年来，Go 终于没那么极端了，为 Go 语言本身和 Go 标准库 添加了许多特性和功能。望 Go 早日成为主流编程语言。