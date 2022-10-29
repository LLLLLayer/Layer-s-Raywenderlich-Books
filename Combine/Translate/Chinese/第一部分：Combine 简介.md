# 第一部分：Combine 简介

在本书的这一部分中，你将了解 Combine 的基础知识并了解它所包含的一些构建块。 你将了解 Combine 旨在解决什么问题，以及它提供了哪些抽象来帮助你解决这些问题：Publisher、Subscriber、Subscription、Subject 等等。



## 第一节：你好，Combine！

本书旨在向你介绍 Combine 框架，以及使用 Swift 为 Apple 平台编写声明式、反应式应用程序。

用 Apple 自己的话来说：“Combine 框架为你的应用程序如何处理事件提供了一种声明性方法。你可以为给定的事件源创建单个处理链，而不是潜在地实现多个委托回调或完成处理程序闭包。链的每个部分是一个组合运算符，它对从上一步接收到的元素执行不同的操作。”

虽然这个描述非常准确且切中要害，但这个令人愉快的定义可能让人第一次听起来有点过于抽象。这就是为什么在深入研究编码练习和处理后续章节中的项目之前，你需要花一点时间来了解一下 Combine 解决的问题以及它用来解决这些问题的工具。

一旦你建立了相关的词汇表并总体上对框架有所了解，就将在编码时理解这些基础知识。逐渐地，随着你在本书中的学习，你将了解更高级的主题并最终完成多个项目。当你学习了所有内容后，你将在最后一章中使用 Combine 构建一个完整的应用程序。



### 异步编程

在简单的单线程语言中，程序按顺序逐行执行。例如，在伪代码中：

```swift w
begin
  var name = "Tom"
  print(name)
  name += " Harding"
  print(name)
end
```



同步代码很容易理解，并且特别容易确定数据的状态。 通过单线程执行，你始终可以确定数据的当前状态。 在上面的示例中，你知道第一次打印将始终打印”Tom”，第二次将始终打印“Tom Harding”。

现在，假设你使用运行异步事件驱动 UI 框架的多线程语言编写程序，例如在 Swift 和 UIKit 上运行的 iOS 应用程序。

考虑可能发生的情况：

``` Swift
--- Thread 1 ---
begin
  var name = "Tom"
  print(name)

--- Thread 2 ---
name = "Billy Bob"

--- Thread 1 ---
  name += " Harding"
  print(name)
end
```



在这里，代码将 `name` 的值设置为“Tom”，然后将“Harding”添加到它，就像以前一样。 但是因为另一个线程可以同时执行，所以程序的其他部分可能会在 `name` 的两个突变之间运行并将其设置为另一个值，例如“Billy Bob”。

当代码在不同的核上并发运行时，很难说代码的哪一部分会先修改共享状态。

上面示例中在“Thread 2”上运行的代码可能是：

- 在与原始代码不同的 CPU 内核上同时执行。

- 在 `name += " Harding"` 之前执行，因此它不是原始值 "Tom"，而是得到 "Billy Bob"。



运行此代码时究竟会发生什么取决于系统负载，并且每次运行程序时你可能会看到不同的结果。运行异步并发代码后，管理应用程序中的可变状态将成为一项必要的任务。



### Foundation 和 UIKit/AppKit

多年来，Apple 一直在不断改进其平台的异步编程。他们创建了几种机制，你可以在不同的系统级别上使用这些机制来创建和执行异步代码。

你可以使用低级别的 API，就像使用 NSThread 管理自己的线程一样，一直到使用 Swift 的现代并发 `async/await` 等。

你可能已经在你的应用程序中使用了以下一些内容：

- **NotificationCenter**：在感兴趣的事件发生时执行一段代码，例如用户更改设备的方向或键盘在屏幕上显示或隐藏。

- **委托模式**：允许你定义一个代表另一个对象或与之协调的对象。 例如，在你的应用程序委托中，你定义了当新的远程通知到达时应该发生什么，但不知道这段代码何时运行或执行多少次。

- **GCD 和 Operation**：帮助你抽象工作的执行。你可以使用它们来安排代码在串行队列中按顺序运行，或者在具有不同优先级的不同队列中同时运行多个任务。

- **闭包**：创建可以在代码中传递的分离代码片段，以便其他对象可以决定是否执行它、执行多少次以及在什么上下文中执行。



由于大多数典型代码异步执行一些工作，并且所有 UI 事件本质上都是异步的，因此无法假设整个应用程序代码将执行的顺序。

然而，编写好的异步程序是可能的。它只是更复杂一些。

当然，造成这些问题的原因之一是一个可靠的、真实的应用程序很可能使用所有不同类型的异步 API，每个都有自己的接口，如下所示：

![image-20220913022914380](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913022914380.png)

Combine 为 Swift 生态系统引入了一种通用的高级语言来设计和编写异步代码。

Apple 也将 Combine 集成到其其他框架中，如 Timer、NotificationCenter 和 Core Data 等核心框架。 同时，Combine 也很容易集成到你自己的代码中。

最后但同样重要的是，Apple 设计了他们令人惊叹的 UI 框架 SwiftUI，以便与 Combine 轻松集成。

为了让你了解 Apple 对使用 Combine 进行响应式编程的重视程度，这里有一个简单的图表，显示了 Combine 在系统层次结构中的位置：

![image-20220912215452368](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220912215452368.png)

从 Foundation 一直到 SwiftUI，各种系统框架都依赖于 Combine，并提供 Combine 集成作为其更“传统”API 的替代方案。



### Swift 的现代并发

Swift 5.5 引入了一系列用于开发异步和并发代码的 API，得益于新的线程池模型，这些 API 允许你的代码安全快速地随意挂起和恢复异步工作。

现代并发 API 使许多经典的异步问题相当容易解决——例如等待网络响应、并行运行多个任务等等。

这些API 解决了与 Combine 相同的一些问题，但 Combine 的优势在于其丰富的操作。随着时间的推移，Combine 提供的用于处理事件的运算符使许多复杂的常见场景更容易解决。

反应式操作符直接解决网络、数据处理和处理 UI 事件中的各种常见问题，因此对于更复杂的应用程序，使用 Combine 开发有很多好处。

而且，谈到 Combine 的优势，让我们快速了解一下反应式编程迄今为止的出色表现。



### Combine 的背景

声明式、反应式编程并不是一个新概念。 它已经存在了很长一段时间，在过去十年中它已经相当引人注目。

第一个“现代”反应式解决方案在 2009 年大行其道，当时微软的一个团队推出了一个名为 Reactive Extensions for .NET (Rx.NET) 的库。微软在 2012 年将 Rx.NET 实现开源，从那时起，许多不同的语言开始使用它的概念。 目前，有许多 Rx 标准的端口，如 RxJS、RxKotlin、RxScala、RxPHP 等等。

对于 Apple 的平台，已经有几个第三方响应式框架，如 RxSwift，它实现了 Rx 标准； ReactiveSwift，直接受 Rx 启发； Interstellar 是一个自定义实现。

Combine 实现了一个与 Rx 不同但相似的标准，称为 Reactive Streams。 Reactive Streams 与 Rx 有一些关键区别，但它们都同意大多数核心概念。在 iOS 13/macOS Catalina 中，Apple 通过内置的系统框架 Combine 为其生态系统带来了响应式编程支持。

