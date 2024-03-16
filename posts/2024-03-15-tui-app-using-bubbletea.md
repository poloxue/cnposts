---
title: "基于 bubbletea 生态开发 TUI 应用"
date: 2024-03-15T08:50:00+08:00
draft: true
comment: true
---

我对 TUI 的兴趣源于一个名为 LazyGit 的终端命令，它让我只需几次按键即可完成一次 Git 提交。

不同于 GUI，TUI 低系统资源消耗和高效的全键盘操作促使我学习如何开发这类应用。最终，我找到了 Bubbletea 这个框架。

一个月前，我写了一篇基于 Bubbletea 开发 TUI 命令的文章。当时的我对这个框架的认识并不深入，仅限于开发一些简单小程序，如果想开发一个如 LazyGit 般复杂的程序时，就没有那么容易了。

今天的文章重点介绍 bubbletea 衍生的一系列扩展库，它们都位于 github.com/charmbracelet。

特别的是，这些扩展库 star 数基本都在千以上。说实在的，很少遇到核心框架之外，扩展库的 star 也基本在千以上的。我觉得主要因为这些库的普适性，可独立于 bubbletea 之外使用。

## Bubble Tea 核心框架

Bubbletea 是这个生态的核心框架，它是我们开发 TUI 应用的起点，基于模型-视图-更新（MVU）架构（The Elm Architecture）。

具体细节可查看前文介绍，其中对 bubbletea 架构的基本架构做了相对详细的说明。

如下是源于官方指南中的 HelloWorld 案例，基于 bubbletea 创建一个简单的计数器。

```go
type model int

// 初始化模型，设置计数器的初始值
func initialModel() model {
	return 0
}

func (m model) Init() tea.Cmd {
	return nil
}

// 更新函数，根据消息更新模型
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyMsg:
		switch msg.String() {
		case "q":
			return m, tea.Quit 
		case "+":
			return m + 1, nil
		case "-":
			return m - 1, nil
		}
	}
	return m, nil
}

func (m model) View() string {
	return fmt.Sprintf("Count: %d\nPress + to increment, - to decrement, q to quit.\n", m)
}

func main() {
	p := tea.NewProgram(initialModel())
	if _, err := p.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "Error running program: %v", err)
		os.Exit(1)
	}
}
```

效果如下：

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-02/2024-02-19-tui-library-bubble-tea-in-golang-03.gif)

## 一些基础扩展

随着我对 Bubble Tea 框架的深入，需求也在变多，我开始了解其它 bubbletea 衍生而来的扩展库。我觉得首先是 3 个极可能受到大家关心的库，分别是布局与样式 - Lipgloss，组件库 - bubbles，交互式 form 表单 - huh

Lipgloss，它提供了定义样式和布局的能力。如通过它调整颜色、边框和布局等，轻松实现美观实用的界面。Lipgloss 能极大地提升 TUI 应用的外观和用户体验。

```go
style := lipgloss.NewStyle().
  BorderStyle(lipgloss.RoundedBorder()). // 圆角边框
  Bold(true).                            // 加粗
  Foreground(lipgloss.Color("#FAFAFA")). // 前景色
  Background(lipgloss.Color("#7D56F4")). // 背景色
  PaddingTop(2).                         // 顶部 padding
  PaddingLeft(4).                        // 左侧 padding
  AlignVertical(lipgloss.Center).        // 垂直居中
  Height(5).                             // 高度
  Width(40)                              // 宽度

// 水平布局
left := style.Render("Hello Left!")
right := style.Render("Hello Right")
fmt.Println(lipgloss.JoinHorizontal(lipgloss.Top, left, right))

// 垂直布局
top := style.Render("Hello Top!")
bottom := style.Render("Hello Bottom!")
fmt.Println(lipgloss.JoinVertical(lipgloss.Top, top, bottom))
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-02-v2.png)

Bubbles，它为 Bubbleta 提供了基础组件。让我能轻松添加各种 UI 组件，如进度条和文本输入框，这些都是创建交互式命令行应用不可或缺的元素。

组件的示例代码要结合 bubbletea 框架，我就不粘贴大段的代码了。

文本输入框：
![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-03.gif)

表格：
![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-04.gif)

进度条：
![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-05.gif)

更多就不一一展示了，查看 bubbles 仓库地址即可。

Huh，为 Bubble Tea 提供了构建交互式表单和提示的能力。相对于 bubbles 中的基础组件，它使我们能轻松实现各种用户输入界面，从选择列表到文本输入框，再到确认对话框。

和 Lipgloss 一样，Huh 也是可脱离 bubbletea 使用的。

一个简单案例：

```go
var name string

_ = huh.NewInput().
  Title("What's your name?").
  Value(&name).
  Run() // this is blocking...

fmt.Printf("Hey, %s!\n", name)
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-06.gif)

Huh 官方 README 文档中提供了一个完整的购买汉堡的订单流程。

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-07.gif)

## 高级工具库

除了上述的基础库，charmbracelet 下还有一些高级的工具库，如 markdown 渲染器 - glamour 和动画制作 - Harmonica。唯一可惜的是，没找到图表库。

Glamour，它为了终端显示 markdown 文档提供了支持，它支持样式自定义，提供了几种预定义主题风格。

```go
in := `# Hello World

