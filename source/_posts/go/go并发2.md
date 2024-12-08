---
title: GO语言事务与并发（2）
date: 2024-09-22 19:11:04
tags: GO
categories: GO
---

## 一、Channel并发控制

通过创建一个切片channel 控制多个携程地并发执行，并收集携程执行获取的数据及错误信息

```go
type ResultDto struct {
    Err  error
    Data interface{}
}
func main() {
    channel := make([]chan *ResultDto, 10) 
    for i := 0; i < 10; i++ {
        channel[i] = make(chan *ResultDto)
        temp := i
        go Process(temp, channel[i])
    }
    for _, ch := range channel {
        fmt.Println(<-ch)
    }
}
func Process(i int, ch chan *ResultDto) {
    // Do some work...
    if i == 1 {
        ch <- &ResultDto{Err: errors.New("do work err")}
    } else {
        ch <- &ResultDto{Data: i}
    }
}
```

### 1.2 channel控制并发数量

通过带缓冲区的channel控制并发执行携程的数量 , 注意这里需要配合 `sync.WaitGroup` 一起使用，不然当执行到i为7 8 9 时，子携程还没有执行完，主携程就退出了

```go
func main() {
    wg := &sync.WaitGroup{}
    ch := make(chan struct{}, 3)
     
    for i := 0; i < 10; i++ {
        ch <- struct{}{}
        wg.Add(1)
         
        // 执行携程
        temp := i
        go Process(wg, temp, ch)
         
    }
     
    wg.Wait()
}
func Process(wg *sync.WaitGroup, i int, ch chan struct{}) {
    defer func() {
        <-ch
        wg.Done()
    }()
     
    // Do some work...
    time.Sleep(1 * time.Second)
    fmt.Println(i)
}
```

## 二、WaitGroup并发控制

### 2.1 WaitGroup 控制协程并行

WaitGroup是Golang应用开发过程中经常使用的并发控制技术。

WaitGroup，可理解为Wait-Goroutine-Group，即等待一组goroutine结束。比如某个goroutine需要等待其他几个goroutine全部完成，那么使用WaitGroup可以轻松实现。

```go
func main() {
    wg := &sync.WaitGroup{}
    for i := 0; i < 10; i++ {
        wg.Add(1)
        temp := i
        go Process(wg, temp)
    }
    wg.Wait()
}

func Process(wg *sync.WaitGroup, i int) {
    defer func() {
        wg.Done()
    }()
    // Do some work...
    time.Sleep(1 * time.Second)
    fmt.Println(i)
}
```

简单的说，上面程序中wg内部维护了一个计数器：

- 启动goroutine前将计数器通过Add(2)将计数器设置为待启动的goroutine个数。
- 启动goroutine后，使用Wait()方法阻塞自己，等待计数器变为0。
- 每个goroutine执行结束通过Done()方法将计数器减1。
- 计数器变为0后，阻塞的goroutine被唤醒。

### 2.2 WaitGroup封装通用函数

waitGroup控制并发执行，limit 并发上限，收集错误返回

```go
func main() {
    funcList := []ExeFunc{
        func(ctx context.Context) error {
            fmt.Println("5 开始")
            time.Sleep(5 * time.Second)
            fmt.Println("5 结束")
            return nil
        },
        func(ctx context.Context) error {
            fmt.Println("3 开始")
            time.Sleep(3 * time.Second)
            fmt.Println("3 结束")
            return nil
        },
    }
    err := GoExeAll(context.Background(), 2,  funcList...)
    if err != nil {
        fmt.Println(err)
    }
}

type ExeFunc func(ctx context.Context) error

// GoExeAll 并发执行所有，limit 为并发上限，收集所有错误返回
func GoExeAll(ctx context.Context, limit int, fs ...ExeFunc) (errs []error) {
    wg := &sync.WaitGroup{}
    ch := make(chan struct{}, limit)
    errCh := make(chan error, len(fs))
    for _, f := range fs {
        fTmp := f
        wg.Add(1)
        ch <- struct{}{}
        go func() {
            defer func() {
                if panicErr := recover(); panicErr != nil {
                    errCh <- errors.New("execution panic:" + fmt.Sprintf("%v", panicErr))
                }
                wg.Done()
                <-ch
            }()
            if err := fTmp(ctx); err != nil {
                errCh <- err
            }
        }()
    }
    wg.Wait()
    close(errCh)
    close(ch)
    for chErr := range errCh {
        errs = append(errs, chErr)
    }
    return
}
```

## 三、Context

Golang context是Golang应用开发常用的并发控制技术，它与WaitGroup最大的不同点是context对于派生goroutine有更强的控制力，它可以控制多级的goroutine。

### 3.1 Context定义的接口

context实际上只定义了接口，凡是实现该接口的类都可称为是一种context，官方包中实现了几个常用的context，分别可用于不同的场景。

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

**Deadline()**

该方法返回一个deadline和标识是否已设置deadline的bool值，如果没有设置deadline，则ok == false，此时deadline为一个初始值的time.Time值

**Done()**

该方法返回一个channel，需要在select-case语句中使用，如”case <-context.Done():”。

当context关闭后，Done()返回一个被关闭的管道，关闭的管道仍然是可读的，据此goroutine可以收到关闭请求；当context还未关闭时，Done()返回nil。

**Err()**

该方法描述context关闭的原因。关闭原因由context实现控制，不需要用户设置。比如Deadline context，关闭原因可能是因为deadline，也可能提前被主动关闭，那么关闭原因就会不同:

**Value()**

有一种context，它不是用于控制呈树状分布的goroutine，而是用于在树状分布的goroutine间传递信息

