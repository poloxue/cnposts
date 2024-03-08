---
title: "Go 中的监视器模式与配置热加载"
date: 2024-03-07T15:46:11+08:00
draft: true
comment: true
---

上篇介绍 GO 的 GUI 库 Fyne 时，提到 Fyne 的数据绑定用到了监听器模式。本文就展开说下我对 Go 中监听器模式的理解和应用吧。

## 监听器模式简介

监听器模式，或称观察者模式，它主要涉及两个组件：主题（Subject）和监听器（Listener）。

Subject 负责维护一系列的监听器，在所观测主题状态变化，将这个事件通知给所有注册的监听器。我统一将其定义为注册中心 Registry。而监听器 Listener 则是实现了特定接口的对象，用于响应事件消息，执行处理逻辑。

这个模式在组件之间建立一种松散耦合的关系。将特定事件通知到关心它的其他组件，无需它们直接相互引用。看起来这个不也是发布-订阅模式吗？差不多一个意思。

监听器模式是事件驱动编程中常用的设计模式，帮助我们构建灵活和解耦的系统。之前的工作中，用到它最多的就是配置的热更新，这篇文章也会介绍基于它的 ETCD 配置热更新。

## Go 实现监听器模式

如何用 Go 实现监听模式？我将定义两个新类型分别是注册中心（Registry）和监听器接口（Listener）。

```go
type Listener interface {
    OnTrigger()
}
```

首先是 `Listener`，它是一个接口，用于实现事件的响应逻辑。它的实现类型要求支持 `OnTrigger` 方法，会在事件发生时被执行。

```go
type Registry struct {
    listeners []Listener
}

func (r *Registry) Register(l Listener) {
    r.listeners = append(r.listeners, l)
}

func (r *Registry) NotifyAll() {
    for _, listener := range r.listeners {
        listener.OnTrigger(key, value)
    }
}
```

`Registry` 是所有监听器的注册地，当特定事件发生，我们通过 `Registry.NotifyAll` 将事件传递给所有 `Listener`。

让我们实现一个简单的案例，当监听到某个事件发生，打印 "A specified event accured"。本案例的事件将先通过主函数调用 `NotifyAll` 模拟触发事件。

为了打印事件发生消息。创建新类型实现 `Listener`：

```go
type EventPrinter struct {
}

func (printer *EventPrinter) OnTrigger() {
  fmt.Println("A specified event accured!")
}
```

写个主函数测试下是否符合预期，代码如下所示：

```go
func main() {
  r := &Registry{}
  r.Reigster(&EventPrinter{})

  // 模拟接收到消息，触发事件通知
  r.NotifyAll()
}
```

让我们执行测试，内容如下所示：

```bash
$ go run main.go
A specified event occurred
```

如果希望自定义接受函数，只需在 `Listener` 支持自定义事件接收函数即可。

```go
type EventHandler struct {
  callback func()
}

func NewEventHandler(callback func()) *EventHandler {
  return &EventHandler{callback: callback}
}
func (e *EventHandler) OnTrigger() {
  e.callback
}
```

注册一个 `EventHandler` 到 `Registry`，主函数代码：

```go
func main() {
  r := &Registry{}
  r.Reigster(&EventPrinter{})
  r.Reigster(NewEventHandler(func() {
    fmt.Println("Custom Print: a specified event occurred!")  
  }))
  r.NotifyAll()
}
```

测试执行：

```bash
$ go run main.go
A specified event occurred!
Custom Print: a specified event occurred!
```

## 基于 Go Channel 实现并发处理

前面的示例中 `NotifyAll` 是通过 for 循环依次调用 `listener.OnTrigger` 将消息发送给 `Listener`，处理效率低下。如何加速呢？最直接的方法是通过 `goroutine` 运行 `listener.OnTrigger` 方法。

```go
func (r *Registry) NotifyAll() {
  for _, listener := range r.listeners {
      go listener.OnTrigger()
  }
}
```

还有一种方法，通过 Channel 传递事件消息，这样每个 `Listener` 有独立的 goroutine 监听和处理。

如下是 `Listener` 的实现代码：

```go
type Listener struct {
	EventChannel chan struct{}
	Callback     func()
}

func NewListener(callback func()) *Listener {
	return &Listener{
		EventChannel: make(chan struct{}, 1), // 带缓冲的 channel，防止阻塞
		Callback:     callback,
	}
}

func (l *Listener) Start() {
	go func() {
		for range l.EventChannel {
			l.Callback()
		}
	}()
}
```

这里 `Listener` 的事件处理函数在单独的 goroutine 中运行。

