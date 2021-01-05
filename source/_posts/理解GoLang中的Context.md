---
title: 理解GoLang中的Context
date: 2020-08-05 17:22:04
tags:
---

# 背景
在实际的业务开发中，经常会有碰到一个在父协程中启动一个子协程的情况。一直没有弄明白这个子协程的关闭到底是否依赖与父协程，并且应该如何与父协程进行通信。今天在此补作一笔。

# 子协程的退出问题
先说结论，子协程是不会随着父协程的结束而结束，可以通过GoLang的MPG模型来理解，在此不做赘述。具体可以通过代码来发现

```go
func main() {
    fmt.Println("main 函数 开始...")
	go func() {
		fmt.Println("父 协程 开始...")
		go func() {
			for {
				fmt.Println("子 协程 执行中...")
				timer := time.NewTimer(time.Second * 2)
				<-timer.C
			}
		}()
		time.Sleep(time.Second*5)
		fmt.Println("父 协程 退出...")
	}()
	time.Sleep(time.Second*10)
	fmt.Println("main 函数 退出")
}
```

```
main 函数 开始...
父 协程 开始...
子 协程 执行中...
子 协程 执行中...
子 协程 执行中...
父 协程 退出...
子 协程 执行中...
子 协程 执行中...
```
可以得到结论

* ```main```函数退出，所有协程退出
* 协程无父子关系，在父协程开启新的协程，若父协程退出，不会影响子协程

# 解决方案
那么如何实现父协程与子协程的通信问题呢

## Context 上下文

先看方式

```go
func main() {
    fmt.Println("main 函数 开始...")
	go func() {
		ctx, cancel := context.WithCancel(context.Background())
		defer cancel()
		fmt.Println("父 协程 开始...")
		go func(ctx context.Context) {
			for {
				for {
					select {
					case <-ctx.Done():
						fmt.Println("子 协程 接受停止信号...")
						return
					default:
						fmt.Println("子 协程 执行中...")
						timer := time.NewTimer(time.Second * 2)
						<-timer.C
					}
				}
			}
		}(ctx)
		time.Sleep(time.Second*5)
		fmt.Println("父 协程 退出...")
	}()
	time.Sleep(time.Second*10)
	fmt.Println("main 函数 退出")
}
```

```
main 函数 开始...
父 协程 开始...
子 协程 执行中...
子 协程 执行中...
子 协程 执行中...
父 协程 退出...
子 协程 接受停止信号...
main 函数 退出
```
总算了我心头一事🐶，原来```context```是这么使用的。

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

其中，
1. ```Deadline```: 返回```context.Context```被取消的时间
2. ```Done```: 返回一个Channel,这个Channel会在当前工作完成或者上下文被取消之后关闭, 多次调用```Done```会返回同一个Channel
3. ```Err```: 返回 ```context.Context``` 结束的原因，它只会在 ```Done``` 返回的 Channel 被关闭时才会返回非空的值
4. ```Value```: 从 ```context.Context``` 中获取键对应的值，对于同一个上下文来说，多次调用 ```Value``` 并传入相同的```Key``` 会返回相同的结果，该方法可以用来传递请求特定的数据

### Context 设计原理

见参考文献

## Channel 管道


# 参考文献
https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/


https://blog.csdn.net/cdq1358016946/article/details/106380790?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param