如果你以前没有使用过上面提到的一个或另一个框架，请不要担心。 到目前为止，响应式编程对于 Apple 的平台来说是一个相当小众的概念，尤其是对于 Swift。

你将首先学习一些 Combine 的基础知识，看看它如何使开发者编写安全可靠的异步代码。



### Combine 的基础

概括地说，Combine 中的三个关键部分是**发布者(Publisher)、操作符(Operator)和订阅者(Subscriber)**。 

你将在第 2 节“发布者和订阅者”中详细了解发布者和订阅者，本书的完整第二部分将引导你尽可能多地了解操作符。在这个介绍性章节中，你将获得一个简单的速成课程，让你大致了解这些类型在代码中的用途以及它们的职责。



#### 发布者(Publisher)

发布者是可以随着时间的推移向一个或多个相关方（例如订阅者）发出值的类型。不管发布者的内部逻辑如何，无论是数学计算、网络或处理用户事件等，每个发布者都可以发出这三种类型的多个事件：

1. 发布者的通用输出类型的输出值。

2. 成功的 completion。

3. 带有发布者失败类型错误的 completion。



发布者可以发出零个或多个输出值，如果它完成，无论是成功还是由于失败，它都不会发出任何其他事件。

以下是发布 Int 值的发布者在时间轴上的可视化效果：

![image-20220913011932213](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913011932213.png)

蓝色框表示在时间线上给定时间发出的值，数字表示发出的值。一条垂直线，就像你在图表右侧看到的那条一样，表示成功的流完成。

三个可能事件的非常简单和普遍，它可以代表你程序中的任何类型的动态数据。 这就是为什么你可以使用 Combine 的发布者在你的应用程序中处理任何任务——无论是处理数字、拨打网络电话、对用户手势做出反应还是在屏幕上显示数据。

不要总是在你的旧工具箱中寻找合适的工具来解决任务，无论是添加委托还是注入回调——你可以尝试去使用发布者。

发布者的最佳功能之一是它们内置了错误处理； 错误处理是你在最后添加的可选内容。

Publisher 协议有两种类型是通用的，正如你在前面的图中可能已经注意到的那样：

- Publisher.Output 是发布者输出值的类型。 如果它是 Int 发布者，它永远不能发出 String 或 Date 值。

- Publisher.Failure 是发布者在失败时可以抛出的错误类型。 如果发布者永远不会失败，你可以使用 Never failure 类型来指定它。

当你订阅给定的发布者时，会知道期望从中获得什么值以及可能会因哪些错误而失败。



#### 操作符(Operator)

操作符是在 Publisher 协议上声明的方法，它们返回相同的或新的发布者。这非常有用，因为你可以一个接一个地调用一堆运算符，从而有效地将它们链接在一起。

因为这些称为操作符的方法是高度解耦和可组合的，所以它们可以组合起来，在单个订阅的执行中实现非常复杂的逻辑。

操作符像拼图一样紧密地组合在一起的设计令人着迷的。如果一个人的输出与下一个人的输入类型不匹配，则它们不能错误地按错误顺序排列或组合在一起：

![image-20220913013326763](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913013326763.png)



你可以以明确的方式定义每个异步抽象工作的顺序以及正确的输入/输出类型和内置错误处理。这非常好。

此外，操作符总是有输入和输出，通常称为上游(Upstream)和下游(Downstream)——这使他们能够避免共享状态。

操作符专注于处理从前一个操作员那里收到的数据，并将其输出提供给链中的下一个操作员。这意味着没有其他异步运行的代码可以“跳入”并更改你正在处理的数据。



#### 订阅者(Subscriber)

最后，你到达订阅链的末端：每个订阅都以一个订阅者结束。订阅者通常对发出的输出或完成事件做“某事”。

![image-20220913013857679](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913013857679.png)



目前，Combine 提供了两个内置订阅者，这使得处理数据流变得简单：

- 接收器订阅者允许你使用闭包接收输出值和 completion。从在那里你可以对收到的事件做任何想做的事情。

- 分配订阅者允许你在不需要自定义代码的情况下将结果输出绑定到数据模型或 UI 控件上的某个属性，通过 key path 直接在屏幕上显示数据。

如果你对数据有其他需求，创建自定义订阅者甚至比创建发布者更容易。 Combine 使用一组非常简单的协议，使你能够合适的构建自己的自定义工具。



#### 订阅(Subscription)

> 注意：本书使用术语订阅(Subscription)来描述 Combine 的订阅协议及其符合对象，以及发布者、操作者和订阅者的完整链。



当你在订阅的末尾添加订阅者时，它会在链的开头“激活”发布者。 这是一个奇怪但重要的细节——**如果没有订阅者可能接收输出，则发布者不会发出任何值**。

订阅是一个很棒的概念，因为它们允许你使用自己的自定义代码和错误处理声明一连串异步事件，并且只需要一次，然后你就不必再考虑它了。

如果你完全使用 Combine，你可以通过订阅来描述你的整个应用程序的逻辑，一旦完成，只需让系统运行所有东西，而不需要推送或拉取数据或回调其他对象：

![image-20220913014812811](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913014812811.png)



一旦订阅代码成功编译并且你的自定义代码中没有逻辑问题，订阅将在每次某个事件时异步“触发”，如用户手势、计时器关闭或其他唤醒发布者的事件。

更好的是，你不需要专门管理订阅，这要归功于 Combine 提供的一个名为 Cancellable 的协议。

系统提供的两个订阅者都符合 Cancellable，这意味着你的订阅代码（例如整个发布者、操作符和订阅者调用链）返回一个 Cancellable 对象。每当你从内存中释放该对象时，它都会取消整个订阅并从内存中释放其资源。

这意味着你可以通过 strong “绑定”订阅的生命周期到视图控制器的属性中。这样，每当用户从视图堆栈中关闭视图控制器时，都会取消初始化其属性并取消你的订阅。

或者为了自动化这个过程，你可以在你的类型上有一个 `[AnyCancellable]` 集合属性，并在其中抛出你想要的任意数量的订阅。当属性从内存中释放时，它们都会被自动取消和释放。



### Combine 有什么好处？

无论如何，你永远不使用 Combine 仍可以创建好的应用程序。同样你也可以在没有 Core Data、URLSession 甚至 UIKit 的情况下创建好的应用程序。但是使用这些框架比自己构建这些抽象更方便、安全和高效。

Combine 和其他系统框架旨在为你的异步代码添加另一个抽象。系统级别的另一个抽象级别意味着经过充分测试、更紧密集成和更安全的技术。

由你决定 Combine 是否适合你的项目，但这里只是一些你可能尚未考虑的原因：

- Combine 在系统级别上集成。这意味着 Combine 本身使用不公开的语言功能，提供了你无法自己构建的 API。

- Combine 将许多常见操作抽象为 Publisher 协议上的方法，并且它们已经过很好的测试。

- 当你所有的异步工作都使用同一个接口——发布者——组合和可重用性变得非常强大。

- Combine 的操作符是高度可组合的。如果你需要创建一个新操作符，该新操作符将立即与其余的 Combine 即插即用。

- Combine'a 异步运算符已经过测试。剩下要做的就是测试自己的业务逻辑。

如你所见，大多数好处都围绕着安全性和便利性。



### App 架构

