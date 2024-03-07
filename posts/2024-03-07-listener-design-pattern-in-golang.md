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

首先，我们需要一个能够监听 ETCD 变化的注册中心：

```go
type EtcdRegistry struct {
    Registry
    cli *clientv3.Client
}

func NewEtcdRegistry(endpoints []string) *EtcdRegistry {
    cli, err := clientv3.New(clientv3.Config{
        Endpoints: endpoints,
    })
    if err != nil {
        log.Fatal(err)
    }

    return &EtcdRegistry{cli: cli}
}

func (er *EtcdRegistry) Watch(key string) {
    rch := er.cli.Watch(context.Background(), key, clientv3.WithPrefix())
    for wresp := range rch {
        for _, ev := range wresp.Events {
            er.NotifyAll(string(ev.Kv.Key), string(ev.Kv.Value))
        }
    }
}
```

我们定义一个监听器来处理配置更新：

```go
type ConfigUpdater struct{}

func (cu *ConfigUpdater) Update(key, value string) {
    fmt.Printf("配置更新 - Key: %s, New Value: %s\n", key, value)
    // 这里可以添加你的配置更新逻辑
}
```

最后，将一切串联起来：

```go
func main() {
    registry := NewEtcdRegistry([]string{"localhost:2379"})
    updater := &ConfigUpdater{}

    registry.Register(updater)
    registry.Watch("/config")
}
```

这个案例演示了如何使用监听器模式监听 ETCD 中的 key 和子 key 的变化，并处理配置更新逻辑。通过使用 Go 的特性，我们不仅能够实现这一功能，还能

保证其高效并发执行。

