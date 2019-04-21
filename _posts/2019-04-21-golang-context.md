---
layout:     post
title:      Golang Context 包
subtitle:   "Golang Context Package"
date:       2019-4-21
author:     "Shunda"
header-style: text
catalog:    true
tags:
    - Go
---

## 背景和使用场景
- 1.7 版本 `golang.org/x/net/context` 包被加入到了官方的库中
- 简化单个请求的多个 goroutine 之间数据共享、取消信号、截止时间等相关操作
- 在上下文之间共享数据，比如用户身份信息、认证token、trace_id、请求截止时间等
- 如果请求超时或者被取消后，所有的 goroutine 都应该马上退出并且释放相关的资源

## Context 定义

```c
type Context interface {
    // 用于返回超时时间，如果没有超时时间，则ok返回false
    Deadline() (deadline time.Time, ok bool)
    // 返回一个通道对象，当 context 取消的时候会将该通道关闭
    // 如果本 context 还不支持取消功能，则返回nil
    Done() <-chan struct{}
    // 在 Done 通道关闭后获取 error 信息，在此之前返回nil
    // Canceled: 被取消
    // DeadlineExceeded: 超时
    Err() error
    //获取键值对信息
    Value(key interface{}) interface{}
}
```

通过 `context` 通知程序退出的例子:
```c
// Stream generates values with DoSomething and sends them to out
// until DoSomething returns an error or ctx.Done is closed.
func Stream(ctx context.Context, out chan<- Value) error {
    for {
        v, err := DoSomething(ctx)
        if err != nil {
            return err
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case out <- v:
        } 
    }
}
```

### 四种基础对象

```c
// 空对象
type emptyCtx int

// 包含键值对的 context 对象
type valueCtx struct {
   Context
   key, val interface{}
}

// 带有取消功能的 context 对象
type cancelCtx struct {
    Context
    mu       sync.Mutex          
    done     chan struct{}        
    children map[canceler]struct{} 
    err      error                
}

// 带超时功能的 context 对象
type timerCtx struct {
    cancelCtx
    timer *time.Timer 
    deadline time.Time
}
```

### 四种衍生函数
- 通过 `With*` 函数可以衍生出子 Context

```c
// 传递一个父 Context 作为参数，返回子 Context，以及一个取消函数用来取消 Context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// 多传递一个截止时间参数，到了这个时间点，会自动取消 Context
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

// 超时自动取消
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// 生成一个绑定了一个键值对数据的 Context
func WithValue(parent Context, key, val interface{}) Context
```

## 各个基础对象和 With 函数详解

### 空对象 (emptyCtx)

- 空对象不支持超时，没有键值对，不支持取消，Done函数返回为空

```c
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}
```

- 通过 `context.Background()` 或者 `context.TODO()` 可以获取空对象

```c
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

### 键值对对象（valueCtx）

```c
type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) String() string {
	return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	// 通过key递归查找value值
	return c.Context.Value(key)
}
```

#### WithValue
- 提供的 key 必须是可比较的
- 不应该使用 string 类型或者任何内建类型作为 key，以避免在不同 package 之间使用 context 时产生冲突

```c
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflect.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

```c
package user

import "context"

// User is the type of value stored in the Contexts.
type User struct {...}

// key is an unexported type for keys defined in this package.
// This prevents collisions with keys defined in other packages.
type key int

// userKey is the key for user.User values in Contexts. It is
// unexported; clients use user.NewContext and user.FromContext
// instead of using this key directly.
var userKey key

// NewContext returns a new Context that carries value u.
func NewContext(ctx context.Context, u *User) context.Context {
    return context.WithValue(ctx, userKey, u)
}

// FromContext returns the User value stored in ctx, if any.
func FromContext(ctx context.Context) (*User, bool) {
    u, ok := ctx.Value(userKey).(*User)
    return u, ok
}
```

### 取消对象（cancelCtx）
- 该对象在context对象基础上为context对象增加了取消功能