This is a simple example of Markdown rendering with Glamour!
Checkout the [other examples](https://github.com/charmbracelet/glamour/tree/master/examples)

Bye!
`
r, _ := glamour.NewTermRenderer(glamour.WithAutoStyle(), glamour.WithWordWrap(40))
out, err := r.Render(in)
if err != nil {
  log.Fatal(err)
}

fmt.Print(out)
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-08.png)

诸如 github、gitlab 和 gitee 的官方 CLI 工具，用的都是这个库实现的 markdown 渲染。

他们还提供了一个基于 Glamour 实现的 glow 命令行工具，支持查看本地或远程的 Markdown 文档。

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-09.gif)

我原本想找一个能和 bubbletea 集成的图表组件库，但只发现了一个名为 Harmonica 的库。

Harmonica 主要是用于模拟弹簧运动，它本身并不提供绘制逻辑，只会生成物理的运动参数（如位置和速度），以实现平滑和自然的动画效果。但它最近已经没更新了。

官方提供的一个案例效果：

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-10.gif)

## SHELL 脚本编写 TUI

我要感慨下，之前看到很多用 rust 写的 CLI 工具，羡慕不已，Go 原来也有这么好用或者说是比之更佳的库。

如果你对 rust 和 Go 都不了解，但想实现一个 TUI 应用，我要推荐这个命令- Gum，它让我们通过 SHELL 脚本即可实现 TUI 应用。

Gum 提供了一系列工具，实现基于 Shell 脚本即可编写一些小巧的 TUI 应用，有助于优化我们的日常工作流。

我们来看看如下是一些简单用法说明：

- **输入（input）**: 提示用户输入文本。
  
```sh
gum input --placeholder "Enter your name"
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-14.gif)

- **选择（choose）**: 创建一个交互式的列表让我们从中选择一个选项。
  
```sh
gum choose "Option 1" "Option 2" "Option 3"
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-11.gif)

**过滤（filter）**: 从输入的列表中过滤出符合条件的项。
  
```sh
echo -e "one\ntwo\nthree" | gum filter
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-13.gif)
**确认（confirm）**: 弹出一个确认对话框让我们选择确认。
  
```sh
gum confirm "Are you sure?" && echo "I'm sure" || echo "No"
```

![](https://cdn.jsdelivr.net/gh/poloxue/images@2024-03/2024-03-15-tui-app-using-bubbletea-12-v1.gif)

**文件选择（file）**: 让用户从文件系统中选择一个文件。
  
```sh
gum file
```

**格式化（format）**: 使用模板格式化字符串。
  
```sh
gum format "Hello, {{.Name}}!" --set Name=World
```


**联接（join）**: 垂直或水平联接文本。
  
```sh
echo -e "Hello\nWorld" | gum join
```

**分页（pager）**: 分页查看长文本文件。
  
```sh
gum pager long-text-file.txt
```

**旋转指示器（spin）**: 在执行长时间命令时显示旋转指示器。
  
```sh
gum spin -- sleep 5
```

**样式（style）**: 应用样式到文本。
  
```sh
echo "Hello, world!" | gum style --foreground 208 --background 235 --bold
```

**表格（table）**: 渲染一个数据表格。
  
```sh
gum table --header "Name,Email" --rows "Alice,alice@example.com\nBob,bob@example.com"
```

**写作（write）**: 提示用户输入长文本。
  
```sh
gum write
```

**日志（log）**: 输出日志信息。
  
```sh
gum log "An error occurred" --level error
```

`gum`是为了让Shell脚本更加"迷人"，通过这些命令，你可以创建更加丰富和交互式的命令行应用。每个命令都支持多种选项，可以通过运行`gum <command> --help`来查看特定命令的帮助信息和可用选项。

## SSH 的远程 TUI 应用

SSH 本质上也是一种传输协议，借助一个名为 wish 的库，我们可轻松实现其他人远程可 SSH 远程访问的 TUI 应用。这与 HTTP 和 Web 服务有这种异曲同工之妙。

charmbracelet 下有一个名为 Soft-Serve 的项目，它其实是 TUI 版本的 gitserver。

## 其他

除了上述介绍的这些，charmbracelet 下还有其他的一些值得推荐的库。如：

vhs，一个终端操作录制命令，提供了一种简单高效的方式捕捉和分享终端操作。

pop，一个支持在终端接发邮件的命令，操作起来高效便捷。

kancli，一个在终端管理看板任务的工具，使得我的项目管理更加高效。

Log，一个支持彩色输出的 Go 日志库，可提高我们日常的调试体验和阅读的舒适度。

## 结语

通过这次深入Bubble Tea及其生态系统的旅程，我不仅学习了如何开发功能丰富的TUI应用，还探索了如何使这些应用更加美观、互动和实用。Bubble Tea生态不仅提供了强大的工具和库，还激发了我对命令行界面新可能性的想象。

正如我在这篇文章的开始提到的，无论是开发简单的行内工具、复杂的全屏应用，还是利用Go编程或Shell脚本，甚至是创建表单、处理Markdown或开发远程TUI应用，Bubble Tea及其扩展库都能提供必要的支持和灵感。

我希望我的经历能够激励更多的开发者探索这个生态系统，并发现它为TUI应用开发带来的乐趣和便利。Bubble Tea不仅是一个框架，它是一个充满可能性的世界，等待着每一个好奇和热情的探索者。