相应的 Registry 实现也需要修改，变更如下所示：

```go
type Registry struct {
	listeners []*Listener
}

func (r *Registry) Register(listener *Listener) {
	r.listeners = append(r.listeners, listener)
	listener.Start() // 启动监听器的 goroutine
}

func (r *Registry) NotifyAll(message string) {
	for _, listener := range r.listeners {
		listener.EventChannel <- struct{}{} // 发送事件到监听器
	}
}

func (r *Registry) Close() {
	for _, listener := range r.listeners {
		close(listener.EventChannel) // 关闭 channel，停止监听器 goroutine
	}
}
```

代码在整体上的变化不大，`Register` 中启动 `Listener` 事件处理 goroutine 等待事件消息。

## 实际案例：ETCD 配置热更新

介绍那么多监视器模式的 Go 实现方式，现在我们来一个实际的案例吧。

假设我们有一个应用，需要根据配置来动态调整行为。这些配置存储在 ETCD 中，我们想要实时监听这些配置项的变化。借助于监听器模式，我们可以轻松实现这一需求。

基于监听器模式实现 ETCD 热更新的要求，我们将设计一个系统，其中：

- 每个监听器可以订阅特定的 key 或 key 前缀。
- 使用 `channel` 来处理 ETCD 的变更事件，并触发对应的监听回调。

这个设计涉及到以下几个组件：

1. **`Watcher`**：负责监听 ETCD 中的 key 变更事件。
2. **`Listener`**：定义了当特定 key 发生变化时需要执行的回调逻辑。
3. **`Registry`**：管理所有的 `Listener`，并将 ETCD 变更事件分发给对应的 `Listener`。

### 定义 Listener 接口

每个 `Listener` 都应该实现以下接口，以便能够处理特定的 key 变更事件：

```go
type Listener interface {
    OnTrigger(key string, value string)
}
```

### 实现 Registry

`Registry` 负责维护 `Listener` 的注册，并在接收到 key 变更事件时通知相关的 `Listener`：

```go
type Registry struct {
    listeners map[string][]Listener
    eventChannel chan *Event
}

func NewRegistry() *Registry {
    return &Registry{
        listeners: make(map[string][]Listener),
        eventChannel: make(chan *Event),
    }
}

func (r *Registry) Register(key string, listener Listener) {
    r.listeners[key] = append(r.listeners[key], listener)
}

func (r *Registry) Start() {
    go func() {
        for event := range r.eventChannel {
            if listeners, ok := r.listeners[event.Key]; ok {
                for _, listener := range listeners {
                    listener.OnTrigger(event.Key, event.Value)
                }
            }
        }
    }()
}

func (r *Registry) Notify(event *Event) {
    r.eventChannel <- event
}
```

这里，`Event` 是一个简单的结构体，表示 ETCD 中 key 的变更事件：

```go
type Event struct {
    Key   string
    Value string
}
```

### 实现 Watcher

`Watcher` 负责从 ETCD 订阅 key 的变更事件，并将这些事件发送到 `Registry` 的 `eventChannel` 上：

```go
func WatchEtcdKeys(client *clientv3.Client, registry *Registry, watchKeys ...string) {
    for _, key := range watchKeys {
        go func(key string) {
            watchChan := client.Watch(context.Background(), key, clientv3.WithPrefix())
            for wresp := range watchChan {
                for _, ev := range wresp.Events {
                    event := &Event{
                        Key:   string(ev.Kv.Key),
                        Value: string(ev.Kv.Value),
                    }
                    registry.Notify(event)
                }
            }
        }(key)
    }
}
```

### 使用示例

让我们实际在 main 函数上使用一下，观察行为是否正常。

```go
func main() {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   []string{"localhost:2379"},
    })
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    registry := NewRegistry()
    registry.Start()

    // 注册监听器
    registry.Register("/config/database", NewDatabaseConfigListener())

    // 开始监听 ETCD key 变更
    WatchEtcdKeys(client, registry, "/config/")
}
```

在这个示例中，我们创建了一个 ETCD 客户端，初始化了一个 `Registry`，并为特定的 key 注册了一个 `Listener`。然后，通过 `WatchEtcdKeys` 函数开始监听 `/config/` 前缀下的所有 key 的变更。

这种设计支持对特定 key 或 key 前缀的监听，当相关的 key 发生变更时，能够通过 `channel` 触发注册的 `Listener` 的回调逻辑，从而实现配置的热更新。

特别说明，示例仅作为概念验证，实际应用中需要更多的错误处理和优化。