这个问题也很重要，接着来了解一些使用 Combine 将如何改变你的旧代码和应用程序架构。

Combine 不是一个影响你如何构建应用程序的框架。Combine 处理异步数据事件和统一通信协议——它不会改变别的内容，例如你将如何分离项目中的职责。

你可以在你的 MVC（模型-视图-控制器）应用程序中使用组合，你可以在你的 MVVM（模型-视图-视图模型）代码、VIPER 等中使用它。

这是采用 Combine 的关键方面之一，你可以迭代地和有选择地添加 Combine 代码，仅在你希望在代码库中改进的部分中使用它。这不是你需要做出的“全有或全无”的选择。

你可以首先转换你的数据模型，或调整你的网络层，或者仅在你添加到应用程序的新代码中使用 Combine，同时保持现有功能不变。

如果你同时采用 Combine 和 SwiftUI，情况会略有不同。在这种情况下，从 MVC 架构中删除 C 确实有意义，这要归功于将 Combine 和 SwiftUI 结合使用。

视图控制器根本没有机会对抗 Combine/SwiftUI 组合。当你从数据模型到视图一直使用响应式编程时，你不需要一个特殊的控制器来控制你的视图：

![image-20220913021246378](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913021246378.png)



这听起来很有趣，在第 15 节“实践：Combine & SwiftUI”中包含了如何结合使用这两个框架的介绍。



### 书籍项目

在本书中，你将首先从概念开始，然后继续学习和尝试多种操作符。

与其他系统框架不同，你可以在 Playground 的隔离上下文中非常方便地使用 Combine。

在 Xcode Playground 学习可以让你在阅读给定章节时轻松前进并快速进行实验，并在 Xcode 的控制台中立即查看结果：

![image-20220913021621283](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913021621283.png)



Combine 不需要任何第三方依赖，通常每节的 Playground 代码中包含的一些简单的帮助文件就足以让你开始运行。 如果在 Playground 中进行实验时 Xcode 卡住了，快速重启可能会解决问题。

一旦你转向比使用单个操作符更复杂的概念，你将在 Playground 和真正的 Xcode 项目（如 Hacker News 应用程序）之间交替学习，这是一个实时显示新闻的新闻阅读器：

![image-20220913021908958](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220913021908958.png)



重要的是，对于每一章，你都从提供的 Playground 或 Project 开始，因为它们可能包含一些与学习 Combine 无关的自定义帮助代码。这些是预先编写的，因此不会分散你对该章节重点的学习。

在最后一章中，你将利用你在本书中学到的所有技能，完成开发一个依赖于 Combine 和 Core Data 的完整 iOS 应用程序。这将为你在使用 Combine 构建真实应用程序的道路上提供最后的推动力！



### 关键点

- Combine 是一个声明式的响应式框架，用于随着时间的推移处理异步事件。

- 它旨在解决现有问题，例如统一用于异步编程的工具、处理可变状态以及错误处理。

- Combine 围绕三种主要类型：发布者随着时间的推移发出事件，操作符异步处理和操作上游事件，订阅者使用结果并使用它们做一些有用的事情。



### 然后去哪儿？

希望这个介绍性章节对你有所帮助，并让你初步了解结合解决的问题，并了解它提供的一些工具，以使你的异步代码更安全、更可靠。

本章的另一个重要内容是对 Combine 的期望以及超出其范围的内容。 现在，当我们谈论随着时间的推移响应式代码或异步事件时，你知道自己在做什么。 当然你也不希望使用 Combine 神奇地解决应用程序在导航或在屏幕上绘图方面的问题。

最后，在接下来的章节中为你准备的内容有望让你对使用 Swift 进行组合和反应式编程感到兴奋。 



## 第二节：发布者和订阅者

现在你已经了解了 Combine 的一些基本概念，是时候开始使用 Combine 的两个核心组件——发布者和订阅者。在本节中，你将尝试各种方法来创建发布者并订阅它们。

> 注意：在本书的每一章中，你都会用到 Playground 和项目的初始版本和最终版本。starter 已准备好让你为每个示例和挑战输入代码。 如果遇到困难，你可以在 final 或查看最终版本或进行比较。



### 入门

在本章中，你将使用导入了 Combine 的 Xcode Playground。 打开项目文件夹中的 starter/playground，你将看到以下内容：

![image-20220914015722757](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220914015722757.png)

在项目导航器中打开源代码（View ▸ Navigators ▸ Show Project Navigator ▸ Combine playground页面），然后选择 `SupportCode.swift`。 它包含以下辅助函数 `example(of:)`：

```swift
public func example(of description: String,
                    action: () -> Void) {
  print("\n——— Example of:", description, "———")
  action()
}
```

你将使用这个函数来封装你将在本书中使用的每个示例。

但是，在开始使用这些示例之前，你首先需要更详细地了解发布者、订阅者和订阅。 它们构成了 Combine 的基础，使你能够发送和接收数据，通常是异步的。



### 你好，Publisher

Combine 的核心是 Publisher 协议。 该协议定义了一种类型的要求，以便能够随时间将一系列值传输给一个或多个订阅者。 换句话说，发布者发布或发出可能包含感兴趣值的事件。

订阅发布者的想法类似于订阅来自 NotificationCenter 的特定通知。使用 NotificationCenter，你可以表达对某些事件的兴趣，然后在有新事件发生时异步通知你。

事实上，它们非常相似，以至于 NotificationCenter 有一个名为 `publisher(for:object:)` 的方法，它提供了一个可以发布通知的 Publisher 类型。

要在实践中检查这一点，请返回到 starter playground 并将 `Add your code here` 占位符替换为以下代码：

```swift
example(of: "Publisher") {
    // 1
    let myNotification = Notification.Name("MyNotification")
    
    // 2
    let publisher = NotificationCenter.default
        .publisher(for: myNotification, object: nil)
}

```



在此代码中：

1. 创建通知名称。

2. 访问 `NotificationCenter` 的默认实例，调用它的 `publisher(for:object:)` 方法，并将它的返回值赋给一个局部常量。

按住 Option 点击 `publisher(for:object:)`，你会看到它返回一个 Publisher，当默认通知中心广播一个通知时，它会发出一个事件。

那么当通知中心已经能够在没有发布者的情况下广播其通知时，发布通知有什么意义呢？ 

你可以将这些类型的方法视为从较旧的异步 API 到较新的替代方案的桥梁——如果你愿意，可以将它们 Combine 起来。

发布者发出两种事件：

1. Value，也称为 Element。

2. Completion 事件。

发布者可以发出零个或多个 Value，但只能发出一个 Completion 事件，它可以是正常Completion 事件或 Error。 一旦发布者发出 Completion 事件，它就完成了，不能再发出任何事件。

> 注意：从这个意义上说，发布者有点类似于 Swift iterator。 一个非常有价值的区别是 Publisher 的 Completion 可能是成功的也可能是失败的，你需要主动从迭代器中获取 Value，而 Publisher 则将 Value 推送给它的消费者。


接下来，你将通过使用 NotificationCenter 来观察并发布通知。 当你不再有兴趣接收该通知时，你还将取消注册该观察者。

将以下代码附加到示例闭包中已有的代码中：

```swift
// 3
let center = NotificationCenter.default

// 4
let observer = center.addObserver(
    forName: myNotification,
    object: nil,
    queue: nil) { notification in
        print("Notification received!")
    }

// 5
center.post(name: myNotification, object: nil)

// 6
center.removeObserver(observer)
```

