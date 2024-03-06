---
title: "2024 03 01 Fyne in Golang"
date: 2024-03-02T18:42:00+08:00
draft: true
comment: true
description: ""
---

今天，推荐一个 Go 实现的 GUI 库 - fyne。

Go 一直以来都没有一个标准 GUI 库，Go 官方也没有提供。在 Go 实现的几个 GUI 库中，Fyne 算是最出色的，它有着简洁的API、支持跨平台能力，且高度可扩展。这也就是说，Fyne 是用来开发 App。

本文将尝试介绍下 Fyne，希望对大家快速上手这个 GUI 框架有所帮助。我最近产生了不少想法，其中有些是对 GUI 有要求的，就想着折腾用 Go 实现，而不是用那些已经很流行和成熟的 GUI 框架。

在写这篇文章时，顺手搞了下它的中文版文档，文档查看 [www.poloxue.com/gofyne](https://www.poloxue.com.com)，希望能帮助那些想继续深入这个框架的朋友。

## 安装 fyne

开始前，确保已成功安装 Go，如果是 MacOS X 系统，要确认安装了 xcode。

如下使用 `go get` 命令安装 Fyne。

```bash
$ mkdir hellofyne
$ cd helloyfyne
$ go mod init hellofyne
$ go get fyne.io/fyne/v2@latest
$ go install fyne.io/fyne/v2/cmd/fyne@latest
```

如果想立刻查看 fyne 提供的演示案例，通过命令检查：

```bash
$ go run fyne.io/fyne/v2/cmd/fyne_demo@latest
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-01-fyne-in-golang-04-v1.png)

看起来，这里面的案例还是不够丰富的。

安装工作到此就完成了。

Fyne 对不同系统有不同依赖，如果安装过程中遇到问题，细节可查看官方提供的[安装文档](https://www.poloxue.com/gofyne/docs/01-started/01-index/)。

## 创建第一个应用

由于 Go 简洁的语法和 fyne 的设计，使用 fyne 创建一个 GUI 应用异常简单。

以下是一个创建基础窗口的例子：

```go
package main

import (
    "fyne.io/fyne/v2/app"
    "fyne.io/fyne/v2/container"
    "fyne.io/fyne/v2/widget"
)

func main() {
    a := app.New()
    w := a.NewWindow("Hello Fyne")
    w.SetContent(widget.NewLabel("Welcome to Fyne!"))
    w.ShowAndRun()
}
```

这段代码创建了一个包含标签的窗口。

通过 `app.New()` 初始化一个 Fyne 应用实例。然后，创建一个标题为 "Hello Fyne" 的窗口，并设置内容为包含 "Welcome to Fyne!" 文本标签。最后，通过`w.ShowAndRun()`显示窗口并启动应用的事件循环。

fyne 中窗口的默认大小是其包含的内容决定，也可预置大小。

如在创建窗口后，提供 `Resize` 重新调整大小。

```go
w.Resize(fyne.NewSize(100, 100))
```

还有，一个 app 下可以有多个窗口的，示例代码：

```go
btn := widget.NewButton("Create a widnow", func() {
	w2 := a.NewWindow("Window 02")
	w2.Resize(fyne.NewSize(200, 200))
	w2.Show()
})

w.SetContent(btn)
```

我们创建一个按钮，它的点击事件是创建一个新的窗口并显示。

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-01-fyne-in-golang-08.gif)

## 布局和控件

布局和控件是 GUI 应用程序设计中必不可少的两类组件。Fyne 提供了多种布局管理器和标准的 UI 控件，支持创建更复杂的界面。

### 布局管理

Fyne 中的布局的实现位于 `container` 包中。它提供了多种不同布局方式安排窗口中的元素。最基本的布局方式有 `HBox` 水平布局和 `VBox` 垂直布局。

通过 `HBox` 创建水平布局的代码如下：

```go
w.SetContent(container.NewHBox(widget.NewButton("Left", func() {
	fmt.Println("Left button clicked")
}), widget.NewButton("Right", func() {
	fmt.Println("Right button clicked")
})))
```

显示效果：

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-01-fyne-in-golang-09.png)

通过 `VBox` 创建垂直布局的例子。

```go
w.SetContent(container.NewVBox(widget.NewButton("Top", func() {
	fmt.Println("Top button clicked")
}), widget.NewButton("Bottom", func() {
	fmt.Println("Bottom button clicked")
})))
```

显示效果：

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-01-fyne-in-golang-10.png)

Fyne 除了基础的水平（`HBoxLayout`）和垂直（`VBoxLayout`）布局，还提供了`GridLayout`、`FormLayout` 甚至是混合布局 C`ombinedLayout` 等高级布局方式。

如 `GridLayout`可以将元素均匀地分布在网格中，而`FormLayout`适用于创建表单，自动排列标签和字段。这些灵活的布局选项支持创建更复杂和功能丰富的GUI界面。

官方文档中的布局列表查看：[Layout List](https://www.poloxue.com/gofyne)。

### 更多控件

前面的演示案例中，用到了两个控件：Label 和 Button，Fyne 还支持其他多种控件，它们都为于 `widget` 包中。

我尝试在一份代码中展示出来，如下是常见控件一览：

```go
// 标签 Label
label := widget.NewLabel("Label")
// 按钮 Button
button := widget.NewButton("Button", func() {})
// 输入框 Entry
entry := widget.NewEntry()
entry.SetPlaceHolder("Entry")
// 复选框 Check
check := widget.NewCheck("Check", func(bool) {})
// 单选框 Check
radio := widget.NewRadioGroup([]string{"Option 1", "Option 2"}, func(string) {})
// 选择框
selectEntry := widget.NewSelectEntry([]string{"Option A", "Option B"}
// 进度条
progressBar := widget.NewProgressBar()
// 滑块
slider := widget.NewSlider(0, 100)
// 组合框
combo := widget.NewSelect([]string{"Option A", "Option B", "Option C"}, func(string) {})
// 表单项
formItem := widget.NewFormItem("FormItem", widget.NewEntry())
form := widget.NewForm(formItem)
// 手风琴
accordion := widget.NewAccordion(widget.NewAccordionItem("Accordion", widget.NewLabel("Content")))
// Tab 选择
tabs := container.NewAppTabs(
	container.NewTabItem("Tab 1", widget.NewLabel("Content 1")),
	container.NewTabItem("Tab 2", widget.NewLabel("Content 2")),
)

// 弹出对话框示例按钮
dialogButton := widget.NewButton("Show Dialog", func() {
	dialog.ShowInformation("Dialog", "Dialog Content", w)
})

// 滚动布局
content := container.NewVScroll(container.NewVBox(
	label, button, entry, check, radio, selectEntry, progressBar, slider,
	combo, form, accordion, tabs, dialogButton,
))
w.SetContent(content)
```

演示效果：

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-01-fyne-in-golang-12.gif)

## 自定义组件

在实际项目中使用 Fyne，基本上是要自定义控件、布局或者使用数据绑定功能的。自定义控件用于创建符合需求的控件。Fyne 提供了自定义控件、布局和主题等。

### 自定义控件

fyne 是支持实现自定义控件的，这涉及定义控件的绘制方法和布局逻辑，我们主要是实现两个接口：`fyne.Widget` 和 `fyne.WidgetRenderer`。

`fyne.Widget` 的定义如下所示：

```go
type Widget interface {
	CanvasObject
	CreateRenderer() WidgetRenderer
}
```

而其中返回的就是 `WiddgetRenderer`，用于定义控件布局渲染的逻辑。

```go
type WidgetRenderer interface {
	Destroy()
	Layout(Size)
	MinSize() Size
	Objects() []CanvasObject
	Refresh()
}
```

现在我要实现一个类似 Label 的控件，如下是控件定义：

```go
type CustomLabel struct {
    widget.BaseWidget
    Text              string
}
```

它继承了 `wiget.BaseWidget` 的基本控件实现，我们只要实现 `CreateRenderer` 方法即可。

定义 `customWidgetRenderer` 类型：

```go
type customWidgetRenderer struct {
    text  *canvas.Text // 使用canvas.Text来绘制文本
    owner *CustomWidget
}
```

## 数据绑定

Fyne 的数据绑定功能允许控件直接与数据源连接，数据的任何更改都会自动反映在UI上，反之亦然。

## 结语

Fyne 以其简单、强大和跨平台的特性，为我们使用 GO 开发现代 GUI 应用提供了一个优秀的选择。无论你是初学者还是有经验的开发者，Fyne都值得一试。随着对 Fyne 的深入，你将能够更加灵活地构建出符合需求的应用，享受Go语言开发GUI带来的乐趣。

