---
title: "基于 bubbletea 生态开发 TUI 应用"
date: 2024-03-15T08:50:00+08:00
draft: true
comment: true
---

我对 TUI 的兴趣源于一个名为 LazyGit 的终端命令，它让我只需几次按键即可完成一次 Git 提交。

不同于 GUI，TUI 低系统资源消耗和高效的全键盘操作促使我学习如何开发这类应用。最终，我找到了 Bubbletea 这个框架。

一个月前，我写了一篇基于 Bubbletea 开发 TUI 命令的文章。当时的我对这个框架的认识并不深入，仅限于开发一些简单小程序，如果想开发一个如 LazyGit 般复杂的程序时，就没有那么容易了。

今天这篇文章重点介绍 bubbletea 衍生出来的一系列扩展，让我们无论是开发简单的行内工具，还是复杂全屏应用，都更得心应手。

### Bubble Tea 核心框架

Bubbletea 是这个生态的核心，它是我们开发 TUI 应用的起点，基于模型-视图-更新（MVU）架构（The Elm Architecture）。具体细节可查看前文的介绍。

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

### 一些基础扩展

随着我对 Bubble Tea 框架的深入，需求也在变多，我开始了解其它 bubbletea 衍生而来的扩展库。我觉得首先是 4 个极可能受到大家关心的扩展库，分别是布局与样式 - Lipgloss，组件库 - bubbles，交互式 form 表单 - huh，markdown 渲染器 - glamour，还有动画制作 - Harmonica。

Lipgloss，它提供了布局和样式的定制能力。如通过它调整布局、颜色、边框等，轻松实现美观实用的界面。Lipgloss 能极大地提升 TUI 应用的外观和用户体验。

Bubbles，它为 Bubbleta 提供了基础组件。让我能轻松添加各种 UI 组件，如进度条和文本输入框，这些都是创建交互式命令行应用不可或缺的元素。

Huh，为Bubble Tea提供了构建交互式表单和提示的能力。它使我能够轻松地设计和实现各种用户输入界面，从选择列表到文本输入框，再到确认对话框。

Harmonica，支持基于在 bubbletea 引人动画效果。虽然，动画效果不是 TUI 应用的强需求，但也为我们提供了更多可能性。

如果你有在终端显示 markdown 文档的需求，推荐使用 glamour 库，它支持在终端上显示 Markdown 内容，且支持样式自定义。

## SHELL 脚本编写 TUI

如果你想通过 SHELL 脚本编写 TUI 应用，推荐一个基于 Bubbletea 开发的命令行应用 - **gum**。它提供了一系列的工具，通过 Shell 脚本即可编写TUI应用。如果不熟悉 Go 或者不想深入 Bubbletea 框架，但又想做一些小应用，可以考虑下它。

## SSH 的远程 TUI 应用

随着我对 Bubble Tea 生态系统的进一步探索，我发现了一些特定用途的工具和库，它们为我的TUI应用开发增加了额外的可能性。

**Wish**是一个我特别感兴趣的库。它利用SSH作为传输协议，允许我开发远程TUI应用。这打开了一个全新的领域，让我能够为用户提供安全、无缝的远程访问体验。通过Wish，我能够设计一个可以从任何地方通过SSH访问的TUI应用，这在管理服务器或提供远程服务时尤其有用。

接着，我遇到了**Soft-Serve**，这是一个TUI版本的Git服务器。它让我惊喜地发现，不仅可以用命令行管理Git仓库，还能拥有一个友好的TUI界面来进行操作。这对于提高我的开发效率有着不小的帮助，也为团队协作提供了一个更直观的方式。

录制终端会话时，**VHS**证明是一个宝贵的资源。无论是为了创建教程还是记录会话，VHS都提供了一种简单而高效的方式来捕捉和分享终端操作。

### 应用案例和工具

在处理文档和笔记时，**Glow**成为了我的好帮手。它们能够在终端中渲染Markdown，使得查看和编辑文档变得既方便又舒适。我尤其喜欢Glow的阅读体验，它让命令行中的Markdown文件就像是精美的电子书一样。

探索了这些库和工具之后，我也开始关注一些实际的应用案例，如**Pop**和**Kancli**。Pop让我能够在终端中发送邮件，这对于快速回复或管理我的电子邮件非常方便。而Kancli为我提供了一个在终端管理看板任务的工具，使得我的项目管理更加高效。

在所有这些探索中，**Log**是我经常回归的工具。它支持彩色日志输出，极大地提高了我的调试效率和日志阅读的舒适度。Log的简单性和实用性让它成为了我开发工具箱中不可或缺的一部分。

### 结语

通过这次深入Bubble Tea及其生态系统的旅程，我不仅学习了如何开发功能丰富的TUI应用，还探索了如何使这些应用更加美观、互动和实用。Bubble Tea生态不仅提供了强大的工具和库，还激发了我对命令行界面新可能性的想象。

正如我在这篇文章的开始提到的，无论是开发简单的行内工具、复杂的全屏应用，还是利用Go编程或Shell脚本，甚至是创建表单、处理Markdown或开发远程TUI应用，Bubble Tea及其扩展库都能提供必要的支持和灵感。

我希望我的经历能够激励更多的开发者探索这个生态系统，并发现它为TUI应用开发带来的乐趣和便利。Bubble Tea不仅是一个框架，它是一个充满可能性的世界，等待着每一个好奇和热情的探索者。