使用此代码，你可以：

3. 获取 `NotificationCenter.default` 的句柄。

4. 创建一个观察者来监听你之前创建的通知。

5. 使用该名称发布通知。

6. 从通知中心移除观察者。

运行 playground，你将看到此输出打印到控制台：

```
——— Example of: Publisher ———
Notification received!
```

该示例的标题有点误导，因为输出实际上并非来自发布者。 为此，你需要订阅者。



### 你好，Subscriber

Subscriber 是一种协议，它定义了能够从发布者接收输入的类型的要求。你将很快深入了解如何遵守发布者和订阅者协议； 现在，你将专注于基本流程。

向 Playground 添加一个新示例，该示例与上一个类似：

```swift
example(of: "Subscriber") {
    let myNotification = Notification.Name("MyNotification")
    let center = NotificationCenter.default
    
    let publisher = center.publisher(for: myNotification, object: nil)
    
}
```

如果你现在要发布通知，发布者不会发出它，因为还没有订阅来使用通知。

#### 使用 `sink(_:_:)` 订阅

继续上一个示例并添加以下代码：

```swift
// 1
let subscription = publisher
    .sink { _ in
        print("Notification received from a publisher!")
    }

// 2
center.post(name: myNotification, object: nil)
// 3
subscription.cancel()
```

使用此代码：

1. 通过在发布者上调用 `sink` 创建订阅。

2. 发布通知。

3. 取消订阅。

不要让 `sink` 方法名的晦涩难懂迷惑。 按住 Option 键单击 `sink`，你会看到它提供了一种简单的方法来附加带有闭包的订阅者以处理来自发布者的输出。在此示例中，米只需打印一条消息以指示已收到通知。 你将很快了解有关取消订阅的更多信息。

运行 playground，你将看到以下内容：

```
——— Example of: Publisher ———
Notification received from a publisher!
```



`skin` 将持续接收与发布者发出的一样多的值——这被称为无限需求(Unlimited demand)。尽管你在前面的示例中忽略了它们，但 `sink` 操作符实际上提供了两个闭包：一个用于处理接收 completion 事件（成功或失败），另一个用于处理接收的 Value。

要尝试这些，请将这个新示例添加到你的 Playground：

```swift
example(of: "Just") {
    // 1
    let just = Just("Hello world!")
    
    // 2
    _ = just
        .sink(
            receiveCompletion: {
                print("Received completion", $0)
            },
            receiveValue: {
                print("Received value", $0)
            })
}
```

在这里：

1. 使用 Just 创建发布者，它允许你从单个值创建发布者。

2. 创建对发布者的订阅并为每个接收到的事件打印一条消息。

运行 playground，你将看到以下内容：

```
——— Example of: Just ———
Received value Hello world!
Received completion finished
```

按住 Option 键单击 `Just`，它是一个发布者，它向每个订阅者发出一次输出，然后完成。

尝试通过将以下代码添加到示例末尾来添加另一个订阅者：

```swift
_ = just
    .sink(
        receiveCompletion: {
            print("Received completion (another)", $0)
        },
        receiveValue: {
            print("Received value (another)", $0)
        })
```

运行 playground，正如它所说的，Just 很高兴地向每个新订阅者发送它的输出一次，然后完成。

```
Received value (another) Hello world!
Received completion (another) finished
```



#### 使用 `assign(to:on:)` 订阅

除了` sink` 之外，内置的 `assign(to:on:)` 运算符能够将接收到的值分配给对象的 KVO 兼容属性。添加此示例以查看其工作原理：

```swift
example(of: "assign(to:on:)") {
    // 1
    class SomeObject {
        var value: String = "" {
            didSet {
                print(value)
            }
        }
    }
    
    // 2
    let object = SomeObject()
    
    // 3
    let publisher = ["Hello", "world!"].publisher
    
    // 4
    _ = publisher
        .assign(to: \.value, on: object)
}
```

从一开始：

1. 定义一个具有属性的类，该属性具有打印新值的` didSet` 属性观察器。

2. 创建该类的实例。

3. 从字符串数组创建发布者。

4. 订阅发布者，将收到的每个值分配给对象的 `value` 属性。

运行 playground，你会看到打印出来的：

```
——— Example of: assign(to:on:) ———
Hello
world!
```

> 注意：在后面的章节中，你将看到 `assign(to:on:)` 在处理 UIKit 或 AppKit 应用程序时特别有用，因为你可以将值直接分配给 label、 textView、checkbox 和其他 UI 组件。



#### 使用 `assign(to:)` 重新发布

`assign` 操作符的一个变体，重新发布发布者可用于通过另一个用 `@Published` 属性包装器标记的属性发出的值。 请将这个新示例添加到 playground：

```swift
example(of: "assign(to:)") {
    // 1
    class SomeObject {
        @Published var value = 0
    }
    
    let object = SomeObject()
    
    // 2
    object.$value
        .sink {
            print($0)
        }
    
    // 3
    (0..<10).publisher
        .assign(to: &object.$value)
}
```

使用此代码：

1. 定义并创建一个类的实例，该实例的属性使用`@Published` 属性包装器注解，除了可作为常规属性访问之外，它还为值创建了一个发布者。

2. 使用 `@Published` 属性上的 `$` 前缀来访问其底层发布者，订阅它，并打印出收到的每个值。

3. 创建一个数字发布者并将它发出的每个值分配给 `object` 的值发布者。 请注意使用 `&` 来表示对属性的 inout 引用。

`assign(to:)` 运算符不返回 `AnyCancellable` token，因为它在内部管理生命周期并在 `@Published` 属性取消初始化时取消订阅。

你可能想知道这与简单地使用 `assign(to:on:)` 相比有何用处？ 考虑以下示例（你无需将其添加到 playground 上）：

```swift
class MyObject {
    @Published var word: String = ""
    var subscriptions = Set<AnyCancellable>()
    
    init() {
        ["A", "B", "C"].publisher
            .assign(to: \.word, on: self)
            .store(in: &subscriptions)
    }
}
```

在此示例中，使用 `assign(to: \.word, on: self)` 并存储生成的 `AnyCancellable` 会导致引用循环。 用 `assign(to: &$word)` 替换 `assign(to:on:)` 可以防止这个问题。

你现在将专注于使用 `sink` 残章符，但你将在以及后续章节中了解更多关于使用 `@Published` 属性的内容。



### 你好，Cancellable

当订阅者完成其工作并且不再希望从发布者接收值时，最好取消订阅以释放资源并停止发生任何相应的活动，例如网络调用。

订阅将 `AnyCancellable` 的实例作为“cancellation token”返回，这使得你可以在完成订阅后取消订阅。 `AnyCancellable` 符合 `Cancellable` 协议，该协议正是为此目的需要 `cancel()` 方法。

在前面的订阅者示例的底部，你添加了代码 `subscription.cancel()`。你可以在订阅上调用 `cancel()`，因为 `Subscription` 协议继承自 `Cancellable`。

如果你没有在订阅上显式调用 `cancel()`，它将一直持续到发布者完成，或者直到正常的内存管理导致存储的订阅取消初始化。届时，它会取消订阅。