```c
type cancelCtx struct {
    Context

    mu       sync.Mutex            // protects following fields
    done     chan struct{}         // created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}

// 创建了一个无缓存的chanel对象并返回
func (c *cancelCtx) Done() <-chan struct{} {
    c.mu.Lock()
    if c.done == nil {
        c.done = make(chan struct{})
    }
    d := c.done
    c.mu.Unlock()
    return d
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}

func (c *cancelCtx) String() string {
    return fmt.Sprintf("%v.WithCancel", c.Context)
}

// cancel 关闭 c.done, 同时取消所有 c 的子 context
// removeFromParent 为 true 时，将 c 从它的父 context 中移除
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        // err不等于空说明已经被取消了，直接返回
        c.mu.Unlock()
        return // already canceled
    }
    c.err = err
    if c.done == nil {
        // closedchan 为已经关闭的 chan 对象
        c.done = closedchan
    } else {
        // 通过关闭 channel，将消息通知给其他 goroutine
        close(c.done)
    }
    for child := range c.children {
        // 该对象的子对象也依次取消
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    // 从父对象中将子对象删除，为了避免父对象取消的时候，已经取消过的子对象又一次执行取消操作。
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```
- 从父 Context 中删除:

```c
// removeChild removes a context from its parent.
func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}
```

#### WithCancel

```c
// 该函数从父类 context 对象中，生成一个子类对象，该对象带取消功能，
// 返回值中 cancel 就是该对象取消的时候的执行入口
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}

func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent}
}

// propagateCancel 使得父 context 对象被取消时，子对象也一起被取消
func propagateCancel(parent Context, child canceler) {
    // 父类不支持取消功能
    if parent.Done() == nil {
        return
    }
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            // 父类已经取消，则子类直接执行取消处理函数
            child.cancel(false, p.err)
        } else {
            // 将子类放入父类的children列表中
            if p.children == nil {
            p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
                case <-child.Done():
            }
        }()
    }
}

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    for {
        switch c := parent.(type) {
        case *cancelCtx:
            return c, true
        case *timerCtx:
            return &c.cancelCtx, true
        case *valueCtx:
            parent = c.Context
        default:
            //非cancelctx和timerctx类的时候才会是false
            return nil, false
        }
    }
}
```
### 超时对象（timerCtx）
- 超时对象就是在取消对象的基础上新增了一个超时配置以及一个定时器, 用于时间到的时候触发cancel操作。

```c
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) String() string {
    return fmt.Sprintf("%v.WithDeadline(%s [%s])", c.cancelCtx.Context, c.deadline, time.Until(c.deadline))
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

#### WithDeadline
```c
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    // 如果父对象也是超时对象，并且超时时间比d还要早，则直接封装父类对象并返回。
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    // 将c放入 parent 对象的 child 中
    propagateCancel(parent, c)
    dur := time.Until(d)
    // 已经超过了 deadline
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded)
        return c, func() { c.cancel(true, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            //创建一个定时器，到期后执行取消操作
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

#### WithTimeout
- 包装了一下 WithDeadline:

```c
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

## 使用原则
- 不要把 Context 放在结构体中，要以参数的方式传递，parent Context 一般为 Background
- 应该要把 Context 作为第一个参数传递给入口请求和出口请求链路上的每一个函数，放在第一位，变量名建议都统一，如 ctx
- 给一个函数方法传递 Context 的时候，不要传递 nil，否则在 tarce 追踪的时候，就会断了连接
- Context 的 Value 相关方法应该传递必须的数据，不要什么数据都使用这个传递
- Context 是线程安全的，可以放心的在多个 goroutine 中传递
- 可以把一个 Context 对象传递给任意个数的 gorotuine，对它执行操作时，所有 goroutine 都会接收到取消信号

## 参考链接
- [go语言源码分析之context包](https://zhuanlan.zhihu.com/p/35713516)
- [Golang Context分析](https://www.jianshu.com/p/e5df3cd0708b)