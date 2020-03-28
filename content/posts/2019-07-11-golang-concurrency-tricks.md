---
title: Golang concurrency tricks
date: 2019-07-11 21:35:44
tags: [golang]
---

看了一些 goroutine 的 patterns，（今天）觉得 goroutine 用起来是一种很爽的感觉

## Channel over channel

Q：`chan chan string` 这样的数据类型的作用？

在 [ref-1](#ref-1) 的视频中使用到这一类型，用于 request & get response.

示例代码见 [gist](https://gist.github.com/inhzus/d7ca93a0f9d605d41e75008621728814)，能想到的应用： request 停止信号，response 错误信息。

## Nil channel

这一 trick 在 [ref-1](#ref-1) 中也讲到了，但不清楚有什么比较实用的做法，看起来还是有更好的替代方法。

示例代码见 [slide code](https://talks.golang.org/2013/advconc.slide#29)

<!--more-->

## Channel as shuttle

有时想让某个 channel 满员后全部发送。

数量比较少的情况下

```go
c := make(chan int, 2)
go routine(c, ...)
go routine(c, ...)
result := []int{<-c, <-c}
```

这种简单的方法在数量稍大些的情况下就变得非常的不优雅了，使用：

```go
count := 5
wg := &sync.WaitGroup{}
c := make(chan int, count)
wg.Add(count)
for i := 0; i < count; i++ {
    go func() {
        defer wg.Done()
        c <- i
    }()   
}
wg.Wait()
```

## Channel 负载

在工程中遇到一段代码比较有意思

```go
var (
	queue = make(chan int, 3)
	wg    = &sync.WaitGroup{}
)

func consumer() {
	for {
		task, open := <-queue
		if !open {
			break
		}
		wg.Add(1)
		go func(val int) {
			defer wg.Done()
			// some logic
		}(task)
	}
}

func producer() {
	var arr []int
	for i := 0; i < 1000000; i++ {
		arr = append(arr, i)
	}
	for _, val := range arr {
		select {
		case queue <- val:
		case <-time.After(time.Millesecond):
			fmt.Print("producer timeout")
		}
	}
}

func main() {
	go consumer()
	producer()
	wg.Wait()
}
```

之后继续学习并更新

## References

<div id="ref-1"><a href="https://blog.golang.org/advanced-go-concurrency-patterns">Video: advanced go concurrency patterns</a></div>
<div id="ref-2"><a href="https://stackoverflow.com/questions/38793573/wait-for-a-buffered-channel-to-be-full">Stackoverflow: Wait for a buffered channel to be full</a></div>