> 注意：忽略 Playground 中订阅的返回值也可以（例如，`_ = just.sink...`）。但是，需要注意的是：如果你没有在完整项目中存储订阅，则该订阅将在程序流退出创建它的范围后立即取消！

这些都是很好的例子，但幕后还有很多事情要做。是时候了解更多关于发布者、订阅者和订阅者在 Combine 中的角色了。



### 了解正在发生的事情

我们先来解释一下发布者和订阅者之间的相互作用：

![image-20220921013402724](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20220921013402724.png)

查看这个 UML 图：

1. 订阅者订阅发布者。

2. 发布者创建订阅并将其提供给订阅者。

3. 订阅者请求值。

4. 发布者发送值。

5. 发布者发送完成。

> 注意：上图提供了正在发生的事情的简化概述。 稍后，你将在后文更深入地了解此过程。


看看 `Publisher` 协议及其最重要的扩展之一：

```swift
public protocol Publisher {
    // 1
    associatedtype Output
    
    // 2
    associatedtype Failure : Error
    
    // 4
    func receive<S>(subscriber: S)
    where S: Subscriber,
          Self.Failure == S.Failure,
          Self.Output == S.Input
}

extension Publisher {
    // 3
    public func subscribe<S>(_ subscriber: S)
    where S : Subscriber,
          Self.Failure == S.Failure,
          Self.Output == S.Input
}
```

仔细看看：

1. 发布者可以生成的值的类型。

2. 发布者可能产生的错误类型，如果保证发布者不会产生错误，则为 `Never`。

3. 订阅者在发布者上调用 subscribe(_:) 以附加到它。

4. subscribe(_:) 的实现将调用 receive(subscriber:) 将订阅者附加到发布者，即创建订阅。

`associatedtype ` 是发布者接口，阅者必须匹配才能创建订阅的。现在，看看订阅者协议：

```swift
public protocol Subscriber: CustomCombineIdentifierConvertible {
    // 1
    associatedtype Input
    
    // 2
    associatedtype Failure: Error
    
    // 3
    func receive(subscription: Subscription)
    
    // 4
    func receive(_ input: Self.Input) -> Subscribers.Demand
    
    // 5
    func receive(completion: Subscribers.Completion<Self.Failure>)
}
```

仔细看看：

1. 订阅者可以接收的值的类型。

2. 订阅者可以收到的错误类型； 如果订阅者不会收到错误则为  `Never`。

3. 发布者在订阅者上调用 `receive(subscription:)` 来给它订阅。

4. 发布者在订阅者上调用 `receive(_:)` 以向它发送它刚刚发布的新值。

5. 发布者在订阅者上调用 `receive(completion:)` 来告诉它它已经完成了生成值，无论是正常还是由于错误。

发布者和订阅者之间的连接是 `subscription`。 这是订阅协议：

```swift
public protocol Subscription: Cancellable, CustomCombineIdentifierConvertible {
    func request(_ demand: Subscribers.Demand)
}
```



订阅者调用 `request(_:)` 表示它愿意接收更多的值，最多可达最大数量或无限制。

> 注意：我们将订阅者的概念称为它愿意接收背压管理的值。如果没有它，或者其他一些策略，订阅者可能会被来自发布者的更多值淹没，而不是它可以处理的。


在 Subscriber 中，请注意 `receive(_:)` 返回一个 `Demand`。 即使 `subscription.request(_:)` 设置了订阅者愿意接收的初始最大值数，你也可以在每次收到新值时调整该最大值。

> 注意：在 `Subscriber.receive(_:)` 中调整最大值是累加的，即新的最大值会添加到当前最大值。 最大值必须为正，传递负值将导致致命错误。 这意味着你可以在每次收到新值时增加原始最大值，但不能减少它。



### 创建自定义订阅者

是时候把你刚刚学到的东西付诸实践了。 将这个新示例添加到你的 playground：

```swift
example(of: "Custom Subscriber") {
    // 1
    let publisher = (1...6).publisher
    
    // 2
    final class IntSubscriber: Subscriber {
        // 3
        typealias Input = Int
        typealias Failure = Never
        
        // 4
        func receive(subscription: Subscription) {
            subscription.request(.max(3))
        }
        
        // 5
        func receive(_ input: Int) -> Subscribers.Demand {
            print("Received value", input)
            return .none
        }
        
        // 6
        func receive(completion: Subscribers.Completion<Never>) {
            print("Received completion", completion)
        }
    }
}
```

你在这里做的是：

1. 通过 range 的发布者属性创建整数发布者。

2. 定义一个自定义订阅者，`IntSubscriber`。

3. 实现类型别名以指定此订阅者可以接收整数输入并且永远不会收到错误。

4. 实现所需的方法，以 `receive(subscription:)` 开头，由发布者调用；并在该方法中，在 subscription 上调用 `.request(_:)`，指定订阅者愿意在订阅时接收最多三个值。
5. 收到后打印每个值，返回 `.none`，表示订阅者不会调整自己的需求； `.none` 等价于 .max(0)。

6. 打印完成事件。

发布者要发布任何内容，就需要订阅者。 在示例末尾添加以下内容：

```swift
let subscriber = IntSubscriber()
publisher.subscribe(subscriber)
```

在此代码中，你将创建一个与发布者的输出和失败类型相匹配的订阅者。 然后你告诉发布者订阅或附加订阅者。

运行 playground，你将看到以下打印到控制台：

```
——— Example of: Custom Subscriber ———
Received value 1
Received value 2
Received value 3
```

你没有收到完成事件。 这是因为发布者具有有限数量的值，并且你指定了 `.max(3)` 的需求。

在你的自定义订阅者的 `receive(_:)` 中，尝试将 `.none` 更改为 `.unlimited`，因此你的 `receive(_:)` 方法如下所示：

```swift
func receive(_ input: Int) -> Subscribers.Demand {
		print("Received value", input)
		return .unlimited
}
```

再次运行 playground，这次你将看到输出包含所有值以及完成事件：

```
——— Example of: Custom Subscriber ———
Received value 1
Received value 2
Received value 3
Received value 4
Received value 5
Received value 6
Received completion finished
```

尝试将 `.unlimited` 更改为 `.max(1)` 并再次运行 Playground。

你将看到与返回 `.unlimited` 时相同的输出，因为每次收到事件时，你都指定要将最大值增加 1。

将 `.max(1)` 改回 `.none`，并将发布者的定义改为字符串数组。

用：

```swift
let publisher = ["A", "B", "C", "D", "E", "F"].publisher
```

 代替：

```swift
let publisher = (1...6).publisher
```

运行 playground， 你会收到一个错误，即 `subscribe` 方法要求 `String` 和 `IntSubscriber.Input` 类型（即 `Int`）等价。 你收到此错误是因为发布者的`Input` 和 `Failure` 关联类型必须匹配订阅者的输入和失败类型才能在两者之间创建订阅。记得将发布者定义更改回整数范围以解决错误。



### 你好，Future

就像你可以使用 `Just` 创建一个向订阅者发出单个值然后完成的发布者一样，`Future` 可以用于异步生成单个结果然后完成。 将这个新示例添加到你的 playground：

```swift
example(of: "Future") {
    func futureIncrement(
        integer: Int,
        afterDelay delay: TimeInterval) -> Future<Int, Never> {
            
        }
}
```