### 3.2 Context控制协程结束

```go
func main() {
    wg := &sync.WaitGroup{}
    ctx, cancelFunc := context.WithCancel(context.Background())
    for i := 0; i < 10; i++ {
        wg.Add(1)
        temp := i
        go Process(ctx, wg, temp)
    }
    time.Sleep(5 * time.Second)
    cancelFunc()
    wg.Wait()
}

func Process(ctx context.Context, wg *sync.WaitGroup, i int) {
    defer wg.Done()
    ch := make(chan error)
    go DoWork(ctx, ch, i)
    select {
    case <-ctx.Done():
        fmt.Println("cancelFunc")
        return
    case <-ch:
        return
    }
}

func DoWork(ctx context.Context, ch chan error, i int) {
    defer func() {
        ch <- nil
    }()
    time.Sleep(time.Duration(i) * time.Second)
    fmt.Println(i)
}
```

## 四、 ErrorGroup

可采用第三方库`golang.org/x/sync/errgroup`堆多个协助并发执行进行控制

**4.1 errorGroup并发执行，limit 为并发上限，timeout超时**

```go
func main() {
    funcList := []ExeFunc{
        func(ctx context.Context) error {
            fmt.Println("5 开始")
            time.Sleep(5 * time.Second)
            fmt.Println("5 结束")
            return nil
        },
        func(ctx context.Context) error {
            fmt.Println("3 开始")
            time.Sleep(3 * time.Second)
            fmt.Println("3 结束")
            return nil
        },
    }
    err := GoExe(context.Background(), 2, 10*time.Second, funcList...)
    if err != nil {
        fmt.Println(err)
    }
}

type ExeFunc func(ctx context.Context) error

// GoExe 并发执行，limit 为并发上限，其中任意一个报错，其他中断，timeout为0不超时
func GoExe(ctx context.Context, limit int, timeout time.Duration, fs ...ExeFunc) error {
    eg, ctx := errgroup.WithContext(ctx)
    eg.SetLimit(limit)
    var timeCh <-chan time.Time
    if timeout > 0 {
        timeCh = time.After(timeout)
    }
    for _, f := range fs {
        fTmp := f
        eg.Go(func() (err error) {
            ch := make(chan error)
            defer close(ch)
            go DoWorkFunc(ctx, ch, fTmp)
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-timeCh:
                return errors.New("execution timeout")
            case err = <-ch:
                return err
            }
        })
    }
    if err := eg.Wait(); err != nil {
        return err
    }
    return nil
}

func DoWorkFunc(ctx context.Context, ch chan error, fs ExeFunc) {
    var err error
    defer func() {
        if panicErr := recover(); panicErr != nil {
            err = errors.New("execution panic:" + fmt.Sprintf("%v", panicErr))
        }
        ch <- err
    }()
    err = fs(ctx)
    return
}
```

## 五、通用协程控制工具封装

```go
import (
    "context"
    "errors"
    "fmt"
    "golang.org/x/sync/errgroup"
    "sync"
    "time"
)


// ExeFunc 要被执行的函数或方法
type ExeFunc func(ctx context.Context) error

// SeqExe 顺序执行，遇到错误就返回
func SeqExe(ctx context.Context, fs ...ExeFunc) error {
    for _, f := range fs {
        if err := f(ctx); err != nil {
            return err
        }
    }
    return nil
}

// GoExe 并发执行，limit 为并发上限，其中任意一个报错，其他中断，timeout为0不超时
func GoExe(ctx context.Context, limit int, timeout time.Duration, fs ...ExeFunc) error {
    eg, ctx := errgroup.WithContext(ctx)
    eg.SetLimit(limit)
    var timeCh <-chan time.Time
    if timeout > 0 {
        timeCh = time.After(timeout)
    }
    for _, f := range fs {
        fTmp := f
        eg.Go(func() (err error) {
            ch := make(chan error)
            defer close(ch)
            go DoWorkFunc(ctx, ch, fTmp)
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-timeCh:
                return errors.New("execution timeout")
            case err = <-ch:
                return err
            }
        })
    }
    if err := eg.Wait(); err != nil {
        return err
    }
    return nil
}

func DoWorkFunc(ctx context.Context, ch chan error, fs ExeFunc) {
    var err error
    defer func() {
        if panicErr := recover(); panicErr != nil {
            err = errors.New("execution panic:" + fmt.Sprintf("%v", panicErr))
        }
        ch <- err
    }()
    err = fs(ctx)
    return
}

// SeqExeAll 顺序执行所有，收集所有错误返回
func SeqExeAll(ctx context.Context, fs ...ExeFunc) (errs []error) {
    for _, f := range fs {
        if err := f(ctx); err != nil {
            errs = append(errs, err)
        }
    }
    return errs
}
// GoExeAll 并发执行所有，limit 为并发上限，收集所有错误返回
func GoExeAll(ctx context.Context, limit int, fs ...ExeFunc) (errs []error) {
    wg := &sync.WaitGroup{}
    ch := make(chan struct{}, limit)
    errCh := make(chan error, len(fs))
    for _, f := range fs {
        fTmp := f
        wg.Add(1)
        ch <- struct{}{}
        go func() {
            defer func() {
                if panicErr := recover(); panicErr != nil {
                    errCh <- errors.New("execution panic:" + fmt.Sprintf("%v", panicErr))
                }
                wg.Done()
                <-ch
            }()
            if err := fTmp(ctx); err != nil {
                errCh <- err
            }
        }()
    }
    wg.Wait()
    close(errCh)
    close(ch)
    for chErr := range errCh {
        errs = append(errs, chErr)
    }
    return
}
```