在这里，你创建了一个返回 `Int` 和 `Never` 类型的 `future` 的工厂函数； 意思是，它将发出一个整数并且永远不会失败。

在示例中，你还添加了一个订阅集，你将在其中存储对 `future` 的订阅。 对于长时间运行的异步操作，不存储订阅将导致在当前代码范围结束后立即取消订阅。 在 Playground 的情况下，那将是立即的。

接下来，填充函数的主体以创建 `future`：

```swift
Future<Int, Never> { promise in
    DispatchQueue.global().asyncAfter(deadline: .now() + delay) {
        promise(.success(integer + 1))
    }
}
```

这段代码定义了 `future`，它创建了一个 `promise`，然后你使用函数调用者指定的值执行该 `promise`，以在延迟后递增整数。

`Future` 是一个发布者，它最终会产生一个单一的值并完成，否则它将失败。它通过在值或错误可用时调用闭包来做到这一点，而闭包实际上就是 `promise`。  Command 单击 `Future` 并选择 Jump to Definition。 你将看到以下内容：

```swift
final public class Future<Output, Failure> : Publisher
  where Failure: Error {
  public typealias Promise = (Result<Output, Failure>) -> Void
  ...
}
```

Promise 是闭包的类型别名，它接收一个 `Result`，其中包含由 `Future` 发布的单个值或错误。

回到 Playground ，在 `futureIncrement` 的定义之后添加以下代码：

```swift
// 1
let future = futureIncrement(integer: 1, afterDelay: 3)

// 2
future
    .sink(receiveCompletion: { print($0) },
          receiveValue: { print($0) })
    .store(in: &subscriptions)
```

在这里，你：

1. 使用你之前创建的工厂函数创建一个未来，指定在三秒延迟后递增你传递的整数。

2. 订阅并打印接收到的值和完成事件，并将生成的订阅存储在订阅集中。 你将在本章稍后部分了解有关将订阅存储在集合中的更多信息，因此如果你不完全理解示例的这一部分，请不要担心。

运行 playground，你将看到打印的示例标题，然后是延迟三秒后的未来输出：

```
——— Example of: Future ———
2
finished
```

通过在 Playground 中输入以下代码来添加对未来的第二个订阅：

```swift
future
    .sink(receiveCompletion: { print("Second", $0) },
          receiveValue: { print("Second", $0) })
    .store(in: &subscriptions)
```

在运行 Playground 之前，在 `futureIncrement` 函数中的 `DispatchQueue` 块之前插入以下打印语句：

```swift
print("Original")
```

运行 Playground。 在指定的延迟之后，第二个订阅接收相同的值。Future 不会重新执行 `promise`； 相反，它共享或重放其输出。

```
——— Example of: Future ———
Original
2
finished
Second 2
Second finished
```

代码在订阅发生之前立即打印 Original。 发生这种情况是因为 `Future` 是贪婪的，意味着一旦创建就执行。 它不需要像 lazy 的常规发布者那样的订阅者。

在最后几个示例中，你一直在使用具有有限数量要发布的值的发布者，这些值是按顺序和同步发布的。

你开始使用的通知中心示例是发布者的示例，它可以无限期地和异步地继续发布值，前提是：

1. 底层通知发送者发出通知。

2. 有指定通知的订阅者。

是否有一种方法可以让你在自己的代码中做同样的事情？ 有！ 在继续之前，注释掉整个“Future”示例，因此每次运行 Playground 时都不会调用 Future——否则它的延迟输出会在最后一个示例之后打印。



### 你好，Subject

你已经学会了如何与发布者和订阅者合作，甚至如何创建自己的自定义订阅者。在本书的后面部分，你将学习如何创建自定义发布者。 我们接着来看是一个 `Subject`。

`Subject` 充当中间人，使非 Combine 命令式代码能够向 Combine 订阅者发送值。

将这个新示例添加到你的 Playground：

```swift
example(of: "PassthroughSubject") {
    // 1
    enum MyError: Error {
        case test
    }
    
    // 2
    final class StringSubscriber: Subscriber {
        typealias Input = String
        typealias Failure = MyError
        
        
        func receive(subscription: Subscription) {
            subscription.request(.max(2))
        }
        
        func receive(_ input: String) -> Subscribers.Demand {
            print("Received value", input)
            // 3
            return input == "World" ? .max(1) : .none
        }
        
        func receive(completion: Subscribers.Completion<MyError>) {
            print("Received completion", completion)
        }
        
    }
    
    // 4
    let subscriber = StringSubscriber()
}
```

使用此代码：

1. 定义自定义错误类型。

2. 定义一个接收字符串和 `MyError` 错误的自定义订阅者。

3. 根据收到的值调整需求。

4. 创建自定义订阅者的实例。

当输入为“World”时，在 `receive(_:)` 中返回 `.max(1)` 会导致新的最大值设置为 3（原始最大值加 1）。

除了定义自定义错误类型并根据接收到的值调整需求之外，这里没有什么新东西。更有趣的部分来了。将此代码添加到示例中：

```swift
// 5
let subject = PassthroughSubject<String, MyError>()

// 6
subject.subscribe(subscriber)

// 7
let subscription = subject
    .sink(
        receiveCompletion: { completion in
            print("Received completion (sink)", completion)
        },
        receiveValue: { value in
            print("Received value (sink)", value)
        }
    )
```

这段代码：

5. 创建一个 `String` 类型的 `PassthroughSubject` 实例和你定义的自定义错误类型。
   1. 为订阅者订阅 `subject`。

6. 使用 `sink` 创建另一个订阅。

`PassthroughSubject` 使你能够按需发布新值。 他们将愉快地传递这些值和完成事件。 与任何发布者一样，你必须提前声明它可以发出的值和错误的类型； 订阅者必须将这些类型与其输入和失败类型相匹配，才能订阅该 `PassthroughSubject`。

现在你已经创建了一个可以发送值和订阅以接收它们的传递主题，是时候发送一些值了。 将以下代码添加到你的示例中：

```swift
subject.send("Hello")
subject.send("World")
```

这使用 `subject` 的 `send` 方法发送两个值。

运行 playground，你会看到的：

```
——— Example of: PassthroughSubject ———
Received value Hello
Received value (sink) Hello
Received value World
Received value (sink) World
```

每个订阅者都会在发布时收到这些值。添加以下代码：

```swift
// 8
subscription.cancel()

// 9
subject.send("Still there?")
```

在这里：

8. 取消第二次订阅。

9. 发送另一个值。

运行 playground，如你所料，只有第一个订阅者会收到该值。 发生这种情况是因为你之前取消了第二个订阅者的订阅：

```
——— Example of: PassthroughSubject ———
Received value Hello
Received value (sink) Hello
Received value World
Received value (sink) World
Received value Still there?
```

将此代码添加到示例中：

```swift
subject.send(completion: .finished)
subject.send("How about another one?")
```

运行 playground，第二个订阅者没有收到“How about another one?”值，因为它在 `subject` 发送值之前收到了 completion 事件。第一个订阅者没有收到完成事件或值，因为它的订阅之前被取消了。

```
——— Example of: PassthroughSubject ———
Received value Hello
Received value (sink) Hello
Received value World
Received value (sink) World
Received value Still there?
Received completion finished
```

在发送完成事件的行之前添加以下代码。

```
subject.send(completion: .failure(MyError.test))
```

再次运行playground，你将看到以下打印到控制台：

```
——— Example of: PassthroughSubject ———
Received value Hello
Received value (sink) Hello
Received value World
Received value (sink) World
Received value Still there?
Received completion failure(...MyError.test)
```

> 注意：为便于阅读，错误类型已缩写。


第一个订阅者收到错误，但没有收到错误后发送的完成事件。 这表明，一旦发布者发送了一个 completion 事件——无论是正常完成还是错误——它就完成了。使用 `PassthroughSubject` 传递值是将命令式代码连接到 `Combine` 的声明性世界的一种方式。 但是，有时你可能还想在命令式代码中查看发布者的当前值——为此，你有一个恰当命名的`subject`：`CurrentValueSubject`。

你可以将多个订阅存储在 `AnyCancellable` 的集合中，而不是将每个 subscription 存储为一个值。 然后，该集合将在集合 deinit 之前自动取消添加到其中的每个 subscription。

将这个新示例添加到你的 playground：

```swift
example(of: "CurrentValueSubject") {
    // 1
    var subscriptions = Set<AnyCancellable>()
    
    // 2
    let subject = CurrentValueSubject<Int, Never>(0)
    
    // 3
    subject
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions) // 4
}
```

这是正在发生的事情：

1. 创建 subscription set。

2. 创建类型为 `Int` 和 `Never` 的 `CurrentValueSubject`。 这将发布整数并且永远不会发布错误，初始值为 0。

3. 创建 `subject` 的 `subscription` 并打印从 `subject` 接收到的值。

4. 将 `subscription` 存储在 subscription set 中（作为 inout 参数而不是 copy 传递）。

你必须使用初始值初始化当前值 subject；新订阅者立即获得该值或该 subject 发布的最新值。 运行 Playground 以查看实际情况：

```
——— Example of: CurrentValueSubject ———
0
```

现在，添加此代码以发送两个新值：

```swift
subject.send(1)
subject.send(2)
```

与 `PassthroughSubject` 不同，你可以随时向当前 subject 询问其 value。 添加以下代码以打印出 `subject` 的当前值：

```swift
print(subject.value)
```

正如你可能通过 subject 的类型名称推断的那样，你可以通过访问其 value 属性来获取其当前值。 运行 Playground，你会看到 2 第二次打印出来。

```
——— Example of: CurrentValueSubject ———
0
1
2
2
```

在当前值主题上调用 `send(_:)` 是发送新值的一种方法。 另一种方法是为其 value 属性分配一个新值。 添加此代码：

```swift
subject.value = 3
print(subject.value)
```

运行 Playground。 你会看到 2 和 3 分别打印了两次——一次由接收订阅者打印，一次是在将主题的值添加到主题后打印主题的值。

接下来，在本示例的最后，创建一个对当前值 subject 的新订阅：

```swift
subject
    .sink(receiveValue: { print("Second subscription:", $0) })
    .store(in: &subscriptions)
```

在这里，你创建订阅并打印接收到的值。 你还将该订阅存储在订阅集中。

你刚才读到 subscription set 会自动取消添加到其中的 subscription ，但是你如何验证这一点？ 你可以使用 `print()` 运算符，它将所有发布事件记录到控制台。

在主题和接收器之间的两个订阅中插入 `print()` 运算符。 每个订阅的开头应如下所示：

```swift
subject
  	.print()
  	.sink...
```

再次运行 Playground，你将看到整个示例的以下输出：

```
——— Example of: CurrentValueSubject ———
receive subscription: (CurrentValueSubject)
request unlimited
receive value: (0)
0
receive value: (1)
1
receive value: (2)
2
2
receive value: (3)
3
3
receive subscription: (CurrentValueSubject)
request unlimited
receive value: (3)
Second subscription: 3
receive cancel
```

代码在打印订阅处理程序中的每个值。 之所以会出现接收取消事件，是因为  subscription set  是在此示例的范围内定义的，因此它会在取消初始化时取消它包含的订阅。

那么，你可能想知道，你是否也可以将完成事件分配给 value 属性？ 通过添加以下代码尝试一下：

```swift
subject.value = .finished
```

没有！ 这会产生错误。 `CurrentValueSubject` 的 value 属性仅用于：value。 你仍然需要使用 `send(_:)` 发送完成事件。 将错误的代码行更改为以下内容：

```swift
subject.send(completion: .finished)
```

再次运行操场。 这次你将在底部看到以下输出：

```
receive finished
receive finished
```

两个订阅都接收完成事件而不是取消事件。 由于它们已经完成，你不再需要取消它们。



### 动态调整 demand

你之前了解到，在 `Subscriber.receive(_:)` 中调整需求是累加的。 你现在可以在更详细的示例中仔细研究它是如何工作的。 将这个新示例添加到 playground 上：

```swift
example(of: "Dynamically adjusting Demand") {
    final class IntSubscriber: Subscriber {
        typealias Input = Int
        typealias Failure = Never
        
        func receive(subscription: Subscription) {
            subscription.request(.max(2))
        }
        
        func receive(_ input: Int) -> Subscribers.Demand {
            print("Received value", input)
            
            switch input {
            case 1:
                return .max(2) // 1
            case 3:
                return .max(1) // 2
            default:
                return .none // 3
            }
        }
        
        func receive(completion: Subscribers.Completion<Never>) {
            print("Received completion", completion)
        }
    }
    
    let subscriber = IntSubscriber()
    
    let subject = PassthroughSubject<Int, Never>()
    
    subject.subscribe(subscriber)
    
    subject.send(1)
    subject.send(2)
    subject.send(3)
    subject.send(4)
    subject.send(5)
    sub
```



大部分代码与本章之前的示例类似，因此你将专注于 `receive(_:)` 方法。你不断调整自定义订阅者内部的需求：

1. 新的最大值为 4（原始最大值为 2 + 新最大值为 2）。

2. 新的最大值为 5（之前的 4 + 新的 1）。

3. 最大值仍然是 5（之前的 4 + 新的 0）。

运行 playground，你将看到以下内容：

```
——— Example of: Dynamically adjusting Demand ———
Received value 1
Received value 2
Received value 3
Received value 4
Received value 5
```

正如预期的那样，代码发出了五个值，但没有打印出第六个值。

在继续之前，你还需要了解一件更重要的事情：向订阅者隐藏有关发布者的详细信息。



### 类型擦除

有时你希望让订阅者订阅来自发布者的事件，而无法访问有关该发布者的其他详细信息。

最好用一个例子来证明这一点，所以把这个新的添加到你的 Playground 中：

```swift
example(of: "Type erasure") {
    // 1
    let subject = PassthroughSubject<Int, Never>()

    // 2
    let publisher = subject.eraseToAnyPublisher()
    
    // 3
    publisher
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
    
    // 4
    subject.send(0)
}
```

使用此代码：

1. 创建一个 `PassthroughSubject`。

2. 从该 `Subject` 创建一个类型擦除的发布者。

3. 订阅类型擦除的发布者。

4. 通过 `PassthroughSubject` 新值。

按住 Option 键单击发布者，你会看到它的类型为 `AnyPublisher<Int, Never>`。

`AnyPublisher` 是符合 `Publisher` 协议的类型擦除结构。 类型擦除允许你隐藏你可能不想向订阅者或下游发布者公开的发布者的详细信息，你将在下一节中了解这些信息。

你现在是否有一种似曾相识的感觉？ 如果是这样，那是因为你之前看到了另一种类型擦除的情况。 `AnyCancellable` 是一个符合 `Cancellable` 的类型擦除类，它允许调用者取消订阅，而无需访问底层 subscription 来执行其他操作。

当你想要对发布者使用类型擦除的一个示例是，当你想要使用一对公共和私有属性时，以允许这些属性的所有者在私有发布者上发送值，并让外部调用者只订阅但不能发送值。

`AnyPublisher` 没有 `send(_:)` 运算符，因此你不能直接向该发布者添加新值。

`eraseToAnyPublisher()` 运算符将提供的发布者包装在 `AnyPublisher` 的实例中，隐藏发布者实际上是 `PassthroughSubject` 的事实。 这也是必要的，因为你不能专门化 `Publisher` 协议，例如，你不能将类型定义为 `Publisher<UIImage, Never>`。

要证明发布者是类型擦除的并且你不能使用它来发送新值，请将此代码添加到示例中。

```swift
publisher.send(1)
```

你收到 `AnyPublisher<Int, Never>` 类型的错误值没有成员 `send`。 在继续之前注释掉那行代码。



### 桥接 Combine 发布者到 async/await

在 iOS 15 和 macOS 12 中，Swift 5.5 中的 Combine 框架新增了两个很棒的功能，可帮助你轻松地将 Combine 与 Swift 中的新 async/await 语法结合使用。

换句话说——你在本书中学到的所有 publisher、future 和 subject 都可以从你的现代 Swift 代码中使用。

将最后一个示例添加到你的 playground：

```swift
example(of: "async/await") {
    let subject = CurrentValueSubject<Int, Never>(0)
    
}
```

在此示例中，你将使用 `CurrentValueSubject`，但如前所述，API 可用于 `Future` 和任何符合 `Publisher `的类型。

你将使用 subject 来发出元素并使用 for 循环来迭代这些元素的异步序列。

你将订阅新异步任务中的值。请添加：

```swift
Task {
    for await element in subject.values {
        print("Element: \(element)")
    }
    print("Completed.")
}
```

Task 创建一个新的异步任务——闭包代码将与本代码示例中的其余代码异步运行。

此代码块中的关键 API 是 subject 的 values 属性。 values 返回一个异步序列，其中包含subject 或发布者发出的元素。你可以像上面那样在一个简单的 for 循环中迭代该异步序列。

一旦发布者完成，无论是成功还是失败，循环都会结束。

接下来，将此代码添加到当前示例中以发出一些值：

```swift
subject.send(1)
subject.send(2)
subject.send(3)
subject.send(completion: .finished)
```

这将发出 1、2 和 3，然后完成 subject。

这很好地包装了这个例子——发送完成的事件也会结束你的异步任务中的循环。 再次运行 Playground 代码，你将看到以下输出：

```
——— Example of: async/await ———
Element: 0
Element: 1
Element: 2
Element: 3
Completed.
```

在 `Future` 发出单个元素（如果有）的情况下，values 属性没有多大意义。 这就是为什么 `Future` 有一个 `value` 属性，你可以使用它来异步获取未来的结果。

你在本章中学到了很多东西，并且你将在本书的其余部分及以后的内容中运用这些新技能。 但没那么快！ 是时候练习你刚刚学到的东西了。



### 挑战

完成挑战有助于巩固你在本章中学到的知识。练习文件下载中有挑战的初始版本和最终版本。

#### 挑战：二十一点发牌

在挑战文件夹中打开 Starter.playground，选择 SupportCode.swift：

查看此挑战的辅助代码，包括

- 一个卡片数组，包含 52 个元组，代表一副标准的卡片组。

- 两种类型别名：Card 是 String 和 Int 的元组，Hand 是 Cards 数组。

- 手头有两个辅助属性：cardString 和 points。

- HandError 错误枚举。

在 playground 主页面中，在注释 `// Add code to update dealtHand here` 下方添加代码，从 hand 的 points 属性返回的结果。如果结果大于 21，则通过 dealtHand subject 发送 HandError.busted。 否则，发送手牌值。

同样在 playground 主页面中，在注释 `// Add subscription to dealtHand here` 之后立即添加代码以订阅 dealtHand 并处理接收值和错误。

对于接收到的值，打印一个字符串，其中包含手牌的 `cardString` 和 `points` 属性的结果。

对于错误，将其打印出来。 提示：你可以在 `receivedCompletion` 块中接收 `.finished` 或 `.failure`，因此你需要区分该完成是否失败。

HandError 符合 `CustomStringConvertible`，打印它会得到用户友好的错误消息。 你可以像这样使用它：

```swift
if case let .failure(error) = $0 {
  	print(error)
}
```

对 `deal(_:)` 的调用当前通过了 3，因此每次运行 Playground 时都会发三张牌。

在真正的二十一点游戏中，你最初会收到两张牌，然后你必须决定再拿一张或多张牌，称为命中，直到你命中 21 或破产。 对于这个简单的示例，你只需立即获得三张牌。



#### 参考方案

你怎么做的？ 要完成这个挑战，你需要添加两件事。 首先是在 deal 函数中更新 dealtHand 发布者，检查手牌点数，如果超过 21 则发送错误：

```swift
// Add code to update dealtHand here
if hand.points > 21 {
  	dealtHand.send(completion: .failure(.busted))
} else {
  	dealtHand.send(hand)
}
```

接下来，你需要订阅 dealtHand 并打印接收到的值或如果是错误的完成事件：

```swift
_ = dealtHand
		.sink(receiveCompletion: {
    		if case let .failure(error) = $0 {
      			print(error)
    		}
  	}, receiveValue: { hand in
    	print(hand.cardString, "for", hand.points, "points")
  	})
```

每次运行 Playground 时，你都会得到一个新的手，并输出类似于以下内容：

```
——— Example of: Create a Blackjack card dealer ———
🃕🃆🃍 for 21 points
```



### 关键点

- 发布者随着时间的推移将一系列值同步或异步传输给一个或多个订阅者。

- 订阅者可以订阅发布者以接收值；但是，订阅者的输入和失败类型必须与发布者的输出和失败类型相匹配。

- 你可以使用两个内置运算符来订阅发布者：`sink(_:_:)` 和 `assign(to:on:)`。

- 订阅者每次收到价值时可能会增加对价值的需求，但不能减少需求。

- 要释放资源并防止不必要的副作用，请在完成后取消每个 subscription。

- 你还可以将订阅存储在 `AnyCancellable` 的实例或集合中，以便在取消初始化时接收自动取消。

- 你使用 `Future` 在以后异步接收单个值。

- `Subject` 是发布者，它使外部调用者能够异步向订阅者发送多个值，有或没有起始值。

- 类型擦除可防止调用者访问基础类型的其他详细信息。

- 使用 `print()` 操作符将所有发布事件记录到控制台并查看发生了什么。



### 然后去哪儿？

恭喜！ 通过完成本章，你向前迈出了一大步。 你学习了如何与发布者一起发送值和 completion 事件，以及如何使用订阅者接收这些值和事件。 接下来，你将学习如何操纵来自发布者的值，以帮助过滤、转换或组合它们。
