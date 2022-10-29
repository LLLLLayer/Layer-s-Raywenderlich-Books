# 第四部分：Combine 进阶

我们已经学习了 Combine 基础，是时候学习一些更高级的概念和主题。你将首先学习如何将 SwiftUI 与 Combine 结合使用，以构建真正的反应式和流畅的 UI 体验，然后转而学习如何正确处理你的 Combine 应用程序中的错误。 然后，你将了解调度程序(Scheduler)，这是在不同执行上下文中调度工作背后的核心概念，并跟进如何创建自己的自定义发布者并通过了解背压(Backpressure)来处理订阅者的需求。

最后，拥有一个流畅的代码库固然很好，但如果没有经过很好的测试就没有多大帮助，因此你将通过学习如何正确测试新的 Combine 代码来结束本部分。



## 第 15 节：实践：Combine & SwiftUI

SwiftUI 是 Apple 用于以声明方式构建应用程序 UI 的最新技术。这与旧的 UIKit 和 AppKit 框架有很大的不同。它为构建用户界面提供了一种非常简洁且易于阅读和编写的语法。SwiftUI 语法清楚地代表了你想要构建的视图层次结构：

```swift
HStack(spacing: 10) {
  Text("My photo")
  Image("myphoto.png")
    .padding(20)
    .resizable()
}
```

你可以轻松地直观地解析层次结构。 HStack 视图——一个水平堆栈——包含两个子视图：一个文本视图和一个图像视图。

每个视图都可以有一个修饰符列表——它们是你在视图上调用的方法。在上面的示例中，你使用视图修饰符 padding(20) 在图像周围添加 20 个填充点。此外，你还可以使用 resizable() 来启用图像内容的大小调整。

SwiftUI 还统一了构建跨平台 UI 的方法。例如，一个 Picker 控件在你的 iOS 应用程序中显示一个新的模式视图，允许用户从列表中选择一个项目，但在 macOS 上，相同的 Picker 控件显示一个 Dropbox。

数据表单的快速代码示例可能是这样的：

```swift
VStack {
  TextField("Name", text: $name)
  TextField("Proffesion", text: $profession)
  Picker("Type", selection: $type) {
    Text("Freelance")
    Text("Hourly")
    Text("Employee")
  }
}
```

此代码将在 iOS 上创建两个单独的视图。类型选择器控件将是一个按钮，将用户带到一个单独的屏幕，其中包含如下选项列表：

![image-20221015154833446](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015154833446.png)

然而，在 macOS 上，SwiftUI 会考虑 mac 上丰富的 UI 屏幕空间，并创建一个带有下拉菜单的单一表单：

![image-20221015154850556](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015154850556.png)



最后，在 SwiftUI 中，屏幕上呈现的用户界面是你的状态的函数。你维护此状态的单个副本，称为“事实来源”，并且 UI 是从该状态动态派生的。幸运的是，Combine 发布者可以轻松地作为数据源插入 SwiftUI 视图。



### 你好，SwiftUI！

如上一节所述，使用 SwiftUI 时，你以声明方式描述用户界面，并将渲染留给框架。

你为 UI 声明的每个视图（文本标签、图像、形状等）都符合 View 协议。 View 的唯一要求是一个名为 body 的属性。

每当你更改数据模型时，SwiftUI 都会向你的每个视图询问它们当前的主体表示。这可能会根据你最新的数据模型更改而发生变化。然后，该框架通过仅计算受模型更改影响的视图来构建视图层次结构以在屏幕上呈现，从而产生高度优化和有效的绘图机制。

实际上，SwiftUI 使 UI “快照”由数据模型的任何更改触发，如下所示：

![image-20221015155023945](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015155023945.png)

在本章中，你将完成许多任务，这些任务涵盖了 Combine 和 SwiftUI 之间的互操作以及一些 SwiftUI 基础知识。



### 内存管理

上述 UI 工作的内容很大一部分是内存管理的转变。



### 无数据重复

让我们看一个例子来说明这意味着什么。在使用 UIKit/AppKit 时，你可以粗略地说，将你的代码在数据模型、某种控制器和视图之间分离：

![image-20221015155348997](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015155348997.png)

这三种类型可以有几个相似的特征。它们包括数据存储、支持可变性、可以是引用类型等等。

假设你想在屏幕上显示当前天气。对于此示例，假设模型类型是一个名为 Weather 的结构，并将当前条件存储在一个名为 conditions 的文本属性中。要向用户显示该信息，你需要创建另一种类型的实例，即 UILabel，并将条件的值复制到标签的文本属性中。

现在，你有两个你使用的值的副本。一个在你的模型类型中，另一个存储在 UILabel 中，只是为了在屏幕上显示它：

![image-20221015155511414](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015155511414.png)

文本和条件之间没有联系或绑定。你只需将字符串值复制到你需要的任何地方。

现在，你已向 UI 添加了依赖项。屏幕上信息的新鲜度取决于 Weather.conditions。每当条件属性更改时，你有责任使用 Weather.conditions 的新副本手动更新标签的文本属性。

SwiftUI 消除了为了在屏幕上显示而复制数据的需要。能够从 UI 中卸载数据存储，可以让你在模型中的一个位置有效地管理数据，并且永远不会让应用程序的用户在屏幕上看到陈旧的信息。

**更少需要“控制”你的视图**

作为额外的奖励，消除在模型和视图之间使用“胶水”代码的需要也可以让你摆脱大部分视图控制器代码！

在本章中，你将学习：

- 简要介绍用于构建声明性 UI 的 SwiftUI 语法基础知识。

- 如何声明各种类型的 UI 输入并将它们连接到它们的“事实来源”。

- 如何使用 Combine 构建数据模型并将数据通过管道传输到 SwiftUI。

> 注意：如果你想了解有关 SwiftUI 的更多信息，请考虑查看 SwiftUI by Tutorials (https://bit.ly/2L5wLLi) 以获得深入的学习体验。


现在，对于我们的功能演示：Combine 和 SwiftUI！



### “News”入门

本章的入门项目已经包含一些代码，以便你可以专注于编写连接 Combine 和 SwiftUI 的代码。

该项目还包括一些文件夹，你可以在其中找到以下内容：

- App 包含 main app type。

- Network 包括上一章完整的 Hacker News API。

- Model 是你可以找到简单模型类型的地方，例如 Story、FilterKeyword 和 Settings。 此外，这是 ReaderViewModel 所在的位置，这是主新闻阅读器视图使用的模型类型。

- View 包含应用程序视图，在 View/Helpers 中，你会发现一些简单的可重用组件，如 button、badge 等。

- 最后，在 Util 中有一个帮助类型，允许你轻松地从磁盘读取和写入 JSON 文件。

完成的项目将显示 Hacker News  Story 列表，并允许用户管理关键字过滤器：

![image-20221015160403861](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015160403861.png)



### 初体验管理视图状态

构建并运行启动项目，你将在屏幕上看到一个空表和一个标题为“设置”的单条按钮：

![image-20221015160510604](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015160510604.png)

这是你开始的地方。 要了解通过更改数据与 UI 交互的工作原理，你将让设置按钮在点击时显示 SettingsView。

打开 View/ReaderView.swift，其中包含显示主应用程序界面的 ReaderView 视图。

该类型已经包含一个名为 presentingSettingsSheet 的属性，它是一个简单的布尔值。 更改此值将显示或关闭设置视图。 向下滚动源代码并找到注释 Set presentingSettingsSheet to true here，替换为：

```swift
self.presentingSettingsSheet = true
```

添加此行后，你将看到以下错误：

![image-20221015160706352](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015160706352.png)

确实 self 是不可变的，因为 view 的 body 是动态属性，因此不能改变 ReaderView。

SwiftUI 提供了许多内置的属性包装器来帮助你指示给定属性是你的状态的一部分，并且对这些属性的任何更改都应该触发一个新的 UI“快照”。

让我们看看这在实践中意味着什么。 调整普通的旧 presentingSettingsSheet 属性，使其如下所示：

```swift
@State var presentingSettingsSheet = false
```

@State 属性包装器：

1. 将属性存储移出视图，因此修改 presentingSettingsSheet 不会改变 self。

2. 将属性标记为本地存储。换句话说，它表示该数据段由视图拥有。

3. 向 ReaderView 添加一个名为 $presentingSettingsSheet 的发布者，有点像 @Published，你可以使用它来订阅该属性或将其绑定到 UI 控件或其他视图。

将 @State 添加到 presentingSettingsSheet 后，错误将清除，因为编译器知道你可以从  non-mutating 上下文中修改此特定属性。

最后，要使用 presentingSettingsSheet，你需要声明新状态如何影响UI。在这种情况下，你将向视图层次结构添加一个 sheet(...) 视图修饰符，并将 $presentingSettingsSheet 绑定到工作表。每当你更改 presentingSettingsSheet 时，SwiftUI 将采用当前值并根据布尔值呈现或关闭你的视图。

找到 // Present the Settings sheet here 将其替换为：

```swift
.sheet(isPresented: self.$presentingSettingsSheet, content: {
  SettingsView()
})
```

sheet(isPresented:content:) 修饰符接受一个 Bool 发布者和一个视图，以在演示发布者发出 true 时呈现。

构建并运行项目。点击设置，你的新演示文稿将显示目标视图：

![image-20221015161203408](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015161203408.png)



### 获取最新 Storiy

接下来，是时候回到一些 Combine 代码了。 在本节中，你将合并现有的 ReaderViewModel 并将其连接到 API 。

打开 Model/ReaderViewModel.swift。 在顶部，插入：

```swift
import Combine
```

这段代码自然允许你在 ReaderViewModel.swift 中使用 Combine 类型。 现在，向 ReaderViewModel 添加一个新的订阅属性来存储你的所有订阅：

```swift
private var subscriptions = Set<AnyCancellable>()
```

有了所有扎实的准备工作，现在是时候创建一种新方法并使用网络 API。 将以下空方法添加到 ReaderViewModel：

```swift
func fetchStories() {

}
```

在此方法中，你将订阅 API.stories() 并将服务器响应存储在模型类型中。 你应该从上一章中熟悉此方法。

在 fetchStories() 中添加以下内容：

```swift
api
  .stories()
  .receive(on: DispatchQueue.main)
```

你使用 receive(on:) 操作符接收主队列上的任何输出。 可以说，你可以将线程管理留给 API 的使用者。但是，由于在 ReaderViewModel 的情况下肯定是 ReaderView，因此你在此处进行优化并切换到主队列以准备提交对 UI 的更改。

接下来，你将使用 sink(...) 订阅者在模型中存储故事和任何发出的错误。 附加：

```swift
.sink(receiveCompletion: { completion in
  if case .failure(let error) = completion {
    self.error = error
  }
}, receiveValue: { stories in
  self.allStories = stories
  self.error = nil
})
.store(in: &subscriptions)
```

首先，你检查 completion 是否失败。 如果是这样，你将关联的错误存储在 self.error 中。 如果你从 Story 发布者那里收到值，则将它们存储在 self.allStories 中。

这就是你要在本节中添加到模型的所有逻辑。 fetchStories() 方法现已完成，你可以在屏幕上显示 ReaderView 后立即“启动”模型。

为此，打开 App/App.swift 并向 ReaderView 添加一个新的 onAppear(...) 视图修饰符，如下所示：

```swift
ReaderView(model: viewModel)
  .onAppear {
    viewModel.fetchStories()
  }
```

现在，ReaderViewModel 并没有真正连接到 ReaderView，所以你不会在屏幕上看到任何变化。 但是，要快速验证一切是否按预期工作，请执行以下操作：返回 Model/ReaderViewModel.swift 并将 didSet 处理程序添加到 allStories 属性：

```swift
private var allStories = [Story]() {
  didSet {
    print(allStories.count)
  }
}
```

运行应用程序并观察控制台。 你应该会看到一个令人放心的输出，如下所示：

```
1
2
3
4
...
```

你可以删除刚刚添加的 didSet 处理程序，以防你不想在每次运行应用程序时都看到该输出。



### 将 ObservableObject 用于 model

ObservableObject 是一种使普通旧数据模型可观察的协议，并让观察者 SwiftUI 视图知道数据已更改，因此它能够重建依赖于该数据的任何用户界面。

该协议要求类型实现一个名为 objectWillChange 的发布者，该发布者在类型的状态即将改变时发出。

协议中已经有该发布者的默认实现，因此在大多数情况下，你不必向数据模型添加任何内容。当你将 ObservableObject 一致性添加到你的类型时，默认协议实现将在你的任何 @Published 属性发出时自动发出！

打开 ReaderViewModel.swift 并将 ObservableObject 一致性添加到 ReaderViewModel，所以它看起来像这样：

```swift
class ReaderViewModel: ObservableObject {
```

接下来，你需要考虑数据模型的哪些属性构成了它的状态。你当前在 sink(...) 订阅者中更新的两个属性是 allStories 和 error。你会认为那些状态改变是值得的。

> 注意：还有第三个属性叫做过滤器。暂时忽略它，稍后你会回到它。

调整 allStories 以包含 @Published 属性包装器，如下所示：

```swift
@Published private var allStories = [Story]()
```

然后，对错误执行相同的操作：

```swift
@Published var error: API.Error? = nil
```

本节的最后一步是，由于 ReaderViewModel 现在符合 ObservableObject，因此将数据模型实际绑定到 ReaderView。

打开 View/ReaderView.swift 并将 @ObservedObject 属性包装器添加到行 var model: ReaderViewModel 中，如下所示：

```swift
@ObservedObject var model: ReaderViewModel
```

你绑定模型，以便在其状态更改时，你的视图将接收最新数据并生成其新的 UI“快照”。

@ObservedObject 包装器执行以下操作：

1. 从视图中删除属性存储，并改为使用与原始模型的绑定。换句话说，它不会复制数据。

2. 将属性标记为外部存储。换句话说，它表示该数据不属于视图。

3. 与@Published 和@State 一样，它向属性添加了一个发布者，以便你可以订阅它和/或在视图层次结构中进一步绑定到它。

通过添加@ObservedObject，你可以使模型动态化。这意味着它会在你的视图模型从 Hacker News 服务器获取故事时获取所有更新。事实上，现在运行应用程序，你将看到视图在模型获取 Story 时自行刷新：

![image-20221015163909480](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015163909480.png)



### 显示错误

你还将以与显示获取的故事相同的方式显示错误。目前，视图模型将任何错误存储在其错误属性中，你可以将其绑定到屏幕上的 UI 警报。

打开 View/ReaderView.swift 并找到注释 // Display errors here。将此注释替换为以下代码以将模型绑定到提示视图：

```swift
.alert(item: self.$model.error) { error in
  Alert(
    title: Text("Network error"), 
    message: Text(error.localizedDescription),
    dismissButton: .cancel()
  )
}
```

alert(item:) 修饰符控制屏幕上的警报显示。 它接受一个带有可选输出的绑定，称为项目。 每当该绑定源发出非零值时，UI 都会显示警报视图。

模型的错误属性默认为 nil，并且只有在模型从服务器获取故事时遇到错误时才会设置为非 nil 错误值。 这是呈现警报的理想方案，因为它允许你将错误直接绑定为 alert(item:) 输入。

要对此进行测试，请打开 Network/API.swift 并将 baseURL 属性修改为无效 URL，例如 https://123hacker-news.firebaseio.com/v0/。

再次运行应用程序，一旦对故事端点的请求失败，你将看到错误警报显示：

![image-20221015165117006](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015165117006.png)

在继续完成下一部分之前，花点时间将你的更改恢复为 baseURL，以便你的应用程序再次成功连接到服务器。



### 订阅外部发布者

有时你不想走 ObservableObject/ObservedObject 路线，因为你想做的只是订阅一个发布者并在你的 SwiftUI 视图中接收它的值。对于像这种更简单的情况，不需要创建额外的类型——你可以简单地使用 onReceive(_) 视图修饰符。它允许你直接从你的视图订阅发布者。

如果你现在运行该应用程序，你将看到每个故事都有一个相对时间，并包含在故事作者的姓名旁边：

![image-20221015165231389](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015165231389.png)

那里的相对时间有助于立即将故事的“新鲜度”传达给用户。然而，一旦呈现在屏幕上，信息会在一段时间后变得陈旧。如果用户长时间打开应用程序，“1 分钟前”可能会关闭一段时间。

在本节中，你将使用计时器发布者定期触发 UI 更新，以便每一行都可以重新计算并显示正确的时间。

代码现在的工作方式如下：

- ReaderView 有一个名为 currentDate 的属性，该属性在创建视图时使用当前日期设置一次。

- Story 列表中的每一行都包含一个 PostedBy(time:user:currentDate:) 视图，该视图使用 currentDate 的值编译作者和时间信息。

要定期“刷新”屏幕上的信息，你将添加一个新的计时器发布者。每次它发出时，你都会更新 currentDate。此外，正如你可能已经猜到的那样，你会将 currentDate 添加到视图的状态中，以便在它发生变化时触发新的 UI“快照”。

要与发布者合作，首先在 ReaderView.swift 的顶部添加：

```swift
import Combine
```



然后，向 ReaderView 添加一个新的发布者属性，该属性创建一个新的计时器发布者，只要有人订阅它就可以立即使用：

```
private let timer = Timer.publish(every: 10, on: .main, in: .common)
  .autoconnect()
  .eraseToAnyPublisher()
```

正如你在本书前面已经了解到的那样，其返回一个可连接的发布者。这是一种“休眠”发布者，需要订阅者连接到它才能激活它。上面你使用 autoconnect() 来指示发布者在订阅时自动“唤醒”。

现在剩下的是在每次计时器发出时更新 currentDate 。你将使用名为 onReceive(_) 的 SwiftUI 修饰符，其行为与 sink(receiveValue:) 订阅者非常相似。向下滚动一点，找到 // Add timer here 将其替换为：

```swift
.onReceive(timer) {
  self.currentDate = $0
}
```

计时器发出当前日期和时间，因此你只需将该值分配给 currentDate。这样做会产生一个有点熟悉的错误：

自然，发生这种情况是因为你无法从非变异上下文中对属性进行变异。和以前一样，你将通过将 currentDate 添加到视图的本地存储状态来解决这个难题。

像这样向属性添加 @State 属性包装器：

```swift
@State var currentDate = Date()
```

这样，对 currentDate 的任何更新都会触发一个新的 UI“快照”，并强制每一行重新计算故事的相对时间，并在必要时更新文本。

再次运行该应用程序并使其保持打开状态。 记下头条新闻是多久前发布的，这是我尝试发布的内容：

等待至少一分钟，你将看到可见行将其信息更新为当前时间。 橙色时间徽章仍将显示故事发布的时间，但标题下方的文本将更新为正确的“...分钟前”文本：

![image-20221015165735835](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015165735835.png)

除了让发布者在你的视图上具有属性外，你还可以通过视图的初始化程序或环境将 Combine 模型中的任何发布者注入到视图中。然后，只需以与上述相同的方式使用 onReceive(...) 即可。



### 初始化应用程序的设置

在本章的这一部分，你将继续使设置视图工作。在处理 UI 本身之前，你需要先完成 Settings 类型的实现。

打开 Model/Settings.swift，你会看到，目前，该类型几乎是简陋的。它包含一个包含 FilterKeyword 值列表的属性。

现在，打开 Model/FilterKeyword.swift。 FilterKeyword 是一种辅助模型类型，它包装了一个关键字，用作主阅读器视图中故事列表的过滤器。它符合 Identifiable 要求，它需要一个唯一标识每个实例的 id 属性，例如当你在 SwiftUI 代码中使用这些类型时。如果你分别仔细阅读 Network/API.swift 和 Model/Story.swift 中的 API.Error 和 Story 定义，你会发现这些类型也符合 Identifiable。

你需要将普通的旧模型设置转换为现代类型，以便与你的组合和 SwiftUI 代码一起使用。

通过在 Model/Settings.swift 顶部添加开始：

```swift
import Combine
```

然后，通过将 @Published 属性包装器添加到 keywords 来为 keywords 添加发布者，如下所示：

```swift
@Published var keywords = [FilterKeyword]()
```

现在，其他类型可以订阅设置的当前关键字。你还可以通过管道将关键字列表传递给接受绑定的视图。

最后，为了启用对设置的观察，使类型符合 ObservableObject ，如下所示：

```swift
final class Settings: ObservableObject {
```

无需添加任何其他内容即可使 ObservableObject 符合性工作。默认实现将在 $keywords 发布者发出的任何时候发出。

这就是你如何通过几个简单的步骤将“设置”变成类固醇的模型类型。现在，你可以将其插入应用程序中的其余反应式代码中。

要绑定应用程序的设置，你将在应用程序中对其进行实例化。打开 App/App.swift 并向 HNReader 添加一个新属性：

```swift
let userSettings = Settings()
```

像往常一样，你还需要一个可取消的集合来存储你的订阅。 为此向 HNReader 添加一个属性：

```swift
private var subscriptions = Set<AnyCancellable>()
```

现在，你可以将 Settings.keywords 绑定到 ReaderViewModel.filter 以便主视图不仅会收到初始关键字列表，还会在用户每次编辑关键字列表时收到更新列表。

你将在初始化 HNReader 时创建该绑定。 向该类型添加一个新的初始化程序：

```swift
init() {
  userSettings.$keywords
    .map { $0.map { $0.value } }
    .assign(to: \.filter, on: viewModel)
    .store(in: &subscriptions)
}
```

你订阅 userSettings.$keywords，它输出 [FilterKeyword]，并通过获取每个关键字的 value 属性将其映射到 [String]。然后，将结果值分配给 viewModel.filter。

现在，无论何时更改 Settings.keywords 的内容，与视图模型的绑定最终都会导致 ReaderView 的新 UI“快照”的生成，因为视图模型是其状态的一部分。

到目前为止，绑定有效。但是，你仍然必须添加 filter 属性才能成为 ReaderViewModel 状态的一部分。你将这样做，以便每次更新关键字列表时，新数据都会转发到视图。

为此，请打开 Model/ReaderViewModel.swift 并添加 @Published 属性包装器以进行过滤，如下所示：

```swift
@Published var filter = [String]()
```

从设置到视图模型再到视图的完整绑定现已完成！

这非常方便，因为在下一节中，你会将 Settings 视图连接到 Settings 模型，并且用户对关键字列表所做的任何更改都将触发整个绑定和订阅链，最终刷新主应用程序视图 Story 列表，例如：

![image-20221015171432194](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015171432194.png)



### 编辑关键字列表

在本章的最后一部分，你将了解 SwiftUI 环境。环境是自动注入视图层次结构的发布者共享池。



#### 系统环境

环境包含系统注入的发布者，如当前日历、布局方向、语言环境、当前时区等。如你所见，这些都是可能随时间变化的值。因此，如果你声明视图的依赖项，或者将它们包含在你的状态中，则视图将在依赖项更改时自动重新呈现。

要尝试观察其中一项系统设置，请打开 View/ReaderView.swift 并向 ReaderView 添加一个新属性：

```swift
@Environment(\.colorScheme) var colorScheme: ColorScheme
```

你使用 @Environment 属性包装器，它定义了环境的哪个键应该绑定到 colorScheme 属性。现在，这个属性是视图状态的一部分。每次系统外观模式在明暗之间切换，反之亦然，SwiftUI 都会重新渲染你的视图。

此外，你将可以访问视图主体中的最新配色方案。因此，你可以在明暗模式下以不同方式渲染它。

向下滚动并找到设置 Story 链接颜色的行 .foregroundColor(Color.blue)。将该行替换为：

```swift
.foregroundColor(self.colorScheme == .light ? .blue : .orange)
```

现在，根据 colorScheme 的当前值，链接将是蓝色或橙色。

通过将系统外观更改为深色来尝试这种新的代码奇迹。在 Xcode 中，打开 Debug ► View Debugging ► Configure Environment Overrides... 或点击 Xcode 底部工具栏上的 Environment Overrides 按钮。然后，打开界面样式旁边的开关。

![image-20221015171722618](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015171722618.png)

#### 自定义环境对象

就像通过 @Environment(_) 观察系统设置一样酷，但这并不是 SwiftUI 环境必须提供的全部。事实上，你也可以环境化你的对象！

这非常方便。尤其是当你有深度嵌套的视图层次结构时。将模型或其他共享资源插入环境中，无需通过大量视图进行依赖注入，直到你到达实际需要数据的深度嵌套视图。

你在视图环境中插入的对象可自动用于该视图的任何子视图及其所有子视图。

这听起来像是与应用程序的所有视图共享用户设置的绝佳机会，以便他们可以使用用户的故事过滤器。

将依赖项注入所有视图的地方是主应用程序文件。这是你之前创建 Settings 的 userSettings 实例并将其 $keywords 绑定到 ReaderViewModel 的位置。现在，你还将将 userSettings 注入到环境中。

打开 App/App.swift 并将 environmentObject 视图修改器添加到 ReaderView，方法是在 ReaderView(model: viewModel) 下方添加：

```swift
.environmentObject(userSettings)
```

environmentObject 修饰符是一个视图修饰符，它将给定对象插入到视图层次结构中。由于你已经有一个设置实例，你只需将其发送到环境即可。

接下来，你需要将环境依赖项添加到要使用自定义对象的视图中。打开 View/SettingsView.swift 并使用 @EnvironmentObject 包装器添加一个新属性：

```swift
@EnvironmentObject var settings: Settings
```

设置属性将自动填充环境中的最新用户设置。

对于你自己的对象，你不需要像系统环境那样指定密钥路径。 @EnvironmentObject 会将属性类型（在本例中为设置）与存储在环境中的对象匹配并找到正确的对象。

现在，你可以像使用任何其他视图状态一样使用 settings.keywords。你可以直接获取该值、订阅它或将其绑定到其他视图。

要完成 SettingsView 功能，你将显示关键字列表并启用从列表中添加、编辑和删除关键字。

找到以下行：

```swift
ForEach([FilterKeyword]()) { keyword in
```

并将其替换为：

```swift
ForEach(settings.keywords) { keyword in
```

更新后的代码将使用屏幕列表的过滤器关键字。但是，这仍将显示一个空列表，因为用户无法添加新关键字。

初始项目包括一个用于添加关键字的视图。因此，你只需在用户点击 + 按钮时显示它。 + 按钮操作在 SettingsView 中设置为 addKeyword()。

滚动到 addKeyword() 方法并在其中添加：

```swift
presentingAddKeywordSheet = true
```

presentingAddKeywordSheet 是一个 published 的属性，与本章前面已经使用过的属性非常相似，用于显示提示。你可以在源代码中稍微向上看到演示文稿声明：.sheet(isPresented: $presentingAddKeywordSheet)。

要尝试手动将对象注入给定视图的工作原理，请切换到 View/ReaderView.swift 并找到你提供 SettingsView 的位置——它是你只需创建一个新实例的单行代码，如下所示：SettingsView()。

与将设置注入 ReaderView 的方式相同，你也可以在此处注入它们。向 ReaderView 添加一个新属性：

```swift
@EnvironmentObject var settings: Settings
```

然后，直接在 SettingsView() 下添加 .environmentObject 修饰符：

```swift
.environmentObject(self.settings)
```

现在，你声明了对 Settings 的 ReaderView 依赖项，并通过环境将该依赖项传递给 SettingsView。在这种特殊情况下，你也可以将它作为参数传递给 SettingsView 的 init。

在继续之前，请再次运行该应用程序。你应该能够点击设置并看到 SettingsView 弹出。

现在，切换回 View/SettingsView.swift 并按照最初的预期完成列表编辑操作。

在 sheet(isPresented: $presentingAddKeywordSheet) 中，已经为你创建了一个新的 AddKeywordView。这是一个包含在启动项目中的自定义视图，它允许用户输入一个新的关键字并点击一个按钮将其添加到列表中。

AddKeywordView 接受一个回调，当用户点击按钮添加新关键字时，它将调用该回调。在 AddKeywordView 的空完成回调中添加：

```swift
let new = FilterKeyword(value: newKeyword.lowercased())
self.settings.keywords.append(new)
self.presentingAddKeywordSheet = false
```

你创建一个 keyword，将其添加到用户设置中，最后关闭显示的工作表。

请记住，在此处将关键字添加到列表中将更新设置模型对象，进而更新阅读器视图模型并刷新阅读器视图。全部按照你的代码中的声明自动进行。

结束 SettingsView，让我们添加删除和移动关键字。查找  // List editing actions 并将其替换为：

```swift
.onMove(perform: moveKeyword)
.onDelete(perform: deleteKeyword)
```

此代码将 moveKeyword() 设置为当用户在列表中向上或向下移动其中一个关键字时的处理程序，并在用户向右滑动删除关键字时将 deleteKeyword() 设置为处理程序。

在当前为空的 moveKeyword(from:to:) 方法中，添加：

```
guard let source = source.first,
      destination != settings.keywords.endIndex else { return }

settings.keywords
  .swapAt(source,
          source > destination ? destination : destination - 1)
```

在 deleteKeyword(at:) 中，添加：

```swift
settings.keywords.remove(at: index.first!)
```

这就是你在列表中启用编辑所需的全部内容！最后一次构建并运行应用程序，你将能够完全管理 Filter 过滤器，包括添加、移动和删除关键字：

![image-20221015174400897](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015174400897.png)



此外，当你导航回故事列表时，你将看到设置与你的订阅和绑定一起在应用程序中传播，并且列表仅显示与你的过滤器匹配的故事。标题还将显示匹配故事的数量：

![image-20221015174540737](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015174540737.png)



### 挑战

本章包括两个完全可选的 SwiftUI 练习，你可以选择完成这些练习。你也可以将它们搁置一旁，在接下来的章节中继续讨论更令人兴奋的 Combine 主题。

#### 挑战 1：在阅读器视图中显示过滤器

在第一个挑战中，你将在 ReaderView 的故事列表标题中插入过滤器关键字列表。目前，标题始终显示“显示所有故事”。更改该文本以显示关键字列表，以防用户添加任何关键字，如下所示：

![image-20221015174620950](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221015174620950.png)

#### 挑战 2：在应用启动之间保持过滤器

启动项目包括一个名为 JSONFile 的辅助类型，它提供两种方法：loadValue(named:) 和 save(value:named:)。

使用此类型：

- 每当用户通过将 didSet 处理程序添加到 Settings.keywords 来修改过滤器时，将关键字列表保存在磁盘上。

- 在 Settings.init() 中从磁盘加载关键字。

这样，用户的过滤器将在应用程序启动之间保持不变，就像在真实应用程序中一样。

如果你不确定这些挑战中的任何一个的解决方案，或者需要一些帮助，请随时查看项目/挑战文件夹中的已完成项目。



### 关键点

- 使用 SwiftUI，你的 UI 是你状态的函数。你可以通过提交对声明为视图状态的数据以及其他视图依赖项的更改来呈现自己的 UI。你学习了在 SwiftUI 中管理状态的各种方法：

- 使用@State 将本地状态添加到视图，并使用@ObservedObject 在你的组合代码中添加对外部 ObservableObject 的依赖。

- 使用 onReceive 视图修饰符直接订阅外部发布者。

- 使用 @Environment 将依赖项添加到系统提供的环境设置之一，并使用 @EnvironmentObject 为你自己的自定义环境对象添加依赖项。



### 接下来去哪儿？

恭喜你使用 SwiftUI 和 Combine 轻松搞定！我希望你现在意识到两者之间的联系是多么紧密和强大，以及 Combine 如何在 SwiftUI 的响应式功能中发挥关键作用。

即使你应该始终致力于编写无错误的应用程序，但世界很少如此完美。这正是为什么你将在下一章中学习如何在 Combine 中处理错误的原因。



## 第 16 节：错误处理

你已经了解了很多关于如何编写 Combine 代码以随着时间的推移发出值的知识。不过，你可能已经注意到一件事：到目前为止，在你编写的大部分代码中，你根本没有处理错误，而主要处理的是“happy path”。

正如你在第 1 节“你好，Combine！”中所了解的，Combine 发布者声明了两个通用约束：Output，定义发布者发出的值的类型，以及 Failure，定义发布者可以完成的失败类型。

到目前为止，你已经将精力集中在发布者的输出类型上，但未能深入了解失败在发布者中的作用。这一节会改变这一点！



### 入门

在 projects/Starter.playground 中打开本章的 Playground。你将使用此 Playground 及其各个页面来试验 Combine 让你处理和操作错误的多种方式。

你现在已准备好深入研究 Combine 中的错误，但首先，请花点时间思考一下。错误是一个广泛的话题，你会从哪里开始呢？从没有错误开始。



### Never

失败类型为 Never 的发布者表示发布者永远不会失败。

虽然这乍一看可能有点奇怪，但它为这些发布者提供了一些极其强大的保证。具有永不失败类型的发布者可让你专注于使用发布者的值，同时绝对确保发布者永远不会失败，只有成功完成。

![image-20221019011943657](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221019011943657.png)

通过按 Command-1 在初学者 Playground 中打开 Project Navigator，然后选择 Never Playground 页面。将以下示例添加到其中：

```swift
example(of: "Never sink") {
  Just("Hello")
}
```

你创建了一个带有 Hello 字符串值的 Just。 Just 是从不失败的。 要确认这一点，请按住 Command 并单击 Just 初始化程序并选择 Jump to Definition，查看定义，你可以看到 Just 失败的类型别名：
公共类型别名失败 = 从不
Combine 对 Never 的无故障保证不仅仅是理论上的，而是深深植根于框架及其各种 API 中。

Combine 提供了几个操作符，这些操作符仅在保证发布者永远不会失败时才可用。 第一个是 sink 的变体，只处理值。

返回 Never Playground 页面并更新上面的示例，使其看起来像这样：

```swift
example(of: "Never sink") {
  Just("Hello")
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
}
```

运行你的 Playground，你会看到 Just 的值被打印出来：

```
——— Example of: Never sink ———
Hello
```

在上面的示例中，你使用 `sink(receiveValue:)`。 这种特定的 sink 重载使你可以忽略发布者的完成事件，而只处理其发出的值。

此重载仅适用于可靠的发布者。在错误处理方面，Combine 是智能且安全的，如果可能抛出错误，它会强制你处理完成事件——即对于非失败的发布者。

要看到这一点，你需要将永不失败的发布者变成可能失败的发布者。 有几种方法可以做到这一点，你将从最流行的一种开始 `setFailureType` 操作符。



**setFailureType**

将可靠的发布者转变为可靠的发布者的第一种方法是使用 `setFailureType`。这是另一个仅适用于失败类型为 Never 的发布者的操作符。

将以下代码和示例添加到你的 Playground 页面：

```swift
enum MyError: Error {
  case ohNo
}

example(of: "setFailureType") {
  Just("Hello")
}
```

首先定义示例范围之外的 `MyError` 错误类型。 稍后你将重用此错误类型。 然后，你通过创建一个与你之前使用的类似的 Just 来开始该示例。

现在，你可以使用 `setFailureType` 将发布者的失败类型更改为 `MyError`。在 Just 之后立即添加以下行：

```swift
.setFailureType(to: MyError.self)
```

要确认这实际上改变了发布者的失败类型，请开始输入 .eraseToAnyPublisher()，自动完成将显示已擦除的发布者类型：

![image-20221019013127326](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221019013127326.png)

在继续之前删除你开始输入的 .erase... 行。

现在是时候使用 sink 来消费发布者了。 在最后一次调用 setFailureType 后立即添加以下代码：

```swift
// 1
.sink(
  receiveCompletion: { completion in
    switch completion {
    // 2
    case .failure(.ohNo):
      print("Finished with Oh No!")
    case .finished:
      print("Finished successfully!")
    }
  },
  receiveValue: { value in
    print("Got value: \(value)")
  }
)
.store(in: &subscriptions)
```

你可能已经注意到关于上述代码的两个有趣的事实：

1. 它正在使用 `sink(receiveCompletion:receiveValue:)`。 `sink(receiveValue:)` 重载不再可用，因为此发布者可能会以失败事件完成。 结合迫使你处理此类发布者的完成事件。

2. 失败类型被严格键入为 `MyError`，这使你可以针对` .failure(.ohNo)` 情况而无需进行不必要的强制转换来处理该特定错误。

运行你的 Playground，你会看到以下输出：

```
——— Example of: setFailureType ———
Got value: Hello
Finished successfully!
```

当然，setFailureType 的作用只是类型系统定义。 由于原始发布者是 Just，因此实际上不会引发任何错误。

在本章后面，你将了解更多关于如何从你自己的发布者那里实际产生错误的信息。 但首先，还有一些专门针对永不失败的发布者的操作符。



**`assign(to:on:)`**

你在第 2 节“发布者和订阅者”中学到的 assign 操作符仅适用于不会失败的发布者，与 `setFailureType` 相同。 如果你仔细想想，这完全有道理。 向提供的 key path 发送错误会导致未处理的错误或未定义的行为。

添加以下示例进行测试：

```swift
example(of: "assign(to:on:)") {
  // 1
  class Person {
    let id = UUID()
    var name = "Unknown"
  }

  // 2
  let person = Person()
  print("1", person.name)

  Just("Shai")
    .handleEvents( // 3
      receiveCompletion: { _ in print("2", person.name) }
    )
    .assign(to: \.name, on: person) // 4
    .store(in: &subscriptions)
}
```

在上面的代码中：

1. 定义一个具有 id 和 name 属性的 Person 类。

2. 创建一个 Person 实例并立即打印其名称。

3. 一旦发布者发送完成事件，使用你之前了解的handleEvents 再次打印此人的 name。

4. 最后，使用assign 将人名设置为发布者发出的任何内容。

运行你的 Playground 并查看调试控制台：

```swift
——— Example of: assign(to:on:) ———
1 Unknown
2 Shai
```

正如预期的那样，只要 Just 发出它的值，assign 就会更新这个人的名字，这是有效的，因为 Just 不会失败。 相反，如果发布者有一个非从不失败的类型，你认为会发生什么？

在 Just("Shai") 正下方添加以下行：

```swift
.setFailureType(to: Error.self)
```

在此代码中，你已将失败类型设置为标准 Swift 错误。 这意味着它不再是 `Publisher<String, Never>`，而是现在的 `Publisher<String, Error>`。

尝试运行你的 Playground 。 对于手头的问题，Combine 非常冗长：

```
referencing instance method 'assign(to:on:)' on 'Publisher' requires the types 'Error' and 'Never' be equivalent
```

删除你刚刚添加的对 setFailureType 的调用，并确保你的 Playground 运行时没有编译错误。



**`assign(to:)`**

`assign(to:on:)` 有一个棘手的部分——它会 strong 地捕获提供给 on 参数的对象。

让我们探讨一下为什么这是有问题的。

在上一个示例之后立即添加以下代码：

```swift
example(of: "assign(to:)") {
  class MyViewModel: ObservableObject {
    // 1
    @Published var currentDate = Date()

    init() {
      Timer.publish(every: 1, on: .main, in: .common) // 2
        .autoconnect() 
        .prefix(3) // 3
        .assign(to: \.currentDate, on: self) // 4
        .store(in: &subscriptions)
    }

  }

  // 5
  let vm = MyViewModel()
  vm.$currentDate
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
}
```

这段代码有点长，让我们分解一下：

1. 在视图模型对象中定义一个@Published 属性。 它的初始值为当前日期。

2. 创建一个计时器发布者，它每秒发出当前日期。

3. 使用前缀操作符只接受 3 个日期更新。

4. 应用 `assign(to:on:)` 操作符将每个日期更新分配给你的 @Published 属性。

5. 实例化你的视图模型，sink 已发布的发布者，并打印出每个值。

如果你运行 Playground，你将看到类似于以下内容的输出：

```
——— Example of: assign(to:on:) strong capture ———
2021-08-21 12:43:32 +0000
2021-08-21 12:43:33 +0000
2021-08-21 12:43:34 +0000
2021-08-21 12:43:35 +0000
```

正如预期的那样，上面的代码打印分配给发布属性的初始日期，然后连续更新 3 次（受前缀操作符限制）。

看起来，一切都很好，那么这里到底出了什么问题呢？

对`assign(to:on:)` 的调用创建了一个 strongly retains self 的订阅。 本质上——self 挂在订阅上，而订阅挂在 self 上，创建了一个导致内存泄漏的保留周期。

![image-20221019015251780](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221019015251780.png)

幸运的是，Apple 的好人意识到这是有问题的，并引入了该操作符的另一个重载 - `assign(to:)`。

该操作符专门处理通过提供对其投影发布者的 inout 引用来将发布的值重新分配给 @Published 属性。

回到示例代码，找到以下两行：

```swift
.assign(to: \.currentDate, on: self) // 3
.store(in: &subscriptions)
```

并将它们替换为以下行：

```swift
.assign(to: &$currentDate)
```

使用 `assign(to:)` 操作符并将 inout 引用传递给预计的发布者会打破保留周期，让你轻松处理上述问题。

此外，它会在内部自动处理订阅的内存管理，这样你就可以省略 `store(in: &subscriptions)` 行。

> 注意：在继续之前，建议将前面的示例注释掉，这样打印出来的计时器事件就不会给控制台输出添加不必要的误导。


在这一点上，你几乎完成了可靠的出版商。但是在开始处理错误之前，你应该知道与可靠发布者相关的最后一个操作符：`assertNoFailure`。



**`assertNoFailure`**

当你想在开发过程中保护自己并确认发布者以成失败事件完成时，`assertNoFailure` 操作符非常有用。它不会阻止上游发出失败事件。但是，如果它检测到错误，它会因致命错误而崩溃，这给了你在开发中修复它的良好动力。

将以下示例添加到你的 playground：

```swift
example(of: "assertNoFailure") {
  // 1
  Just("Hello")
    .setFailureType(to: MyError.self)
    .assertNoFailure() // 2
    .sink(receiveValue: { print("Got value: \($0) ")}) // 3
    .store(in: &subscriptions)
}
```

在前面的代码中：

1. 使用 Just 创建一个可靠的发布者并将其失败类型设置为 MyError。

2. 如果发布者以失败事件完成，则使用 assertNoFailure 以致命错误崩溃。 这会将发布者的失败类型转回 Never。

3. 使用 sink 打印出任何接收到的值。 请注意，由于 assertNoFailure 将失败类型设置回 Never，因此 sink(receiveValue:) 重载再次由你使用。

运行你的 Playground，正如预期的那样，它应该可以正常工作：

```
——— Example of: assertNoFailure ———
Got value: Hello 
```

现在，在 setFailureType 之后，添加以下行：

```
.tryMap { _ in throw MyError.ohNo }
```

一旦 Hello 被推送到下游，你刚刚使用 tryMap 引发错误。 你将在本章后面了解更多关于以 try 为前缀的操作符。

再次运行你的 Playground 并查看控制台。 你将看到类似于以下内容的输出：

```
Playground execution failed:

error: Execution was interrupted, reason: EXC_BAD_INSTRUCTION (code=EXC_I386_INVOP, subcode=0x0).

...

frame #0: 0x00007fff232fbbf2 Combine`Combine.Publishers.AssertNoFailure...
```

由于发布者 failure ，playground 崩溃。 在某种程度上，你可以将 `assertFailure()` 视为代码的保护机制。 虽然你不应该在生产中使用它，但在开发过程中“早早崩溃并严重崩溃”非常有用。

在继续下一部分之前注释掉对 tryMap 的调用。



### 处理失败

哇，到目前为止，你已经在错误处理一章中学到了很多关于如何处理根本不会失败的发布者的知识！ :] 虽然有点讽刺，但我希望你现在能够理解彻底了解可靠发布者的特征和保证是多么重要。

考虑到这一点，是时候让你了解一下 Combine 提供的一些技术和工具，以应对实际失败的发布者。这包括内置发布者和你自己的发布者！

但首先，你实际上是如何产生失败事件的？如上一节所述，有几种方法可以做到这一点。你刚刚使用了 tryMap，那么为什么不进一步了解这些 try 操作符的工作原理呢？



#### try* 操作符

在“操作符”中，你了解了大部分 Combine 的操作符以及如何使用它们来操纵发布者发出的值和事件。你还学习了如何组合多个操作符的逻辑链来产生所需的输出。

在这些章节中，你了解到大多数操作符都有以 try 为前缀的并行操作符，我们现在将了解他们。

Combine 提供了一个有趣的区分可能引发错误和可能不会引发错误的操作符。

> 注意：Combine 中所有以 try 为前缀的操作符在遇到错误时的行为方式相同。你将只在本章中尝试使用 tryMap 操作符。


首先，从 Project navigator 中选择 try operator* playground 页面。向其中添加以下代码：

```swift
example(of: "tryMap") {
  // 1
  enum NameError: Error {
    case tooShort(String)
    case unknown
  }

  // 2
  let names = ["Marin", "Shai", "Florent"].publisher

  names
    // 3
    .map { value in
      return value.count
    }
    .sink(
      receiveCompletion: { print("Completed with \($0)") },
      receiveValue: { print("Got value: \($0)") }
    )
}
```

在上面的示例中：

1. 定义一个 NameError 错误枚举，你将立即使用它。

2. 创建发布三个不同字符串的发布者。

3. 将每个字符串映射到它的长度。

运行示例并查看控制台输出：

```
——— Example of: tryMap ———
Got value: 5
Got value: 4
Got value: 7
Completed with finished
```

正如预期的那样，所有名称都映射没有问题。 但是随后你收到了一个新的产品要求：如果你的代码接受的名称少于 5 个字符，则它应该引发错误。

将上面示例中的 map 替换为以下内容：

```swift
.map { value -> Int in
  // 1
  let length = value.count

  // 2
  guard length >= 5 else {
    throw NameError.tooShort(value)
  }

  // 3
  return value.count
}
```

在上面的映射中，你检查字符串的长度是否大于或等于 5。否则，你会尝试抛出适当的错误。

但是，只要你添加上述代码或尝试运行它，你就会看到编译器会产生错误：

```
Invalid conversion from throwing function of type '(_) throws -> _' to non-throwing function type '(String) -> _'
```

由于 map 是一个非抛出操作符，因此你不能从其中抛出错误。 幸运的是，try* 操作符就是为此目的而设计的。

用 tryMap 替换 map 并再次运行你的 Playground。 它现在将编译并产生以下输出（截断）：

```
——— Example of: tryMap ———
Got value: 5
Got value: 5
Completed with failure(...NameError.tooShort("Shai"))
```



#### 映射错误

map 和 tryMap 之间的区别不仅仅是后者允许抛出错误。 虽然 map 继承了现有的失败类型并且只操作发布者的值，但 tryMap 没有——它实际上将错误类型擦除为普通的 Swift 错误。 与带有 try 前缀的对应物相比，所有操作符都是如此。

切换到 Mapping errors playground 页面并在其中添加以下代码：

```swift
example(of: "map vs tryMap") {
  // 1
  enum NameError: Error {
    case tooShort(String)
    case unknown
  }

  // 2
  Just("Hello")
    .setFailureType(to: NameError.self) // 3
    .map { $0 + " World!" } // 4
    .sink(
      receiveCompletion: { completion in
        // 5
        switch completion {
        case .finished:
          print("Done!")
        case .failure(.tooShort(let name)):
          print("\(name) is too short!")
        case .failure(.unknown):
          print("An unknown name error occurred")
        }
      },
      receiveValue: { print("Got value \($0)") }
    )
    .store(in: &subscriptions)
}
```

在上面的示例中：

1. 定义一个用于此示例的 NameError。

2. 创建一个只发出字符串 Hello 的 Just。

3. 使用 setFailureType 设置失败类型为 NameError。

4. 使用 map 将另一个字符串附加到已发布的字符串。

5. 最后，使用 sink 的 receiveCompletion 为 NameError 的每个失败情况打印出适当的消息。

运行 Playground，你将看到以下输出：

```
——— Example of: map vs tryMap ———
Got value Hello World!
Done!
```

接下来，找到 switch completion { 行，Option-click 点击 completion：

![image-20221023004505006](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023004505006.png)

请注意，Completion 的失败类型是 NameError，这正是你想要的。 setFailureType 操作符允许你专门针对 NameError 故障，例如 failure(.tooShort(let name))。

接下来，将 map 更改为 tryMap。你会立即注意到操场不再编译。 Option-click 再次点击 completion：

![image-20221023004624718](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023004624718.png)

很有意思！ tryMap 删除了你的严格类型错误并将其替换为通用 Swift.Error 类型。即使你实际上并没有从 tryMap 中抛出错误，也会发生这种情况——你只是使用了它！这是为什么？

仔细想想，原因很简单：Swift 还不支持类型化 throws，尽管自 2015 年以来 Swift Evolution 中一直在讨论这个主题。这意味着当你使用带有 try 前缀的操作符时，你的错误类型将总是被抹去到最常见的祖先：Swift.Error。

所以你能对它做点啥？对于发布者来说，严格类型的失败的全部意义在于让你处理——在这个例子中——特别是 NameError，而不是任何其他类型的错误。

一种天真的方法是将通用错误手动转换为特定的错误类型，但这不是最理想的。它打破了严格类型错误的整个目的。幸运的是，Combine 为这个问题提供了一个很好的解决方案，称为 mapError。

在调用 tryMap 之后，立即添加以下行：

```swift
.mapError { $0 as? NameError ?? .unknown }
```

mapError 接收上游发布者抛出的任何错误，并让你将其映射到你想要的任何错误。在这种情况下，你可以利用它将错误转换回 NameError 或回退到 NameError.unknown 错误。在这种情况下，你必须提供一个回退错误，因为从理论上讲，强制转换可能会失败——即使它不会在这里——并且你必须从此操作符返回 NameError。

这会将 Failure 恢复为其原始类型，并将你的发布者转回 Publisher<String, NameError>。

构建并运行 Playground。它最终应该可以按预期编译和工作：

```
——— Example of: map vs tryMap ———
Got value Hello World!
Done!
```

最后，将对 tryMap 的整个调用替换为：

```swift
.tryMap { throw NameError.tooShort($0) }
```

此调用将立即从 tryMap 中引发错误。再次检查控制台输出，并确保你得到正确输入的 NameError：

```
——— Example of: map vs tryMap ———
Hello is too short!
```



#### 设计自己的可能会发生错误的 API

在构建你自己的基于 Combine 的代码和 API 时，你通常会使用来自其他来源的 API，这些 API 会返回因各种类型而失败的发布者。在创建自己的 API 时，你通常还希望围绕该 API 提供自己的错误。尝试这个比仅仅理论化更容易，所以你将继续深入研究一个例子！

在本节中，你将构建一个快速 API，让你可以从 https://icanhazdadjoke.com/api 上的 icanhazdadjoke API 获取一些有趣的笑话。

首先切换到 Designing your fallible APIs playground 页面，并向其中添加以下代码，这构成了下一个示例的第一部分：

```swift
example(of: "Joke API") {
  class DadJokes {
    // 1
    struct Joke: Codable {
      let id: String
      let joke: String
    }

    // 2
    func getJoke(id: String) -> AnyPublisher<Joke, Error> {
      let url = URL(string: "https://icanhazdadjoke.com/j/\(id)")!
      var request = URLRequest(url: url)
      request.allHTTPHeaderFields = ["Accept": "application/json"]
      
      // 3
      return URLSession.shared
        .dataTaskPublisher(for: request)
        .map(\.data)
        .decode(type: Joke.self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
    }

  }
}
```

在上面的代码中，你通过以下方式创建了新 DadJokes 类实例：

1. 定义一个 Joke 结构。 API 响应将被解码为 Joke 的一个实例。

2. 提供一个 getJoke(id:) 方法，该方法当前返回一个发布者，该发布者发出一个 Joke，并且可能会因标准 Swift.Error 而失败。

3. 使用 URLSession.dataTaskPublisher(for:) 调用 icanhazdadjoke API 并使用 JSONDecoder 和 decode 操作符将结果数据解码为 Joke。

最后，你需要实际使用你的新 API。 在 DadJokes 类下面直接添加以下内容，但仍在示例范围内：

```swift
// 4
let api = DadJokes()
let jokeID = "9prWnjyImyd"
let badJokeID = "123456"

// 5
api
  .getJoke(id: jokeID)
  .sink(receiveCompletion: { print($0) },
        receiveValue: { print("Got joke: \($0)") })
  .store(in: &subscriptions)
```

在此代码中：

4. 创建一个 DadJokes 的实例，并定义具有有效和无效笑话 ID 的两个常量。

5. 使用有效的笑话 ID 调用 DadJokes.getJoke(id:) 并打印任何完成事件或解码的笑话本身。

运行你的 Playground 并查看控制台：

```
——— Example of: Joke API ———
Got joke: Joke(id: "9prWnjyImyd", joke: "Why do bears have hairy coats? Fur protection.")
finished
```

所以你的 API 目前完美地处理了正常路径，但这是一个错误处理章节。 在包装其他发布者时，你需要问自己：“这个特定发布者会导致哪些错误？”

在这种情况下：

- 调用 dataTaskPublisher 可能会因各种原因失败并返回 URLError，例如连接错误或请求无效。

- 提供的笑话 ID 可能不存在。

- 如果 API 响应更改或其结构不正确，解码 JSON 响应可能会失败。

- 任何其他未知错误！ 错误很多而且是随机的，因此不可能考虑每个边缘情况。 出于这个原因，你总是希望有一个案例来涵盖未知或未处理的错误。

记住这个列表，在 DadJokes 类中添加以下代码，紧邻 Joke 结构体下方：

```swift
enum Error: Swift.Error, CustomStringConvertible {
 // 1
  case network
  case jokeDoesntExist(id: String)
  case parsing
  case unknown

  // 2
  var description: String {
    switch self {
    case .network:
      return "Request to API Server failed"
    case .parsing:
      return "Failed parsing response from server"
    case .jokeDoesntExist(let id):
      return "Joke with ID \(id) doesn't exist"
    case .unknown:
      return "An unknown error occurred"
    }
  }
}
```

此错误定义：

1. 概述 DadJokes API 中可能出现的所有错误。

2. 符合 CustomStringConvertible，让你可以为每个错误情况提供友好的描述。

添加上述错误类型后，你的 Playground 将不再编译。 这是因为 getJoke(id:) 返回一个 AnyPublisher<Joke, Error>。 之前，Error 指的是 Swift.Error，但现在它指的是 DadJokes.Error——在这种情况下，这实际上是你想要的。

那么，你怎么能把各种可能的和不同类型的错误都映射到你的 DadJoke.Error 中呢？ 如果你一直在关注本章，你可能已经猜到了答案：mapError 是你的朋友。

在对 decode 和 eraseToAnyPublisher() 的调用之间将以下内容添加到 getJoke(id:) 中：

```swift
.mapError { error -> DadJokes.Error in
  switch error {
  case is URLError:
    return .network
  case is DecodingError:
    return .parsing
  default:
    return .unknown
  }
}
```

这个简单的 mapError 使用 switch 语句将发布者可能抛出的任何类型的错误替换为 DadJokes.Error。 你可能会问自己：“我为什么要包装这些错误？” 这个问题的答案有两个：

1. 现在保证你的发布者只会因 DadJokes.Error 而失败，这在使用 API 和处理可能出现的错误时很有用。 你确切地知道你将从类型系统中得到什么。

2. 你不会泄露 API 的实现细节。 想一想，如果你使用 URLSession 执行网络请求并使用 JSONDecoder 解码响应，那么你的 API 的使用者是否关心？ 明显不是！ 消费者只关心你的 API 本身定义的错误——而不关心它的内部依赖。

还有一个你没有处理的错误：一个不存在的笑话 ID。 尝试替换以下行：

```swift
.getJoke(id: jokeID)
```

为：

```swift
.getJoke(id: badJokeID)
```

再次运行 Playground。 这一次，你将收到以下错误：

```
failure(Failed parsing response from server)
```

有趣的是，当你发送一个不存在的 ID 时，icanhazdadjoke 的 API 并不会因 HTTP 代码 404（未找到）而失败——正如大多数 API 所期望的那样。 相反，它会发回一个不同但有效的 JSON 响应：

```
{
    message = "Joke with id \"123456\" not found";
    status = 404;
}
```

处理这种情况需要一些技巧，但绝对不是你不能处理的！

回到 getJoke(id:)，用以下代码替换对 map(\.data) 的调用：

```swift
.tryMap { data, _ -> Data in
  // 6
  guard let obj = try? JSONSerialization.jsonObject(with: data),
        let dict = obj as? [String: Any],
        dict["status"] as? Int == 404 else {
    return data
  }

  // 7
  throw DadJokes.Error.jokeDoesntExist(id: id)
}
```

在上面的代码中，你使用 tryMap 在将原始数据传递给解码操作符之前执行额外的验证：

6. 你使用 JSONSerialization 来尝试检查状态字段是否存在且值为 404 — 即，笑话不存在。 如果不是这种情况，你只需返回数据，以便将其推送到下游的解码操作员。

7. 如果你确实找到了 404 状态码，你会抛出一个 .jokeDoesntExist(id:) 错误。

再次运行你的 Playground，你会发现另一个需要解决的小问题：

```
——— Example of: Joke API ———
failure(An unknown error occurred)
```

失败实际上被视为未知错误，而不是 DadJokes.Error，因为你没有在 mapError 中处理该类型。在你的 mapError 中，找到以下行：

```swift
return .unknown
```

替换为：

```swift
return error as? DadJokes.Error ?? .unknown
```

如果没有其他错误类型匹配，你尝试将其强制转换为 DadJokes.Error，然后放弃并退回到未知错误。

再次打开你的操场并查看控制台：

```
——— Example of: Joke API ———
failure(Joke with ID 123456 doesn't exist)
```

这一次，你收到正确的错误，类型正确！ 惊人的。 :]

在结束本示例之前，你可以在 getJoke(id:) 中进行最后一项优化。

你可能已经注意到，笑话 ID 由字母和数字组成。 在我们的“Bad ID”的情况下，你只发送了数字。 无需执行网络请求，你可以抢先验证你的 ID 并在不浪费资源的情况下失败。

在 getJoke(id:) 的开头添加以下最后一段代码：

```swift
guard id.rangeOfCharacter(from: .letters) != nil else {
  return Fail<Joke, Error>(
    error: .jokeDoesntExist(id: id)
  )
  .eraseToAnyPublisher()
}
```

在此代码中，你首先要确保 id 至少包含一个字母。如果不是这种情况，你会立即返回 Fail。

Fail 是一种特殊的发布者，它可以让你立即且强制地失败并显示提供的错误。它非常适合你希望根据某些条件提前失败的情况。最后，你使用 eraseToAnyPublisher 获得预期的 AnyPublisher<Joke, DadJokes.Error> 类型。

使用无效的 ID 再次运行你的示例，你将收到相同的错误消息。但是，它会立即发布并且不会执行网络请求。巨大的成功！

在继续之前，将你的调用恢复为 getJoke(id:) 以使用 jokeID 而不是 badJokeId。

此时，你可以通过手动“破坏”你的代码来验证你的错误逻辑。执行以下每个操作后，撤消你的更改，以便你可以尝试下一个操作：

1. 当你在上面创建 URL 时，在其中添加一个随机字母以破坏 URL。运行 Playground，你会看到：失败（对 API 服务器的请求失败）。

2. 注释掉以 request.allHttpHeaderFields 开头的行并运行 Playground。由于服务器响应将不再是 JSON，而只是纯文本，因此你将看到输出：失败（来自服务器的解析响应失败）。

像以前一样，向 getJoke(id:) 发送一个随机 ID。运行 Playground，你会得到：

```
failure(Joke with ID {your ID} doesn't exist).
```

就是这样！你刚刚构建了自己的基于 Combine 的生产级 API 层，其中包含自己的错误。



#### 捕获并重试

你学到了很多关于 Combine 代码错误处理的知识，但我们将最好的知识留到了最后，还有两个最终主题：捕获错误和重试失败的发布者。

Publisher 是一种表示工作的统一方式的好处在于，你拥有许多操作符，可以让你用很少的代码行完成大量工作。

继续并直接进入示例。

首先切换到 Project navigator 中的 Catching and retrying 页面。 展开 Playground 的 Sources 文件夹并打开 PhotoService.swift。

它包括一个带有 fetchPhoto(quality:failingTimes:) 方法的 PhotoService，你将在本节中使用该方法。 PhotoService 使用自定义发布者获取高质量或低质量的照片。 对于这个例子，要求一个高质量的图像总是会失败——所以你可以

尝试各种技术以重试并在发生故障时捕获故障。

返回到 Catching and retrying playground 页面，并将这个简单的示例添加到你的 Playground：

```swift
let photoService = PhotoService()

example(of: "Catching and retrying") {
  photoService
    .fetchPhoto(quality: .low)
    .sink(
      receiveCompletion: { print("\($0)") },
      receiveValue: { image in
        image
        print("Got image: \(image)")
      }
    )
    .store(in: &subscriptions)
}
```

上面的代码现在应该很熟悉了。 你实例化一个 PhotoService 并以 .low 质量调用 fetchPhoto。 然后使用 sink 打印出任何完成事件或获取的图像。

请注意， photoService 的实例化超出了示例的范围，因此它不会立即被释放。

运行你的 Playground 并等待它完成。 你应该看到以下输出：

```
——— Example of: Catching and retrying ———
Got image: <UIImage:0x600000790750 named(lq.jpg) {300, 300}>
finished
```

点击receiveValue 中第一行旁边的显示结果按钮，你会看到一张漂亮的低质量图片......好吧，一个组合。

![image-20221023015007259](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023015007259.png)

接下来，将质量从 .low 更改为 .high 并再次运行 Playground。 你将看到以下输出：

```
——— Example of: Catching and retrying ———
failure(Failed fetching image with high quality)
```

如前所述，要求高质量图像将失败。 这是你的起点！ 你可以在这里改进一些事情。 你将从重试失败开始。

很多时候，当你请求资源或执行某些计算时，失败可能是由于网络连接不良或其他资源不可用而导致的一次性事件。

在这些情况下，你通常会编写一个大型机制来重试不同的工作，同时跟踪尝试次数并决定如果所有尝试都失败了该怎么办。幸运的是，Combine 让这一切变得非常简单。

就像Combine 中的所有好东西一样，有一个操作符！

重试操作符接受一个数字。如果发布者失败，它将重新订阅上游并重试至你指定的次数。如果所有重试都失败，它只是将错误推送到下游，就像没有重试操作符一样。

是时候让你试试这个了。在 fetchPhoto(quality: .high) 行下方，添加以下行：

```swift
.retry(3)
```

等等，是这样吗？！是的。

对于包装在发布者中的每件工作，你都会获得一个免费的重试机制，就像调用这个简单的重试操作符一样简单。

在运行你的 Playground 之前，在 fetchPhoto 调用之间添加此代码并重试：

```swift
.handleEvents(
  receiveSubscription: { _ in print("Trying ...") },
  receiveCompletion: {
    guard case .failure(let error) = $0 else { return }
    print("Got error: \(error)")
  }
)
```

此代码将帮助你查看何时发生重试 - 它打印出 fetchPhoto 中发生的订阅和失败。

现在你准备好了！ 运行你的 Playground 并等待它完成。 你将看到以下输出：

```swift
——— Example of: Catching and retrying ———
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
failure(Failed fetching image with high quality)
```

如你所见，有四次尝试。 初始尝试，加上由重试操作符触发的三次重试。 由于获取高质量照片不断失败，因此操作员会耗尽所有重试尝试并将错误推送到 sink。

将以下调用替换为 fetchPhoto：

```swift
.fetchPhoto(quality: .high)
```

为：

```swift
.fetchPhoto(quality: .high, failingTimes: 2)
```

faliingTimes 参数将限制获取高质量图像失败的次数。 在这种情况下，它会在你调用它的前两次失败，然后成功。

再次运行你的 Playground，看看输出：

```
——— Example of: Catching and retrying ———
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got image: <UIImage:0x600001268360 named(hq.jpg) {1835, 2446}>
finished
```

如你所见，这次有 3 次尝试，最初的 1 次加两次重试。该方法前两次尝试失败，然后成功并返回这张华丽的、高质量的田间联合收割机照片：

![image-20221023015716082](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023015716082.png)

惊人的！但是，在此服务调用中，你还需要改进最后一项功能。如果获取高质量图像失败，你的产品人员要求你回退到低质量图像。如果获取低质量图像也失败，你应该回退到硬编码图像。

你将从两项任务中的后者开始。 Combine 包含一个名为 replaceError(with:) 的便捷操作符，如果发生错误，你可以使用该操作符回退到发布者类型的默认值。这也会将发布者的失败类型更改为从不，因为你将所有可能的失败都替换为后备值。

首先，从 fetchPhoto 中删除 failedTimes 参数，因此它会像以前一样不断失败。

然后在调用重试之后立即添加以下行：

```swift
.replaceError(with: UIImage(named: "na.jpg")!)
```

再次运行你的 Playground，看看这次的图像结果。在四次尝试之后——即最初加上三次重试——你回到磁盘上的硬编码图像：

![image-20221023015846417](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023015846417.png)

此外，查看控制台输出会显示你所期望的：有四次失败的尝试，然后是硬编码的后备图像：

```
——— Example of: Catching and retrying ———
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Got image: <UIImage:0x6000020e9200 named(na.jpg) {200, 200}>
finished
```

现在，对于本章的第二个任务和最后一部分：如果高质量图像失败，则回退到低质量图像。 Combine 为这项任务提供了完美的操作符，称为 catch。 它使你可以从发布者那里捕获故障并通过不同的发布者从中恢复。

要查看实际情况，请在重试之后、replaceError(with:) 之前添加以下代码：

```swift
.catch { error -> PhotoService.Publisher in
  print("Failed fetching high quality, falling back to low quality")
  return photoService.fetchPhoto(quality: .low)
}
```

最后一次运行你的 Playground 并查看控制台：

```
——— Example of: Catching and retrying ———
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Trying ...
Got error: Failed fetching image with high quality
Failed fetching high quality, falling back to low quality
Got image: <UIImage:0x60000205c480 named(lq.jpg) {300, 300}>
finished
```

和以前一样，初始尝试加上三次重试以获取高质量图像都失败了。 一旦操作员用尽了所有重试，catch 就会发挥作用并订阅 photoService.fetchPhoto，请求低质量的图像。 这导致从失败的高质量请求回退到成功的低质量请求。



### 关键点

- 失败类型为 Never 的发布者保证不会发出失败完成事件。

- 许多操作符只与可靠的发布者合作。例如：sink(receiveValue:)、setFailureType、assertNoFailure 和 assign(to_:on_:)。
- 以 try 为前缀的操作符允许你从其中抛出错误，而非 try 操作符则不能。
- 由于 Swift 不支持类型化的 throws，调用以 try 为前缀的操作符会将发布者的失败删除为普通的 Swift 错误。
- 使用 mapError 映射发布者的失败类型，并将发布者中的所有失败类型统一为单一类型。
- 当基于其他发布者使用自己的失败类型创建自己的 API 时，将所有可能的错误包装到自己的错误类型中以统一它们并隐藏 API 的实现细节。
- 你可以使用重试操作符重新订阅失败的发布者多次。
- 当你想为你的发布者提供一个默认的后备值时，replaceError(with:) 很有用，以防万一失败。
- 最后，你可以使用 catch 将失败的发布者替换为不同的后备发布者。



### 接下来去哪儿？

恭喜你读完本章。你基本上已经掌握了有关 Combine 中的错误处理的所有知识。

你只在本章的 try* 操作符部分中试验了 tryMap 操作符。你可以在 https://apple.co/3233VRB 上的 Apple 官方文档中找到带有 try 前缀的操作符的完整列表。

随着你对错误处理的掌握，是时候了解 Combine 中较低级别但最重要的主题之一：调度程序。继续下一章，了解什么是调度程序以及如何使用它们。



## 第 17 节：调度程序 Scheduler

在阅读本书的过程中，你已经阅读了有关将 Scheduler 作为参数的操作符。大多数情况下，你会简单地使用 DispatchQueue.main，因为它方便、易于理解并带来令人放心的安全感。

作为开发人员，你至少对 DispatchQueue 是什么有一个大致的了解。除了 DispatchQueue.main，你肯定已经使用了全局并发队列之一，或者创建了一个串行调度队列来串行运行操作。如果你不记得或不记得详细信息，请不要担心。你将在本章中重新评估有关调度队列的一些重要信息。

但是，为什么 Combine 需要一个新的类似概念呢？现在是你深入了解 Combine Scheduler 的真正性质、意义和目的的时候了！

在本章中，你将了解为什么会出现 Scheduler 的概念。你将探索 Combine 如何使异步事件和操作易于使用，当然，你将尝试使用 Combine 提供的所有 Scheduler。



### 调度器简介

根据 Apple 的文档，Scheduler 是一种定义何时以及如何执行闭包的协议。尽管定义是正确的，但这只是一部分。

调度程序提供上下文以尽快或在将来的某个日期执行未来的操作。该操作是协议本身中定义的闭包。但是术语闭包也可以隐藏发布者在特定调度程序上执行的某些值的传递。

你是否注意到此定义有意避免对线程的任何引用？这是因为具体的实现是定义调度程序协议提供的“上下文”在哪里执行的！

因此，你的代码将在哪个线程上执行的确切细节取决于你选择的 Scheduler。

记住这个重要的概念：Scheduler 不等于线程。你将在本章后面详细了解这对每个 Scheduler 意味着什么。

让我们从事件流的角度来看 Scheduler 的概念：

![image-20221023150356813](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023150356813.png)

你在上图中看到的内容：

- 在主 (UI) 线程上发生用户操作（按钮按下）。

- 它会触发一些工作在后台调度程序上进行处理。

- 要显示的最终数据在主线程上传递给订阅者，因此订阅者可以更新应用程序的 UI。

你可以看到 Scheduler 的概念如何深深植根于前台/后台执行的概念。此外，根据你选择的实现，工作可以串行化或并行化。

因此，要全面了 Scheduler，需要查看哪些类符合 Scheduler 协议。

但首先，你需要了解与Scheduler 相关的两个重要操作符！

> 注意：在下一节中，你将主要使用符合 Combine 的 Scheduler 协议的 DispatchQueue。



### Scheduler 操作符

Combine 框架提供了两个基本的操作符来使用调度器：

- subscribe(on:) 和 subscribe(on:options:) 在指定的 Scheduler 上创建订阅（开始工作）。
- receive(on:) 和 receive(on:options:) 在指定的 Scheduler 上传递值。

此外，以下操作符将调度程序和调度程序选项作为参数。你在第 6 节了它们：

- debounce(for:scheduler:options:)

- delay(for:tolerance:scheduler:options:)

- measureInterval(using:options:)

- throttle(for:scheduler:latest:)

- timeout(_:scheduler:options:customError:)

如果你需要刷新对这些操作符的记忆，请立即回顾第 6 节。



**介绍 subscribe(on:)**

请记住——在你订阅它之前，发布者是一个无生命的实体。但是当你订阅发布者时会发生什么？有几个步骤：

![image-20221023150832513](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023150832513.png)

1. 发布者 receive 订阅者并创建 Subscription。
2. 订阅者 receive Subscription 并从发布者请求值（虚线）。
3. 发布者开始工作（通过 Subscription）。
4. 发布者发出值（通过 Subscription）。
5. 操作符转换值。
6. 订阅者收到最终值。

当你的代码订阅发布者时，步骤一、二和三通常发生在当前线程上。 但是当你使用 subscribe(on:) 操作符时，所有这些操作都在你指定的 Scheduler 上运行。

> 注意：在查看 receive(on:) 操作符时，你将回到此图。 然后，你将了解底部的两个框，其中标有步骤五和六。


你可能希望发布者在后台执行一些昂贵的计算以避免阻塞主线程。 执行此操作的简单方法是使用 subscribe(on:)。

是时候看一个例子了！



打开项目文件夹中的 Starter.playground 并选择 subscribeOn-receiveOn 页面。确保显示 Debug 区域，然后添加以下代码：

```swift
// 1
let computationPublisher = Publishers.ExpensiveComputation(duration: 3)

// 2
let queue = DispatchQueue(label: "serial queue")

// 3
let currentThread = Thread.current.number
print("Start computation publisher on thread \(currentThread)")
```

以下是上述代码的细分：

1. 这个 Playground 在 Sources/Computation.swift 中定义了一个名为 ExpensiveComputation 的特殊发布者，它模拟一个长时间运行的计算，在指定的持续时间后发出一个字符串。
2.  一个串行队列，你将使用它来触发特定调度程序上的计算。正如你在上面了解到的，DispatchQueue 符合调度程序协议。
3. 你获取当前执行线程号。在 Playground 中，主线程（线程编号 1）是你的代码运行的默认线程。Thread 类的编号扩展在 Sources/Thread.swift 中定义。


注意：ExpensiveComputation 发布者如何实现的细节目前并不重要。你将在下一节中了解有关创建自己的发布者的更多信息。


回到 subscribeOn-receiveOn Playground 页面，你需要订阅 computePublisher 并显示它发出的值：

```swift
let subscription = computationPublisher
  .sink { value in
    let thread = Thread.current.number
    print("Received computation result on thread \(thread): '\(value)'")
  }
```

执行 Playground 并查看输出：

```
Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 1
Beginning expensive computation on thread 1
Completed expensive computation on thread 1
Received computation result on thread 1 'Computation complete'
```

让我们深入研究各个步骤以了解会发生什么：

- 你的代码在主线程上运行。 从那里，它订阅计算发布者。

- ExpensiveComputation 发布者接收订阅者。

- 它创建一个订阅，然后开始工作。

- 工作完成后，发布者通过订阅传递结果并完成。

你可以看到所有这些都发生在主线程线程 1 上。

现在，更改发布者订阅以插入 subscribe(on:) 调用：

```swift
let subscription = computationPublisher
  .subscribe(on: queue)
  .sink { value in...
```

再次执行 Playground 可以看到类似如下的输出：

```
Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 5
Beginning expensive computation from thread 5
Completed expensive computation on thread 5
Received computation result on thread 5 'Computation complete'
```

啊!这是不同的！现在你可以看到你仍在从主线程订阅，但是将委托 Combine 到你提供的队列以有效地执行订阅。队列在其线程之一上运行代码。由于计算在线程 5 上开始并完成，然后从该线程发出结果值，因此你的接收器也接收该线程上的值。

> 注意：由于 DispatchQueue 的动态线程管理特性，你可能会在此日志中看到不同的线程号，并在本章中看到更多日志。重要的是一致性：相同的线程号应该在相同的步骤中显示。


但是，如果你想更新一些屏幕信息怎么办？你需要在接收器闭包中执行类似 DispatchQueue.main.async { ... } 的操作，以确保你正在从主线程执行 UI 更新。

有一种更有效的方法可以使用 Combine！



**介绍 receive(on:)**

你想知道的第二个重要的操作符是 receive(on:)。它允许你指定应该使用哪个调度程序向订阅者传递值。但是，这是什么意思？

在订阅中的接收器之前插入一个调用 receive(on:) ：

```swift
let subscription = computationPublisher
  .subscribe(on: queue)
  .receive(on: DispatchQueue.main)
  .sink { value in
```

然后，再次执行 Playground。现在你看到这个输出：

```
Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 4
Beginning expensive computation from thread 4
Completed expensive computation on thread 4
Received computation result on thread 1 'Computation complete'
```

> 注意：你可能会在与接下来的两个步骤不同的线程上看到第二条消息。由于 Combine 中的内部管道，这一步和下一步可能在同一个队列上异步执行。由于 Dispatch 动态管理自己的线程池，因此你可能会看到这一行和下一行的线程号不同，但不会看到线程 1。

成功！即使计算工作正常并从后台线程发出结果，你现在也可以保证始终在主队列上接收值。这是你安全地执行 UI 更新所需要的。

在本调度操作符介绍中，你使用了 DispatchQueue。 Combine 扩展了它来实现调度器协议，但它不是唯一的！是时候深入了解 Schedule 了！



### Scheduler 实现

Apple 提供了几种调度器协议的具体实现：

- ImmediateScheduler：一个简单的 Scheduler，它立即在当前线程上执行代码，这是默认的执行上下文，除非使用 subscribe(on:)、receive(on:) 或任何其他将调度程序作为参数的操作符进行修改。
- RunLoop：绑定到 Foundation 的 Thread 对象。
- DispatchQueue：可以是串行的或并发的。
- OperationQueue：规范工作项执行的队列。

在本章的其余部分，你将了解所有这些及其具体细节。

> 注意：这里一个明显的遗漏是缺少 TestScheduler，它是任何响应式编程框架的测试部分不可或缺的一部分。如果没有这样一个虚拟的、模拟的 Scheduler，彻底测试你的 Conbine 代码是一项挑战。你将在第 19 节“测试”中探索有关这种特殊调度程序的更多细节。



#### ImmediateScheduler

调度程序类别中最简单的条目也是 Combine 框架提供的最简单的：ImmediateScheduler。这个名字已经破坏了细节，所以看看它的作用！

打开 Playground 的 ImmediateScheduler 页面。你不需要此调试区域，但请确保你使实时视图可见。你将使用这个 Playground 中内置的一些精美的新工具来跨调度程序跟踪你的发布者值！

开始创建一个简单的计时器，就像你在前几章中所做的那样：

```swift
let source = Timer
  .publish(every: 1.0, on: .main, in: .common)
  .autoconnect()
  .scan(0) { counter, _ in counter + 1 }
```

接下来，准备一个创建发布者的闭包。你将使用 Sources/Record.swift 中定义的自定义操作符：recordThread(using:)。该算子记录了算子看到一个值通过时当前的线程，并且可以多次记录从发布者源到最终接收器。

> 注意：此 recordThread(using:) 操作符仅用于测试目的，因为该操作符将数据类型更改为内部值类型。它的实现细节超出了本章的范围，但喜欢冒险的读者可能会发现它很有趣。


添加此代码：

```swift
// 1
let setupPublisher = { recorder in
  source
    // 2
    .recordThread(using: recorder)
    // 3
    .receive(on: ImmediateScheduler.shared)
    // 4
    .recordThread(using: recorder)
    // 5
    .eraseToAnyPublisher()
}

// 6
let view = ThreadRecorderView(title: "Using ImmediateScheduler", setup: setupPublisher)
PlaygroundPage.current.liveView = UIHostingController(rootView: view)
```

在上面的代码中：

1. 准备一个返回发布者的闭包，使用给定的记录器对象通过 recordThread(using:) 设置当前线程记录。
2. 在这个阶段，定时器发出一个值，所以你记录当前线程。你已经猜到是哪一个了吗？
3. 确保发布者在共享的 ImmediateScheduler 上传递值。
4. 记录你现在所在的线程。
5. 闭包必须返回一个 AnyPublisher 类型。这主要是为了方便内部实现。
6. 准备并实例化一个 ThreadRecorderView，它显示发布的值在各个记录点的线程之间的迁移。

执行 Playground 页面，几秒钟后查看输出：

![image-20221023153712833](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023153712833.png)

此表示显示源发布者（计时器）发出的每个值。在每一行上，你都可以看到值正在经历的线程。每次添加 recordThread(using:) 操作符时，都会在该行上看到一个额外的线程号。

在这里你可以看到在你添加的两个记录点处，当前线程是主线程。这是因为 ImmediateScheduler 立即在当前线程上“调度”。

此表示显示源发布者（计时器）发出的每个值。在每一行上，你都可以看到值正在经历的线程。每次添加 recordThread(using:) 操作符时，都会在该行上看到一个额外的线程号。

在这里你可以看到在你添加的两个记录点处，当前线程是主线程。这是因为 ImmediateScheduler 立即在当前线程上“调度”。

为了验证这一点，你可以做一个小实验！回到你的 setupPublisher 闭包定义，就在第一条 recordThread 行之前，插入以下内容：

```swift
.receive(on: DispatchQueue.global())
```

这请求源发出的值在全局并发队列上进一步可用。这会产生有趣的结果吗？执行 Playground：

![image-20221023153915913](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023153915913.png)

这是完全不同的！你能猜到为什么线程一直在变化吗？你将在本章的 DispatchQueue 介绍中了解更多信息！



**ImmediateScheduler 选项**

由于大多数操作符在其参数中接受调度程序，你还可以找到一个接受 SchedulerOptions 值的选项参数。在 ImmediateScheduler 的情况下，此类型被定义为 Never，因此在使用 ImmediateScheduler 时，你永远不应该为操作符的 options 参数传递值。

**ImmediateScheduler 的陷阱**

关于 ImmediateScheduler 的一件事是它是即时的。你将无法使用 Scheduler 协议的任何 schedule(after:) 变体，因为你需要指定延迟的 SchedulerTimeType 没有公共初始化程序，并且对于立即调度毫无意义。

你将在本节中学习的第二种调度程序存在类似但不同的陷阱：RunLoop。



#### RunLoop scheduler

早期 iOS 和 macOS 开发人员熟悉 RunLoop。早于 DispatchQueue，它是一种在线程级别管理输入源的方法，包括在主 (UI) 线程中。你的应用程序的主线程仍然有一个关联的 RunLoop。你还可以通过从当前线程调用 RunLoop.current 为任何基础线程获取一个。

> 注意：现在 RunLoop 是一个不太有用的类，因为 DispatchQueue 在大多数情况下是一个明智的选择。这就是说，仍有一些特定情况下 RunLoop 很有用。例如，Timer 将自己安排在 RunLoop 上。 UIKit 和 AppKit 依赖 RunLoop 及其执行模式来处理各种用户输入情况。描述有关 RunLoop 的所有内容超出了本书的范围。


要查看 RunLoop，请在 Playground 中打开 RunLoop 页面。你之前使用的 Timer 源是相同的，所以已经为你编写好了。在其后添加此代码：

```swift
let setupPublisher = { recorder in
  source
    // 1
    .receive(on: DispatchQueue.global())
    .recordThread(using: recorder)
    // 2
    .receive(on: RunLoop.current)
    .recordThread(using: recorder)
    .eraseToAnyPublisher()
}

let view = ThreadRecorderView(title: "Using RunLoop", setup: setupPublisher)
PlaygroundPage.current.liveView = UIHostingController(rootView: view)
```

1. 和之前一样，首先让值通过全局并发队列。为什么？因为很好玩！
2. 然后，你要求在 RunLoop.current 上接收值。

但是什么是 RunLoop.current？它是与调用时当前的线程相关联的 RunLoop。闭包由 ThreadRecorderView 调用，从主线程设置发布者和订阅者。因此，RunLoop.current 就是主线程的 RunLoop。

执行 Playground 看看会发生什么：

![image-20221023154756596](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023154756596.png)

正如你所要求的，第一个 recordThread 显示每个值都通过一个全局并发队列的线程，然后在主线程上继续。

**一点理解挑战**

如果你第一次使用 subscribe(on: DispatchQueue.global()) 而不是 receive(on:) 会发生什么？试试看！

你会看到所有内容都记录在线程一上。一开始可能并不明显，但完全合乎逻辑。是的，发布者已在并发队列上订阅。但是请记住，你正在使用一个 Timer，它在主 RunLoop 上发出其值！因此，无论你选择在哪个 Scheduler 上订阅此发布者，值都将始终在主线程上开始它们的旅程。

**使用 RunLoop 调度代码执行**

Scheduler 允许你安排尽快执行的代码，或在未来某个日期之后执行。虽然不可能将后一种形式与 ImmediateScheduler 一起使用，但 RunLoop 完全能够延迟执行。

每个 Scheduler 实现都定义了自己的 SchedulerTimeType。在你弄清楚要使用的数据类型之前，这会让事情变得有点复杂。在 RunLoop 的情况下，SchedulerTimeType 值是一个 Date。

你将安排一个操作，在几秒钟后取消 ThreadRecorderView 的订阅。事实证明 ThreadRecorder 类有一个可选的 Cancellable 可用于停止其对发布者的订阅。

首先，你需要一个变量来保存对 ThreadRecorder 的引用。在页面的开头，添加以下行：

```swift
var threadRecorder: ThreadRecorder? = nil
```

现在你需要捕获线程记录器实例。执行此操作的最佳位置是 setupPublisher 闭包。但是怎么做？你可以：

- 向闭包添加显式类型，分配 threadRecorder 变量并返回发布者。你需要添加显式类型，因为糟糕的 Swift 编译器会抱怨“可能无法推断出复杂的闭包返回类型。
- 在订阅时使用一些操作符来捕获记录器。

去狂野做后者！

在你的 setupPublisher 闭包中添加此行，然后在 eraseToAnyPublisher() 之前：

```swift
.handleEvents(receiveSubscription: { _ in threadRecorder = recorder })
```

捕获记录器的有趣选择！

> 注意：你已经在第 10 节“调试”中了解了 handleEvents。它有一个很长的签名，让你可以在发布者生命周期的不同点执行代码（在反应式编程术语中，这称为注入副作用），而无需实际与其发出的值进行交互。在这种情况下，你正在拦截记录器订阅发布者的时刻，以便在你的全局变量中捕获记录器。不漂亮，但它以一种有趣的方式完成了这项工作！

现在你已准备就绪，可以在几秒钟后安排一些操作。在页面末尾添加此代码：

```swift
RunLoop.current.schedule(
  after: .init(Date(timeIntervalSinceNow: 4.5)),
  tolerance: .milliseconds(500)) {
    threadRecorder?.subscription?.cancel()
  }
```

这个 schedule(after:tolerance:) 让你可以安排提供的闭包何时执行以及可容忍的漂移，以防系统无法在所选时间精确执行代码。你将当前日期添加 4.5 秒，以允许在执行之前发送四个值。

运行 Playground。你可以看到列表在第四项之后停止更新。这是你的取消机制有效！

> 注意：如果你只获得三个值，这可能意味着你的 Mac 运行速度有点慢并且无法容纳半秒的容差，因此你可以尝试更多地调整日期，例如，将 timeIntervaleSinceNow 设置为 5.0，将容差设置为 1.0。



**RunLoop 选项**

与 ImmediateScheduler 一样，RunLoop 不为采用 SchedulerOptions 参数的调用提供任何合适的选项。

**RunLoop 陷阱**

RunLoop 的使用应仅限于主线程的 RunLoop，以及你在需要时控制的 Foundation 线程中可用的 RunLoop。也就是说，任何你自己使用 Thread 对象开始的东西。

要避免的一个特殊陷阱是在 DispatchQueue 上执行的代码中使用 RunLoop.current。这是因为 DispatchQueue 线程可能是短暂的，这使得它们几乎不可能依赖 RunLoop。

你现在已经准备好学习最通用和最有用的调度程序：DispatchQueue！



#### DispatchQueue Scheduler

在本章和之前的章节中，你一直在各种情况下使用 DispatchQueue。毫不奇怪，DispatchQueue 符合 Scheduler 协议，并且完全可用于所有将 Scheduler 作为参数的操作符。

但首先，快速回顾一下调度队列。 Dispatch 框架是 Foundation 的一个强大组件，它允许你通过向系统管理的调度队列提交工作来在多核硬件上同时执行代码。

DispatchQueue 可以是串行的（默认）或并发的。串行队列按顺序执行你提供给它的所有工作项。并发队列将并行启动多个工作项，以最大限度地提高 CPU 使用率。两种队列类型都有不同的用法：

- 串行队列通常用于保证某些操作不重叠。因此，如果所有操作都发生在同一个队列中，他们可以使用共享资源而无需锁定。
- 并发队列将同时执行尽可能多的操作。因此，它更适合纯计算。



**队列和线程**

你一直使用的最熟悉的队列是 DispatchQueue.main。它直接映射到主（UI）线程，在这个队列上执行的所有操作都可以自由地更新用户界面。 UI 更新只允许从主线程进行。

所有其他队列，无论是串行的还是并发的，都在系统管理的线程池中执行它们的代码。这意味着你永远不应该对队列中运行的代码中的当前线程做出任何假设。特别是，你不应使用 RunLoop.current 来安排工作，因为 DispatchQueue 管理其线程的方式。

所有调度队列共享同一个线程池。你提供工作执行的串行队列将使用该池中的任何可用线程。一个直接的结果是，来自同一队列的两个连续工作项可能使用不同的线程，同时仍按顺序执行。

这是一个重要的区别：当使用 subscribe(on:)、receive(on:) 或任何其他采用 Scheduler 参数的操作符时，你永远不应假设支持调度程序的线程每次都是相同的。



**使用 DispatchQueue 作为 Scheduler**

是你试验的时候了！像往常一样，你将使用计时器来发出值并观察它们在调度程序之间的迁移。但是这一次，你将使用 Dispatch Queue 计时器创建计时器。

打开名为 DispatchQueue 的游乐场页面。首先，你将创建几个队列以供使用。将此代码添加到你的 Playground：

```swift
let serialQueue = DispatchQueue(label: "Serial queue")
let sourceQueue = DispatchQueue.main
```

你将使用 sourceQueue 发布来自计时器的值，然后使用 serialQueue 来试验切换调度程序。

现在添加以下代码：

```swift
// 1
let source = PassthroughSubject<Void, Never>()

// 2
let subscription = sourceQueue.schedule(after: sourceQueue.now,
                                        interval: .seconds(1)) {
  source.send()
}
```

1. 当计时器触发时，你将使用 Subject 发出一个值。你不关心实际的输出类型，因此你只需使用 Void。
2. 正如你在第 11 节“定时器”中所了解的，队列完全能够生成定时器，但没有用于队列定时器的 Publisher API。 你必须使用调度程序协议中的 schedule() 方法的重复变体。它立即开始并返回一个 Cancellable。每次计时器触发时，你都会通过 source 发送一个 Void 值。

> 注意：你是否注意到你是如何使用 now 属性来指定计时器的开始时间的？这是调度程序协议的一部分，并返回使用调度程序的 SchedulerTimeType 表示的当前时间。每个实现调度器协议的类都为此定义了自己的类型。

现在，你可以开始练习 scheduler 了。通过添加以下代码来设置你的发布者：

```swift
let setupPublisher = { recorder in
  source
    .recordThread(using: recorder)
    .receive(on: serialQueue)
    .recordThread(using: recorder)
    .eraseToAnyPublisher()
}
```


这里没有什么新东西，你在本章中已经多次编写了类似的模式。

然后，与前面的示例一样，设置显示：

```swift
let view = ThreadRecorderView(title: "Using DispatchQueue",
                              setup: setupPublisher)
PlaygroundPage.current.liveView = UIHostingController(rootView: view)
```

执行 Playground。很容易，你会看到它的意图：

![image-20221023161035542](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023161035542.png)

1. 计时器在主队列上触发并通过 Subject 发送 Void 值。

2. 发布者在你的串行队列上接收值。

你有没有注意到第二个 recordThread(using:) 在 receive(on:) 操作符之后是如何记录当前线程中的变化的？这是 DispatchQueue 如何不保证每个工作项在哪个线程上执行的一个完美示例。在 receive(on:) 的情况下，工作项是从当前调度程序跳到另一个调度程序的值。

现在，如果你从串行队列中发出值并保持相同的  receive(on:)  操作符会发生什么？值还会在旅途中改变线程吗？

试试看！回到代码的开头，将 sourceQueue 定义更改为：

```swift
let sourceQueue = serialQueue
```

现在，再次执行 Playground：

![image-20221023161244842](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023161244842.png)



有趣的！你再次看到 DispatchQueue 的无线程保证效果，但你也看到 receive(on:) 操作符从不切换线程！看起来内部正在进行一些优化以避免额外的切换。你将在本章的挑战中探索这一点！

**DispatchQueue 选项**

DispatchQueue 是唯一提供一组选项的 Scheduler，当操作符采用 SchedulerOptions 参数时，你可以传递这些选项。这些选项主要围绕指定 QoS（服务质量）值独立于 DispatchQueue 上已设置的值。工作项还有一些额外的标志，但在绝大多数情况下你都不需要它们。

不过，要查看如何指定 QoS，请将 setupPublisher 中的 receive(on:options:) 修改为以下内容：

```swift
.receive(
  on: serialQueue,
  options: DispatchQueue.SchedulerOptions(qos: .userInteractive)
)
```

你将 DispatchQueue.SchedulerOptions 的实例传递给指定最高服务质量的选项：.userInteractive。它指示操作系统尽最大努力将价值交付优先于不太重要的任务。当你想尽快更新用户界面时，可以使用此功能。相反，如果快速交付的压力较小，你可以使用 .background 服务质量。在此示例的上下文中，你不会看到真正的区别，因为它是唯一正在运行的任务。

在实际应用程序中使用这些选项有助于操作系统决定在你同时有许多队列忙碌的情况下首先安排哪个任务。它确实可以微调你的应用程序性能！

你几乎完成了调度程序！再坚持一点。你需要了解最后一个 Scheduler。



#### OperationQueue

你将在本章中学习的最后一个调度程序是 OperationQueue。文档将其描述为一个规范操作执行的队列。它是一种丰富的监管机制，可让你创建具有依赖关系的高级操作。但是在Combine 的上下文中，你将不会使用这些机制。

由于 OperationQueue 在后台使用 Dispatch，因此在使用另一种时表面上几乎没有区别。或者有吗？

举一个简单的例子。打开 OperationQueue Playground 页面并开始编码：

```
let queue = OperationQueue()

let subscription = (1...10).publisher
  .receive(on: queue)
  .sink { value in
    print("Received \(value)")
  }
```

你正在创建一个简单的发布者发出 1 到 10 之间的数字，确保值到达你创建的 OperationQueue。然后在接收器中打印该值。

你能猜到会发生什么吗？展开 Debug 区域并执行 Playground：

```
Received 4
Received 3
Received 2
Received 7
Received 5
Received 10
Received 6
Received 9
Received 1
Received 8
```

这令人费解！按顺序发出但无序到达！怎么会这样？要找出答案，你可以更改打印行以显示当前线程号：

```swift
print("Received \(value) on thread \(Thread.current.number)")
```

再次执行 Playground：

```
Received 1 on thread 5
Received 2 on thread 4
Received 4 on thread 7
Received 7 on thread 8
Received 6 on thread 9
Received 10 on thread 10
Received 5 on thread 11
Received 9 on thread 12
Received 3 on thread 13
Received 8 on thread 14
```

啊哈！如你所见，每个值都是在不同的线程上接收的！如果你查看有关 OperationQueue 的文档，有一条关于线程的说明，其中说 OperationQueue 使用 Dispatch 框架（因此是 DispatchQueue）来执行操作。这意味着它不保证它会为每个交付的值使用相同的底层线程。

此外，每个 OperationQueue 中都有一个参数可以解释一切：它是 maxConcurrentOperationCount。它默认为系统定义的数字，允许操作队列同时执行大量操作。由于你的发布者几乎在同一时间发出所有项目，它们被 Dispatch 的并发队列分派到多个线程！

对你的代码进行一些修改。定义队列后，添加以下行：

```swift
queue.maxConcurrentOperationCount = 1
```

然后运行页面，查看调试区域：

```
Received 1 on thread 3
Received 2 on thread 3
Received 3 on thread 3
Received 4 on thread 3
Received 5 on thread 4
Received 6 on thread 3
Received 7 on thread 3
Received 8 on thread 3
Received 9 on thread 3
Received 10 on thread 3
```

这一次，你将获得真正的顺序执行——将 maxConcurrentOperationCount 设置为 1 相当于使用串行队列——并且你的值按顺序到达。



**OperationQueue 选项**

OperationQueue 没有可用的 SchedulerOptions。它实际上是别名为 RunLoop.SchedulerOptions 的类型，它本身没有提供任何选项。

**OperationQueue 陷阱**

你刚刚看到 OperationQueue 默认并发执行操作。你需要非常清楚这一点，因为它可能会给你带来麻烦：默认情况下，OperationQueue 的行为类似于并发 DispatchQueue。

但是，当你每次发布者发出值时都有大量工作要执行时，它可能是一个很好的工具。你可以通过调整 maxConcurrentOperationCount 参数来控制负载。



### 挑战

#### 挑战 1：停止计时器

在本章关于 DispatchQueue 的部分中，你创建了一个可取消的计时器来为你的源发布者提供值。

设计两种不同的方法在 4 秒后停止计时器。提示：你需要使用 DispatchQueue.SchedulerTimeType.advanced(by:)。

找到解决方案了吗？将它们与 projects/challenge/challenge1/ final playground 中的进行比较：

1. 使用串行队列的调度程序协议 schedule(after:_:) 方法来安排取消订阅的闭包的执行。

2. 使用 serialQueue 的普通 asyncAfter(_:_:) 方法（pre-Combine）来做同样的事情。



#### 挑战 2：发现优化

在本章前面，你读到了一个有趣的问题：当你在连续的 receive(on:) 调用中使用相同的调度程序时，Combine 是在优化，还是 Dispatch 框架优化？

要找出答案，你需要转到挑战 2。你的挑战是设计一种方法来回答这个问题。这不是很复杂，但也不是微不足道的。

在 Dispatch 框架中，DispatchQueue 的初始化程序采用可选的目标参数。它使你可以指定要在其上执行代码的队列。换句话说，你创建的队列只是一个影子，而执行代码的真正队列是目标队列。

因此，尝试猜测是 Combine 还是 Dispatch 正在执行优化的想法是使用两个不同的队列，其中一个以另一个为目标。所以在 Dispatch 框架级别，代码都在同一个队列上执行，但是（希望）Combine 不会注意到。

因此，如果你执行此操作并看到在同一线程上接收到所有值，则很可能 Dispatch 正在为你执行优化。你为解决方案编写代码所采取的步骤是：

1. 创建第二个串行队列，以第一个为目标。

2. 为第二个串行队列添加 .receive(on:) 以及 .recordThread 步骤。

完整的解决方案可在 projects/challenge/challenge2 final playground 中找到。



### 关键点

- Scheduler 定义了一项工作的执行上下文。
- Apple 的操作系统提供了丰富多样的工具来帮助你安排代码执行。
- 将这些 Scheduler 与 Scheduler 协议相结合，以帮助你在任何给定情况下选择最适合工作的 Scheduler。
- 每次使用 receive(on:) 时，发布者中的其他操作符都会在指定的调度程序上执行。也就是说，除非他们自己采用 Scheduler 参数！



### 接下来去哪儿？

你学到了很多，你的大脑一定会被所有这些信息融化！下一章会涉及更多内容，因为它会教你创建自己的发布者和处理背压。确保你现在安排一个当之无愧的休息时间，并为下一章恢复精神！



## 第 18 章：自定义发布者和处理背压

在你学习 Combine 的过程中，你可能会觉得框架中缺少很多操作符。如果你有其他反应式框架的经验，这可能尤其如此，这些反应式框架通常提供丰富的操作符生态系统，包括内置的和第三方的。Combine 允许你创建自己的发布者。本节将向你展示如何操作。

你将在本章中学习的第二个相关主题是背压(Backpressure)管理。你将了解什么是背压以及如何创建处理它的发布者。



### 创建自己的发布者

创建自己的发布者的复杂性从“简单”到“相当复杂”不等。对于你实现的每个操作符，你将寻求最简单的实现形式来实现你的目标。在本节中，你将了解创建自己的发布者的三种不同方法：

- 在 Publisher 命名空间中使用简单的扩展方法。

- 在 Publishers 命名空间中使用产生值的 Subscription 实现一个类型。

- 与上面相同，但订阅会转换来自上游发布者的值。

> 注意：从技术上讲，可以在没有自定义 Subscription 的情况下创建自定义发布者。如果你这样做，你将失去应对订阅者 demand 的能力，这使你的发布者在 Combine 生态系统中是非法的。提前取消也可能成为一个问题。这不是推荐的方法，本节将教你如何以正确的方式编写发布者。



### 发布者作为扩展方法

你的首要任务是通过重用现有的操作符来实现一个简单的操作符。

为此，你将添加一个新的 unwrap() 操作符，它解开可选值并忽略它们的 nil 值。这将是一个非常简单的练习，因为你可以重用现有的 compactMap(_:) 操作符，它就是这样做的，尽管它需要你提供一个闭包。

使用新的 unwrap() 操作符将使你的代码更易于阅读，并且会使你正在做的事情变得非常清晰。读者甚至不必查看闭包的内容。

你将在 Publisher 命名空间中添加你的操作员，就像你添加所有其他操作符一样。

打开本章的 Playground，可以在 projects/Starter.playground 中找到，然后从 Project Navigator 打开其 Unwrap 操作页面。

然后，添加以下代码：

```swift
extension Publisher {
  // 1
  func unwrap<T>() -> Publishers.CompactMap<Self, T> where Output == Optional<T> {
    // 2
    compactMap { $0 }
  }
}
```

1. 将自定义操作符编写为方法，最复杂的部分是签名。

2. 实现很简单：只需在 self 上使用 compactMap(_:)！

方法签名的制作可能令人难以置信。分解它来看看它是如何工作的：

```swift
func unwrap<T>()
```

你的第一步是使操作符通用，因为它的输出是上游发布者的可选类型包装的类型。

```swift
-> Publishers.CompactMap<Self, T>
```

该实现使用单个 compactMap(_:)，因此返回类型由此派生。如果你查看 Publishers.CompactMap，你会发现它是一个泛型类型：public struct CompactMap<Upstream, Output>。在实现自定义操作符时，Upstream 是 Self（你正在扩展的发布者），Output 是包装类型。

```swift
where Output == Optional<T> {
```

最后，你将操作符限制为 Optional 类型。你可以方便地编写它以将包装的类型 T 与你的方法的泛型类型匹配......等等！

> 注意：当开发更复杂的操作符作为方法时，例如使用操作符链时，签名会很快变得非常复杂。一个好的技巧是让你的操作符返回一个 AnyPublisher<OutputType, FailureType>。在该方法中，你将返回一个以 eraseToAnyPublisher() 结尾的发布者，以对签名进行类型擦除。



**测试你的自定义操作符**

现在你可以测试你的新操作符了。在扩展名下方添加此代码：

```swift
let values: [Int?] = [1, 2, nil, 3, nil, 4]

values.publisher
  .unwrap()
  .sink {
    print("Received value: \($0)")
  }
```

运行 Playground，正如预期的那样，只有非 nil 值会打印到调试控制台：

```
Received value: 1
Received value: 2
Received value: 3
Received value: 4
```

现在你已经了解了如何制作简单的操作符方法，是时候深入研究更丰富、更复杂的发布者了。你可以像这样对发布者进行分组：

- 充当“生产者”并直接自己生产价值的操作符。

- 发布者充当“转换器”，转换上游发布者产生的价值。

在本节，你将学习如何使用这两种方法，但你首先需要了解订阅发布者时发生的事情的详细信息。



### 订阅机制

Subscription 是 Combine 的无名英雄：虽然你到处都能看到发布者，但它们大多是无生命的实体。当你订阅发布者时，它会实例化一个订阅，该订阅负责接收订阅者的需求并产生事件（例如，值和完成）。

以下是订阅生命周期的详细信息：

![image-20221023203025770](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221023203025770.png)

1. 订阅者订阅发布者。
2. 发布者创建一个订阅，然后将其交给订阅者（调用 receive(subscription:)）。
3. 订阅者通过向订阅发送所需数量的值（调用订阅的 request(_:) 方法）从订阅中请求值。
4. Subscription 开始工作并开始发出值。它将它们一一发送给订阅者（调用订阅者的 receive(_:) 方法）。
5. 收到值后，订阅者返回一个新的 Subscribers.Demand，它会添加到先前的总需求中。
6. Subscription 会一直发送值，直到发送的值数量达到请求的总数。

如果 Subscription 发送的值与订阅者请求的值一样多，它应该在发送更多值之前等待新的请求请求。你可以绕过此机制并继续发送值，但这会破坏订阅者和订阅之间的合同，并可能导致基于 Apple 定义的发布者树中出现未定义的行为。

最后，如果出现错误或订阅的值源完成，订阅会调用订阅者的 receive(completion:) 方法。



### 发布者发出值

在第 11 节“定时器”中，你了解了 Timer.publish()，但发现将调度队列用于定时器有点不方便。为什么不基于 Dispatch 的 DispatchSourceTimer 开发自己的计时器呢？

你将这样做，同时检查订阅机制的详细信息。

要开始使用，请打开 Playground 的 DispatchTimer 发布者页面。

你将首先定义一个配置结构，这将使订阅者及其订阅之间共享计时器配置变得容易。将此代码添加到 Playground：

```swift
struct DispatchTimerConfiguration {
  // 1
  let queue: DispatchQueue?
  // 2
  let interval: DispatchTimeInterval
  // 3
  let leeway: DispatchTimeInterval
  // 4
  let times: Subscribers.Demand
}
```

如果你曾经使用过 DispatchSourceTimer，那么其中一些属性对你来说应该很熟悉：

1. 你希望你的计时器能够在某个队列上触发，但如果你不在乎，使队列成为可选的。在这种情况下，计时器将在其选择的队列上触发。
2. 计时器触发的时间间隔，从订阅时间开始。
3. 系统可以延迟传递计时器事件的截止日期之后的最大时间量。
4. 你想要接收的计时器事件数。由于你正在制作自己的计时器，因此请使其灵活并能够在完成之前交付有限数量的事件！



#### 添加 DispatchTimer 发布者

你现在可以开始创建 DispatchTimer 发布者。 这将很简单，因为所有工作都发生在 Subscription 内！

在你的配置下方添加此代码：

```swift
extension Publishers {
  struct DispatchTimer: Publisher {
    // 5
    typealias Output = DispatchTime
    typealias Failure = Never
    // 6
    let configuration: DispatchTimerConfiguration
    

    init(configuration: DispatchTimerConfiguration) {
      self.configuration = configuration
    }

  }
}
```

5. 你的计时器将当前时间作为 DispatchTime 值发出。 当然，它永远不会失败，所以发布者的 Failure 类型是 Never。

6. 保留给定配置的副本。 你现在不使用它，但是当你收到订阅者时会需要它。

> 注意：你将在编写代码时开始看到编译器错误。 请放心，在你完成实施要求时，你将解决这些问题。


现在，通过将此代码添加到 DispatchTimer 定义中，在你的初始化程序下方，实现 Publisher 协议所需的 receive(subscriber:) 方法：

```swift
// 7
func receive<S: Subscriber>(subscriber: S)
  where Failure == S.Failure,
        Output == S.Input {
  // 8
  let subscription = DispatchTimerSubscription(
    subscriber: subscriber,
    configuration: configuration
  )
  // 9
  subscriber.receive(subscription: subscription)
}
```

7. 功能是通用的；它需要一个编译时特化来匹配订阅者类型。

8. 大部分动作将发生在你将在短时间内定义的 DispatchTimerSubscription 中。
9. 正如你在第 2 节“发布者和订阅者”中所了解的，订阅者会收到一个订阅，然后它可以向该订阅发送值请求。

这就是发布者的全部内容！真正的工作将在 Subscription 内部进行。



#### 建立你的 Subscription

Subscription 的作用是：

- 接受订阅者的初始 Demend。

- 按需生成定时器事件。

- 每次订阅者收到一个值并返回一个需求时，都添加到需求计数。
- 确保它不会提供比配置中要求的更多的值。

这可能听起来像很多代码，但它并不复杂！

开始在 Publishers 的扩展下方定义订阅：

```swift
private final class DispatchTimerSubscription
  <S: Subscriber>: Subscription where S.Input == DispatchTime {
}
```

签名本身提供了很多信息：

- 此 Subscription 在外部不可见，只能通过订阅协议，因此你将其设为私有。

- 这是一个类，因为你想通过引用传递它。 然后订阅者可以将它添加到 Cancellable 集合中，但也可以保留它并独立调用 cancel()。

- 它用于输入值类型为 DispatchTime 的订阅者，这是订阅发出的内容。



#### 将所需属性添加到你的 Subscription

现在将这些属性添加到订阅类的定义中：

```swift
// 10
let configuration: DispatchTimerConfiguration
// 11
var times: Subscribers.Demand
// 12
var requested: Subscribers.Demand = .none
// 13
var source: DispatchSourceTimer? = nil
// 14
var subscriber: S?
```

此代码包含：

10. 订阅者的配置。
11. 计时器将触发的最大次数，你从配置中复制。你将使用它作为每次发送值时递减的计数器。
12. 当前 Demend；例如，订阅者请求的值的数量 - 每次发送值时都会减少它。
13. 将生成定时器事件的内部 DispatchSourceTimer。
14. 订阅者。这清楚地表明，只要订阅没有完成、失败或取消，Subscription 就有责任保留订阅者。

> 注意：最后一点对于理解 Combine 中的所有权机制至关重要。 Subscription 是订阅者和发布者之间的链接。它可以让订阅者——例如，一个持有闭包的对象，如 AnySubscriber 或 sink——在必要时保持存在。这就解释了为什么，如果你不保留订阅，你的订阅者似乎永远不会收到值：一旦订阅被解除分配，一切都会停止。内部实现当然可能会根据你正在编码的发布者的具体情况而有所不同。



#### 初始化和取消你的 Subscription

现在，在 DispatchTimerSubscription 定义中添加一个初始化器：

```swift
init(subscriber: S,
     configuration: DispatchTimerConfiguration) {
  self.configuration = configuration
  self.subscriber = subscriber
  self.times = configuration.times
}
```

这很简单。初始化程序将时间设置为发布者应接收计时器事件的最大次数，如配置指定的那样。每次发布者发出事件时，此计数器都会递减。当它达到零时，计时器以完成的事件结束。

现在，实现 cancel()，Subscription 必须提供的必需方法：

```swift
func cancel() {
  source = nil
  subscriber = nil
}
```

将 DispatchSourceTimer 设置为 nil 足以阻止它运行。将订阅者属性设置为 nil 会将其从订阅的范围中释放出来。不要忘记在你自己的订阅中执行此操作，以确保你不会在内存中保留不再需要的对象。

你现在可以开始编写 Subscription 的核心代码：request(_:)。



#### 让你的 Subscription 请求值

你还记得你在第 2 节“发布者和订阅者”中学到的内容吗？一旦订阅者通过订阅发布者获得订阅，它必须从订阅中请求值。

这就是所有魔法发生的地方。要实现它，请将此方法添加到类中，在取消方法上方：

```swift
// 15
func request(_ demand: Subscribers.Demand) {
  // 16
  guard times > .none else {
    // 17
    subscriber?.receive(completion: .finished)
    return
  }
}
```

15. 这个必需的方法接收来自订阅者的请求。
16. Demand 是累积的：它们加起来形成订阅者请求的值的总数。验证你是否已经向订阅者发送了足够的值，如配置中指定的那样。也就是说，如果你发送了最大数量的预期值，则与发布者收到的要求无关。
17. 如果是这种情况，你可以通知订阅者发布者已完成发送值。

在 guard 语句之后添加以下代码来继续实现此方法：

```swift
// 18
requested += demand

// 19
if source == nil, requested > .none {

}
```

18. 通过添加新需求来增加请求值的总数。

19. 检查定时器是否已经存在。如果没有，并且请求的值存在，那么是时候启动它了。



#### 配置你的计时器

将此代码添加到最后一个的 if：

```swift
// 20
let source = DispatchSource.makeTimerSource(queue: configuration.queue)
// 21
source.schedule(deadline: .now() + configuration.interval,
                repeating: configuration.interval,
                leeway: configuration.leeway)
```

20. 从你配置的队列中创建 DispatchSourceTimer。
21. 安排计时器在每 configuration.interval 秒后触发。

一旦计时器启动，你将永远不会停止它，即使你不使用它向订阅者发出事件。它会一直运行，直到订阅者取消订阅——或者你取消分配订阅。

你现在已准备好编写计时器的核心，它向订阅者发出事件。仍然在 if 正文中，添加以下代码：

```swift
// 22
source.setEventHandler { [weak self] in
  // 23
  guard let self = self,
        self.requested > .none else { return }

  // 24
  self.requested -= .max(1)
  self.times -= .max(1)
  // 25
  _ = self.subscriber?.receive(.now())
  // 26
  if self.times == .none {
    self.subscriber?.receive(completion: .finished)
  }
}
```

22. 为你的计时器设置事件处理程序。 这是计时器每次触发时调用的简单闭包。 确保保持对 self 的弱引用，否则订阅将永远不会解除分配。
23. 验证当前是否有请求的值——发布者可以在没有当前需求的情况下暂停，正如你将在本章后面了解背压时看到的那样。
24. 减少两个计数器，因为你要发出一个值。
25. 向订阅者发送一个值。
26. 如果要发送的值的总数达到配置指定的最大值，你可以认为发布者已完成并发出完成事件！



#### 激活你的计时器

现在你已经配置了源计时器，存储对它的引用并通过在 setEventHandler 之后添加以下代码来激活它：

```swift
self.source = source
source.activate()
```

那是很多步骤，并且很容易在此过程中不经意地放错一些代码。这段代码应该已经清除了操场上的所有错误。如果没有，你可以通过查看上述步骤或将你的代码与项目/Final.playground 中的 Playground 完成版本进行比较来仔细检查你的工作。

最后一步：在 DispatchTimerSubscription 的整个定义之后添加此扩展，以定义一个操作符，以便轻松链接此发布者：

```swift
extension Publishers {
  static func timer(queue: DispatchQueue? = nil,
                    interval: DispatchTimeInterval,
                    leeway: DispatchTimeInterval = .nanoseconds(0),
                    times: Subscribers.Demand = .unlimited)
                    -> Publishers.DispatchTimer {
    return Publishers.DispatchTimer(
      configuration: .init(queue: queue,
                           interval: interval,
                           leeway: leeway,
                           times: times)
                      )
  }
}
```



#### 测试你的计时器

你现在可以测试你的新计时器了！

新计时器操作符的大多数参数，除了间隔，都有一个默认值，以便在常见用例中更容易使用。这些默认值创建了一个永不停止的计时器，具有最小的回旋余地，并且不指定要在哪个队列上发出值。

在扩展之后添加此代码以测试你的计时器：

```swift
// 27
var logger = TimeLogger(sinceOrigin: true)
// 28
let publisher = Publishers.timer(interval: .seconds(1),
                                 times: .max(6))
// 29
let subscription = publisher.sink { time in
  print("Timer emits: \(time)", to: &logger)
}
```

27. 这个 Playground 定义了一个类 TimeLogger，它与你在第 10 节“调试”中学习创建的类非常相似。唯一的区别是这个可以显示两个连续值之间的时间差，或者自创建计时器以来经过的时间。在这里，你要显示自开始记录以来的时间。
28. 你的计时器发布者将准确触发六次，每秒一次。
29. 记录你通过 TimeLogger 收到的每个值。

运行 Playground，你会看到这个漂亮的输出——或类似的东西，因为时间会略有不同：

```
+1.02668s: Timer emits: DispatchTime(rawValue: 183177446790083)
+2.02508s: Timer emits: DispatchTime(rawValue: 183178445856469)
+3.02603s: Timer emits: DispatchTime(rawValue: 183179446800230)
+4.02509s: Timer emits: DispatchTime(rawValue: 183180445857620)
+5.02613s: Timer emits: DispatchTime(rawValue: 183181446885030)
+6.02617s: Timer emits: DispatchTime(rawValue: 183182446908654)
```

设置时会有轻微的偏移——而且 Playgrounds 也可能会增加一些延迟——然后计时器每秒触发一次，六次。

你还可以测试取消计时器，例如，几秒钟后。添加此代码以执行此操作：

```swift
DispatchQueue.main.asyncAfter(deadline: .now() + 3.5) {
  subscription.cancel()
}
```

再次运行 Playground。这一次，你只看到三个值。看起来你的计时器工作得很好！

尽管它在 Combine API 中几乎看不到，但正如你刚刚发现的那样，Subscription 完成了大部分工作。

享受你的成功。接下来你将进行另一次深潜！



### 发布者改变值

你现在可以开发自己的操作符，甚至是相当复杂的操作符。接下来要学习的是如何创建 Subscription 来转换来自上游发布者的值。这是完全控制发布者、Subscription 的关键。

在第 9 节“网络”中，你了解了共享 Subscription 有多么有用。当底层发布者执行重要工作时，例如从网络请求数据，你希望与多个订阅者共享结果。但是，你希望避免多次发出相同的请求来检索相同的数据。

如果你不需要再次执行工作，那么将结果重播给未来的订阅者也是有益的。

为什么不尝试实现 shareReplay()，它可以完全满足你的需求？这将是一项有趣的任务！要编写此操作符，你将创建一个执行以下操作的发布者：

- 在第一个订阅者时订阅上游发布者。

- 将最后 N 个值重播给每个新订阅者。

- 中继完成事件，如果事先发出了一个。

请注意，实施起来绝非易事，但你肯定已经掌握了！你将逐步完成，最后，你将拥有一个 shareReplay()，你可以在未来的 Combine 驱动项目中使用它。

在 Playground 中打开 ShareReplay 操作员页面以开始使用。



#### 实现 ShareReplay 操作符

要实现 shareReplay()，你需要：

1. 符合 Subscription 协议的类型。这是每个订阅者将收到的 Subscription。为确保你能够应对每个订阅者的 demand 和取消，每个订阅者都将收到单独的 Subscription。

2. 符合 Publisher 协议的类型。你将把它实现为一个类，因为所有订阅者都希望共享同一个实例。

首先添加此代码以创建你的 Subscription 类：

```swift
// 1
fileprivate final class ShareReplaySubscription<Output, Failure: Error>: Subscription {
  // 2
  let capacity: Int
  // 3
  var subscriber: AnySubscriber<Output,Failure>? = nil
  // 4
  var demand: Subscribers.Demand = .none
  // 5
  var buffer: [Output]
  // 6
  var completion: Subscribers.Completion<Failure>? = nil
}
```

从一开始：

1. 你使用通用类而不是结构来实现订阅：发布者和订阅者都需要访问和改变订阅。
2. 重播缓冲区的最大容量将是你在初始化期间设置的常数。
3. 在订阅期间保留对订阅者的引用。使用类型擦除的 AnySubscriber 可以使你免于与类型系统作斗争。 :]
4. 跟踪发布者从订阅者那里收到的累积 demand，以便你可以准确地交付请求数量的值。
5. 将挂起的值存储在缓冲区中，直到它们被传递给订阅者或被丢弃。
6. 这样可以保留潜在的完成事件，以便在新订阅者开始请求值时立即将其交付给他们。

> 注意：如果你觉得没有必要保留完成事件，而你将立即交付它，请放心，情况并非如此。订阅者应该首先接收它的订阅，然后在它准备好接受值时立即接收一个完成事件——如果之前发出过一个。它发出的第一个 request(_:) 发出信号。发布者不知道这个请求什么时候发生，所以它只是将完成交给订阅，以便在正确的时间交付它。



#### 初始化你的订阅

接下来，将初始化程序添加到订阅定义中：

```swift
init<S>(subscriber: S,
        replay: [Output],
        capacity: Int,
        completion: Subscribers.Completion<Failure>?)
        where S: Subscriber,
              Failure == S.Failure,
              Output == S.Input {
  // 7
  self.subscriber = AnySubscriber(subscriber)
  // 8
  self.buffer = replay
  self.capacity = capacity
  self.completion = completion
}
```

此初始化程序从上游发布者接收多个值并将它们设置在此订阅实例上。具体来说，它：

7. 存储订阅者的类型擦除版本。
8. 存储上游发布者的当前缓冲区、最大容量和完成事件（如果已发出）。



#### 向订阅者发送完成事件和未完成的值

你需要一种将完成事件中继给订阅者的方法。将以下内容添加到订阅类以满足该需求：

```swift
private func complete(with completion: Subscribers.Completion<Failure>) {
  // 9
  guard let subscriber = subscriber else { return }
  self.subscriber = nil
  // 10
  self.completion = nil
  self.buffer.removeAll()
  // 11
  subscriber.receive(completion: completion)
}
```

此私有方法执行以下操作：

9. 在方法执行期间保留订阅者，但在类中将其设置为 nil。 这种防御措施确保用户在完成时可能错误地发出的任何呼叫都将被忽略。
10. 通过将完成设置为 nil 来确保只发送一次完成，然后清空缓冲区。
11. 将完成事件中继给订阅者。

你还需要一种可以向订阅者发出值的方法。 添加此方法以根据需要发出值：

```swift
private func emitAsNeeded() {
  guard let subscriber = subscriber else { return }
  // 12
  while self.demand > .none && !buffer.isEmpty {
    // 13
    self.demand -= .max(1)
    // 14
    let nextDemand = subscriber.receive(buffer.removeFirst())
    // 15
    if nextDemand != .none {
      self.demand += nextDemand
    }
  }
  // 16
  if let completion = completion {
    complete(with: completion)
  }
}
```

首先，此方法确保有订阅者。 如果有，该方法将：

12. 仅当缓冲区中有一些值并且有未完成的 Demand 时才发出值。
13. 将未完成的 Demand 减一。
14. 向订阅者发送第一个值，并收到新的 Demand。
15. 将新 Demand 添加到未完成的总 Demand 中，但前提是它不是 .none。 否则，你将遇到崩溃，因为 Combine 不会将 Subscribers.Demand.none 视为零，并且添加或减去 .none 将触发异常。
16. 如果完成事件未决，请立即发送。

事情正在形成！ 现在，实现 Subscription 最重要的要求：

```swift
func request(_ demand: Subscribers.Demand) {
  if demand != .none {
    self.demand += demand
  }
  emitAsNeeded()
}
```

那是一件容易的事。 请记住检查 .none 以避免崩溃 - 并密切关注未来版本的 Combine 修复此问题 - 然后继续发射。

> 注意：即使需求是 .none ，调用 emitAsNeeded() 也能保证你正确地传递已经发生的完成事件。



#### 取消订阅

取消订阅更加容易。 添加此代码：

```swift
func cancel() {
  complete(with: .finished)
}
```

与订阅者一样，你需要实现接受值的方法和完成事件。 首先添加此方法以接受值：

```swift
func receive(_ input: Output) {
  guard subscriber != nil else { return }
  // 17
  buffer.append(input)
  if buffer.count > capacity {
    // 18
    buffer.removeFirst()
  }
  // 19
  emitAsNeeded()
}
```

确保有订阅者后，此方法将：

17. 将值添加到未完成的缓冲区。你可以针对最常见的情况进行优化，例如无限需求，但现在这将完美地完成工作。
18. 确保缓冲的值不要超过请求的容量。你以先进先出的方式处理此问题——当一个已经满的缓冲区接收每个新值时，当前的第一个值将被删除。
19. 将结果交付给订阅者。



#### 结束你的订阅

现在，添加以下方法来接受完成事件，你的订阅类将完成：

```swift
func receive(completion: Subscribers.Completion<Failure>) {
  guard let subscriber = subscriber else { return }
  self.subscriber = nil
  self.buffer.removeAll()
  subscriber.receive(completion: completion)
}
```

此方法删除订阅者，清空缓冲区——因为这只是良好的内存管理——并将完成发送到下游。

你已完成订阅！这不是很有趣吗？现在，是时候为发布者编写代码了。



#### 编码你的发布者

Publishers 通常是 Publishers 命名空间中的值类型（结构）。有时将发布者实现为类如 Publishers.Multicast 或 Publishers.Share 是有意义的。对于这个发布者，你需要一个类，类似于 share()。不过，这是规则的例外，因为大多数情况下你会使用结构。

首先添加此代码以在订阅后定义你的发布者类：

```swift
extension Publishers {
  // 20
  final class ShareReplay<Upstream: Publisher>: Publisher {
    // 21
    typealias Output = Upstream.Output
    typealias Failure = Upstream.Failure
  }
}
```


你希望多个订阅者能够共享此操作符的单个实例，因此你使用类而不是结构。它也是通用的，上游发布者的最终类型作为参数。

这个新的发布者不会改变上游发布者的输出或失败类型——它只是使用上游的类型。



#### 添加发布者所需的属性

现在，将你的发布者需要的属性添加到 ShareReplay 的定义中：

```swift
// 22
private let lock = NSRecursiveLock()
// 23
private let upstream: Upstream
// 24
private let capacity: Int
// 25
private var replay = [Output]()
// 26
private var subscriptions = [ShareReplaySubscription<Output, Failure>]()
// 27
private var completion: Subscribers.Completion<Failure>? = nil
```

这段代码的作用：

22. 因为你将同时提供多个订阅者，所以你需要一个锁来保证对可变变量的独占访问。
23. 保留对上游发布者的引用。你将在订阅生命周期的各个阶段需要它。
24. 你可以在初始化期间指定重放缓冲区的最大记录容量。
25. 当然，你还需要存储你记录的值。
26. 你提供多个订阅者，因此你需要将它们留在身边以通知他们事件。每个订阅者都从一个专用的 ShareReplaySubscription 获取其值——你将在短时间内编写此代码。
27. 操作符即使在完成后也可以重播值，因此你需要记住上游发布者是否完成。

看起来，还有一些代码要写！最后，你会发现它并没有那么多，但还有一些工作要做，例如使用适当的锁定，以便你的操作员在所有条件下都能顺利运行。



#### 初始化值并将其转发给你的发布者

首先，将必要的初始化程序添加到你的 ShareReplay 发布者：

```swift
init(upstream: Upstream, capacity: Int) {
  self.upstream = upstream
  self.capacity = capacity
}
```

这里没什么特别的，只是存储上游发布者和容量。接下来，你将添加几个方法来帮助将代码拆分为更小的块。

添加将来自上游的传入值中继到订阅者的方法：

```swift
private func relay(_ value: Output) {
  // 28
  lock.lock()
  defer { lock.unlock() }

  // 29
  guard completion == nil else { return }

  // 30
  replay.append(value)
  if replay.count > capacity {
    replay.removeFirst()
  }
  // 31
  subscriptions.forEach {
    $0.receive(value)
  }
}
```

此代码执行以下操作：

28. 由于多个订阅者共享此发布者，因此你必须使用锁保护对可变变量的访问。在这里使用 defer 并不是绝对必要的，但这是一种很好的做法，以防你以后修改方法，添加提前返回语句并忘记解锁你的锁。
29. 仅在上游尚未完成时才中继值。
30. 将值添加到滚动缓冲区并仅保留最新的容量值。这些是重播给新订阅者的内容。
31. 将缓冲的值中继到每个连接的订阅者。



#### 完成后通知你的发布者

其次，添加这个方法来处理完成事件：

```swift
private func complete(_ completion: Subscribers.Completion<Failure>) {
  lock.lock()
  defer { lock.unlock() }
  // 32
  self.completion = completion
  // 33
  subscriptions.forEach {
    $0.receive(completion: completion)
  }
}
```

使用此代码：

32. 为未来的订阅者保存完成事件。

33. 将其转发给每个连接的订阅者。

你现在已准备好开始编写每个发布者必须实现的接收方法。此方法将接收订阅者。它的职责是创建一个新订阅，然后将其交给订阅者。

添加此代码以开始定义此方法：

```swift
func receive<S: Subscriber>(subscriber: S)
  where Failure == S.Failure,
        Output == S.Input {
  lock.lock()
  defer { lock.unlock() }
}
```

receive(subscriber:) 的这个标准原型指定订阅者，无论它是什么，都必须具有与发布者的输出和失败类型匹配的输入和失败类型。 还记得第 2 章“发布者和订阅者”中的内容吗？



#### 创建你的订阅

接下来，将此代码添加到创建订阅并将其交给订阅者的方法中：

```swift
// 34
let subscription = ShareReplaySubscription(
  subscriber: subscriber,
  replay: replay,
  capacity: capacity,
  completion: completion)

// 35
subscriptions.append(subscription)
// 36
subscriber.receive(subscription: subscription)
```

34. 新 Subscription 引用订阅者并接收当前重放缓冲区、容量和任何未完成的完成事件。
35. 你保留订阅以将未来的事件传递给它。
36. 你将订阅发送给订阅者，订阅者可能（现在或以后）开始请求值。



#### 订阅上游发布者并处理其输入

你现在已准备好订阅上游发布者。 你只需要做一次：当你收到第一个订阅者时。

将此代码添加到 receive(subscriber:) - 请注意，你故意不包括结束 } 因为还有更多代码要添加：

```swift
// 37
guard subscriptions.count == 1 else { return }

let sink = AnySubscriber(
  // 38
  receiveSubscription: { subscription in
    subscription.request(.unlimited)
  },
  // 39
  receiveValue: { [weak self] (value: Output) -> Subscribers.Demand in
    self?.relay(value)
    return .none
  },
  // 40
  receiveCompletion: { [weak self] in
    self?.complete($0)
  }
)
```

使用此代码你：

37. 只向上游发布者订阅一次。
38. 使用方便的 AnySubscriber 类，它接受闭包，并在订阅时立即请求 .unlimited 值以让发布者运行完成。
39. 将你收到的值转发给下游订阅者。
40. 使用你从上游获得的完成事件来完成你的发布者。

> 注意：你最初可以请求 .max(self.capacity) 并仅接收它，但请记住，Combine 是 Demand 驱动的！如果你请求的值没有发布者能够产生的那么多值，那么你可能永远不会收到完成事件！


为避免保留循环，你只保留对 self 的弱引用。

你快完成了！现在，你需要做的就是将 AnySubscriber 订阅到上游发布者。

通过添加以下代码完成此方法的定义：

```swift
upstream.subscribe(sink)
```

再一次，Playground 上的所有错误现在都应该清楚了。请记住，你可以通过将其与项目/最终版中的 Playground 完成版本进行比较来仔细检查你的工作。



#### 添加便利操作符

你的发布者已完成！当然，你还想要一件事：一个便利的操作符，帮助将这个新发布者与其他发布者联系起来。

将其作为扩展添加到 Playground 末尾的 Publishers 命名空间：

```swift
extension Publisher {
  func shareReplay(capacity: Int = .max)
    -> Publishers.ShareReplay<Self> {
    return Publishers.ShareReplay(upstream: self,
                                  capacity: capacity)
  }
}
```

你现在拥有一个功能齐全的 shareReplay(capacity:) 操作符。这是很多代码，现在是时候尝试一下了！



#### 测试你的订阅

将此代码添加到 Playground 的末尾以测试你的新操作符：

```swift
// 41
var logger = TimeLogger(sinceOrigin: true)
// 42
let subject = PassthroughSubject<Int,Never>()
// 43
let publisher = subject.shareReplay(capacity: 2)
// 44
subject.send(0)
```

下面是这段代码的作用：

41. 使用这个 Playground 中定义的方便的 TimeLogger 对象。

42. 为了模拟在不同时间发送值，你使用了一个 Subject。

43. 分享 Subject 并仅重播最后两个值。

44. 通过 Subject 发送一个初始值。 没有订阅者连接到共享发布者，因此你应该看不到任何输出。

现在，创建你的第一个订阅并发送更多值：

```swift
let subscription1 = publisher.sink(
  receiveCompletion: {
    print("subscription1 completed: \($0)", to: &logger)
  },
  receiveValue: {
    print("subscription1 received \($0)", to: &logger)
  }
)
subject.send(1)
subject.send(2)
subject.send(3)
```

接下来，创建第二个订阅并发送更多值和完成事件：

```swift
let subscription2 = publisher.sink(
  receiveCompletion: {
    print("subscription2 completed: \($0)", to: &logger)
  },
  receiveValue: {
    print("subscription2 received \($0)", to: &logger)
  }
)

subject.send(4)
subject.send(5)
subject.send(completion: .finished)
```

这两个订阅显示他们收到的每个事件，以及自开始以来经过的时间。

添加一个稍有延迟的订阅，以确保它在发布者完成后发生：

```swift
var subscription3: Cancellable? = nil

DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
  print("Subscribing to shareReplay after upstream completed")
  subscription3 = publisher.sink(
    receiveCompletion: {
      print("subscription3 completed: \($0)", to: &logger)
    },
    receiveValue: {
      print("subscription3 received \($0)", to: &logger)
    }
  )
}
```

请记住，订阅在解除分配时终止，因此你需要使用一个变量来保留延迟的订阅。 一秒钟的延迟演示了发布者在未来如何重播数据。 你准备好测试了！ 运行 Playground 以在调试控制台中查看以下结果：

```
+0.02967s: subscription1 received 1
+0.03092s: subscription1 received 2
+0.03189s: subscription1 received 3
+0.03309s: subscription2 received 2
+0.03317s: subscription2 received 3
+0.03371s: subscription1 received 4
+0.03401s: subscription2 received 4
+0.03515s: subscription1 received 5
+0.03548s: subscription2 received 5
+0.03716s: subscription1 completed: finished
+0.03746s: subscription2 completed: finished
Subscribing to shareReplay after upstream completed
+1.12007s: subscription3 received 4
+1.12015s: subscription3 received 5
+1.12057s: subscription3 completed: finished
```

你的新操作符运行良好：

- 0 值永远不会出现在日志中，因为它是在第一个订阅者订阅共享发布者之前发出的。
- 每个值都会传播给当前和未来的订阅者。
- 你在三个值通过主题后创建了订阅 2，因此它只看到最后两个（值 2 和 3）
- 你在主题完成后创建了订阅 3，但订阅仍收到主题发出的最后两个值。
- 完成事件会正确传播，即使订阅者是在共享发布者完成之后出现的。



#### 验证你的订阅

极好的！这完全符合你的要求。或者是吗？如何验证发布者只被订阅一次？当然，通过使用 print(_:) 操作符！你可以通过在 shareReplay 之前插入它来尝试。

找到这段代码：

```swift
let publisher = subject.shareReplay(capacity: 2)
```

并将其更改为：

```
let publisher = subject
  .print("shareReplay")
  .shareReplay(capacity: 2)
```

再次运行 Playground，它将产生以下输出：

```
shareReplay: receive subscription: (PassthroughSubject)
shareReplay: request unlimited
shareReplay: receive value: (1)
+0.03004s: subscription1 received 1
shareReplay: receive value: (2)
+0.03146s: subscription1 received 2
shareReplay: receive value: (3)
+0.03239s: subscription1 received 3
+0.03364s: subscription2 received 2
+0.03374s: subscription2 received 3
shareReplay: receive value: (4)
+0.03439s: subscription1 received 4
+0.03471s: subscription2 received 4
shareReplay: receive value: (5)
+0.03577s: subscription1 received 5
+0.03609s: subscription2 received 5
shareReplay: receive finished
+0.03759s: subscription1 received completion: finished
+0.03788s: subscription2 received completion: finished
Subscribing to shareReplay after upstream completed
+1.11936s: subscription3 received 4
+1.11945s: subscription3 received 5
+1.11985s: subscription3 received completion: finished
```

所有以 shareReplay 开头的行都是显示原始 Subject 发生情况的日志。 现在你确定它只执行一次工作并与所有当前和未来的订阅者共享结果。 工作做得很好！

本章教你创建自己的发布者的几种技术。 它又长又复杂，因为有很多代码要写。 你现在差不多完成了，但在继续之前，你还需要了解最后一个主题。



### 处理背压

在流体动力学中，背压是与通过管道的所需流体流动相反的阻力或力。在 Combine 中，它是反对来自发布者的期望值流的阻力。但这个阻力是什么？通常，订阅者需要处理发布者发出的值。一些例子是：

- 处理高频数据，例如来自传感器的输入。

- 执行大文件传输。

- 在数据更新时渲染复杂的 UI。

- 等待用户输入。

- 更一般地说，处理订阅者无法以传入速度跟上的传入数据。

Combine 提供的发布者-订阅者机制是灵活的。这是一种拉式设计，而不是推式设计。这意味着订阅者要求发布者发出值并指定他们想要接收的数量。这种请求机制是自适应的：每次订阅者收到一个新值时，需求都会更新。这允许订阅者在他们不想接收更多数据时通过“关闭水龙头”来处理背压，并在他们准备好接收更多数据时“打开它”。

> 注意：请记住，你只能以累加的方式调整需求。你可以在订阅者每次收到新值时增加需求，方法是返回新的 .max(N) 或 .unlimited。或者你可以返回 .none，表示需求不应增加。然而，订阅者随后“挂机”以接收至少达到新的最大需求的值。例如，如果之前的最大需求是接收三个值，而订阅者只接收到一个，则在订阅者的 receive(_:) 中返回 .none 不会“关闭水龙头”。当发布者准备好发送它们时，订阅者仍将最多接收两个值。

当更多值可用时会发生什么完全取决于你的设计。你可以：

- 通过管理 Demand 来控制流量，以防止发布者发送超出你处理能力的值。

- 缓冲值，直到你可以处理它们 - 存在耗尽可用内存的风险。

- 删除你无法立即处理的值。

- 以上的一些组合，根据你的要求。

浏览所有可能的组合和实现可能需要几个章节。除了上述之外，处理背压还可以采取以下形式：

- 具有处理拥塞的自定义订阅的发布者。

- 在发布者链末端提供值的订阅者。

在本背压管理简介中，你将专注于实现后者。你将创建一个你已经很熟悉的 sink 函数的可暂停变体。



#### 使用可暂停的 sink 来处理背压

首先，切换到 Playground 的 PausableSink 页面。

第一步，创建一个协议，让你从暂停中恢复：

```swift
protocol Pausable {
  var paused: Bool { get }
  func resume()
}
```

这里不需要 pause() 方法，因为你将在收到每个值时确定是否暂停。当然，更复杂的可暂停订阅者可以有一个 pause() 方法，你可以随时调用！现在，你将让代码尽可能简单明了。

接下来，添加此代码以开始定义可暂停订阅者：

```swift
// 1
final class PausableSubscriber<Input, Failure: Error>:
  Subscriber, Pausable, Cancellable {
  // 2
  let combineIdentifier = CombineIdentifier()
}
```

1. 你的可暂停订阅者既可以暂停也可以取消。这是你的 pausableSink 函数将返回的对象。这也是为什么你将它作为一个类而不是一个结构来实现的原因：你不希望一个对象被复制，并且你需要在其生命周期的某些点上具有可变性。

2. 订阅者必须为 Combine 提供唯一标识符以管理和优化其发布者流。

现在添加这些附加属性：

```swift
// 3
let receiveValue: (Input) -> Bool
// 4
let receiveCompletion: (Subscribers.Completion<Failure>) -> Void

// 5
private var subscription: Subscription? = nil
// 6
var paused = false
```

3. receiveValue 闭包返回一个 Bool：true 表示它可能会收到更多的值，false 表示订阅应该暂停。

4. 完成闭包将在收到来自发布者的完成事件时被调用。

5. 保留订阅，以便它可以在暂停后请求更多值。当你不再需要它时，你需要将此属性设置为 nil 以避免保留循环。

6. 根据 Pausable 协议公开 paused 属性。

接下来，将以下代码添加到 PausableSubscriber 以实现初始化程序并符合 Cancelable 协议：

```swift
// 7
init(receiveValue: @escaping (Input) -> Bool,
     receiveCompletion: @escaping (Subscribers.Completion<Failure>) -> Void) {
  self.receiveValue = receiveValue
  self.receiveCompletion = receiveCompletion
}

// 8
func cancel() {
  subscription?.cancel()
  subscription = nil
}
```

7. 初始化器接受两个闭包，订阅者将在收到来自发布者的新值和完成时调用它们。闭包类似于你使用 sink 函数的闭包，但有一个例外：receiveValue 闭包返回一个布尔值以指示接收器是否准备好接受更多值，或者你是否需要暂停订阅。

8. 取消订阅时，不要忘记之后将其设置为 nil 以避免保留循环。

现在添加此代码以满足订阅者的要求：

```swift
func receive(subscription: Subscription) {
  // 9
  self.subscription = subscription
  // 10
  subscription.request(.max(1))
}

func receive(_ input: Input) -> Subscribers.Demand {
  // 11
  paused = receiveValue(input) == false
  // 12
  return paused ? .none : .max(1)
}

func receive(completion: Subscribers.Completion<Failure>) {
  // 13
  receiveCompletion(completion)
  subscription = nil
}
```

9. 收到发布者创建的订阅后，将其存储以备后用，以便你能够从暂停中恢复。

10. 立即请求一个值。你的订阅者可以暂停，你无法预测何时需要暂停。这里的策略是一个一个地请求值。
11. 当接收到新值时，调用 receiveValue 并相应更新暂停状态。
12. 如果订阅者被暂停，返回 .none 表示你现在不想要更多的值——记住，你最初只请求了一个。否则，请再请求一个值以保持循环继续。
    13.收到完成事件后，将其转发给receiveCompletion，然后将订阅设置为 nil，因为你不再需要它。

最后，实现 Pausable 的其余部分：

```swift
func resume() {
  guard paused else { return }

  paused = false
  // 14
  subscription?.request(.max(1))
}
```

如果发布者“暂停”，则请求一个值以重新开始循环。

就像你对以前的发布者所做的那样，你现在可以在 Publishers 命名空间中公开新的可暂停接收器。

在 Playground 的末尾添加以下代码：

```swift
extension Publisher {
  // 15
  func pausableSink(
    receiveCompletion: @escaping ((Subscribers.Completion<Failure>) -> Void),
    receiveValue: @escaping ((Output) -> Bool))
    -> Pausable & Cancellable {
    // 16
    let pausable = PausableSubscriber(
      receiveValue: receiveValue,
      receiveCompletion: receiveCompletion)
    self.subscribe(pausable)
    // 17
    return pausable
  }
}
```

15. 你的 pausableSink 操作符与 sink 操作符非常接近。唯一的区别是receiveValue 闭包的返回类型：Bool。
16. 实例化一个新的 PausableSubscriber 并将其订阅给自己，即发布者。
17. 订阅者是你用来恢复和取消订阅的对象。



#### 测试你的新 Sink

你现在可以试试你的新水槽了！为简单起见，请模拟发布者应停止发送值的情况。添加此代码：

```swift
let subscription = [1, 2, 3, 4, 5, 6]
  .publisher
  .pausableSink(receiveCompletion: { completion in
    print("Pausable subscription completed: \(completion)")
  }) { value -> Bool in
    print("Receive value: \(value)")
    if value % 2 == 1 {
      print("Pausing")
      return false
    }
    return true
}
```

数组的发布者通常按顺序发出所有值，一个接一个。使用你的可暂停接收器，此发布者将在收到值 1、3 和 5 时暂停。

运行 Playground，你会看到：

```
Receive value: 1
Pausing
```

要恢复发布者，你需要异步调用 resume()。使用计时器很容易做到这一点。添加此代码以设置计时器：

```swift
let timer = Timer.publish(every: 1, on: .main, in: .common)
  .autoconnect()
  .sink { _ in
    guard subscription.paused else { return }
    print("Subscription is paused, resuming")
    subscription.resume()
  }
```

再次运行 Playground，你会看到暂停/恢复机制在起作用：

```
Receive value: 1
Pausing
Subscription is paused, resuming
Receive value: 2
Receive value: 3
Pausing
Subscription is paused, resuming
Receive value: 4
Receive value: 5
Pausing
Subscription is paused, resuming
Receive value: 6
Pausable subscription completed: finished
```

恭喜！你现在有了一个功能性的可暂停接收器，并且你已经了解了如何处理代码中的背压！

> 注意：如果你的发布者无法保存值并等待订阅者请求它们怎么办？在这种情况下，你需要使用 buffer(size:prefetch:whenFull:) 操作符来缓冲值。此操作符可以将值缓冲到你在 size 参数中指定的容量，并在订阅者准备好接收它们时传递它们。其他参数确定缓冲区如何填满 - 订阅时立即填满，保持缓冲区满，或根据订阅者的请求 - 以及缓冲区满时会发生什么 - 即删除它收到的最后一个值，drop 最旧的或因错误而终止。



### 关键点

哇，这是一个漫长而复杂的章节！ 你学到了很多关于发布者的知识：

- 发布者可以是一种利用其他发布者方便的简单方法。
- 编写自定义发布者通常涉及创建随附的订阅。
- 订阅是订阅者和发布者之间的真正链接。
- 在大多数情况下，订阅是完成所有工作的订阅。
- 订阅者可以通过调整其需求来控制价值的传递。
- 订阅有责任尊重订阅者的需求。 Combine 不会强制执行它，但你绝对应该尊重它，将其视为 Combine 生态系统的好公民。



### 接下来去哪儿？

你了解了发布者的内部运作方式，以及如何设置机制来编写自己的内容。 当然，你编写的任何代码——尤其是发布者！ — 应彻底测试。 继续下一章，了解有关测试组合代码的所有信息！



## 第 19 节：测试 Combine 代码

研究表明，开发人员跳过编写测试有两个原因：

1. 他们编写无错误的代码。
2. 你还在读这个吗？

如果你不能直截了当地说你总是写出没有错误的代码——并且假设你对第二个回答是肯定的——那么本章就是为你准备的。 感谢你的陪伴！

编写测试是确保应用程序中预期功能的好方法，因为你正在开发新功能，尤其是在事后，以确保你的最新工作不会在以前运行良好的代码中引入回归。

本章将向你介绍针对你的 Combine 代码编写单元测试，你将在此过程中获得一些乐趣。 你将针对这个方便的应用程序编写测试：

ColorCalc 是使用 Combine 和 SwiftUI 开发的。 不过也有一些问题。 如果它只有一些体面的单元测试来帮助发现和解决这些问题。 还好你在这里！



### 入门

在 projects/starter 文件夹中打开本章的入门项目。这旨在为你输入的十六进制颜色代码提供红色、绿色、蓝色和不透明度（也称为 alpha）值。如果可能，它还将调整背景颜色以匹配当前十六进制，并在可用时给出颜色名称.如果无法从当前输入的十六进制值导出颜色，则背景将设置为白色。这就是它的设计目的。

你有一个全面的 QA 团队，他们会花时间查找和记录问题。你的工作是简化开发-QA 流程，不仅要修复这些问题，还要编写一些测试来验证修复后的正确功能。运行应用程序并确认你的 QA 团队报告的以下问题：

Issue 1

- 行动：启动应用程序。
- 预期：名称标签应显示为 aqua。
- 实际：名称标签显示 Optional(ColorCalc.ColorNam....

Issue 2

- 操作：点击 ← 按钮。
- 预期：在十六进制显示中删除最后一个字符。
- 实际：删除最后两个字符。

Issue 3

- 操作：点击 ← 按钮。

- 预期：背景变为白色。

- 实际：背景变为红色。

Issue 4

- 行动：点击⊗按钮。

- 预期：十六进制值显示清除为#。

- 实际：十六进制值显示不变。

Issue 5

- 行动：输入十六进制值 006636。

- 预期：红绿蓝不透明度显示显示 0、102、54、255。

- 实际：红-绿-蓝-不透明度显示显示 0、62、32、155。

你很快就会着手编写测试并修复这些问题，但首先，你将通过测试 Combine 的实际代码来学习测试 Combine 代码——等等！具体来说，你将测试一些操作符。

> 注意：本章假设你对 iOS 中的单元测试有一定的了解。如果没有，你仍然可以继续进行，一切都会正常进行。但是，本章不会深入研究测试驱动开发（也称为 TDD）的细节。如果你想更深入地了解该主题，请查看 raywenderlich.com 中的 iOS 测试驱动开发教程。



### 测试 Combine 操作符

在本章中，你将使用 Given-When-Then 模式来组织你的测试逻辑：

- 给定一个条件。

- 执行操作时。
- 然后出现预期的结果。

仍然在 ColorCalc 项目中，打开 ColorCalcTests/CombineOperatorsTests.swift。

首先，添加一个订阅属性来存储订阅，并在 tearDown() 中将其设置为一个空数组。 你的代码应如下所示：

```swift
var subscriptions = Set<AnyCancellable>()

override func tearDown() {
  subscriptions = []
}
```



#### 测试 collect()

你的第一个测试将针对 collect。 回想一下，这个操作符将缓冲上游发布者发出的值，等待它完成，然后在下游发出一个包含这些值的数组。

使用 Given-When-Then 模式，通过在 tearDown() 下方添加以下代码来开始新的测试方法：

```swift
func test_collect() {
  // Given
  let values = [0, 1, 2]
  let publisher = values.publisher
}
```

使用此代码，你可以创建一个整数数组，然后从该数组创建一个发布者。

现在，将此代码添加到测试中：

```swift
// When
publisher
  .collect()
  .sink(receiveValue: {
    // Then
    XCTAssert(
     $0 == values,
     "Result was expected to be \(values) but was \($0)"
    )
  })
  .store(in: &subscriptions)
```

在这里，你使用 collect 操作符，然后订阅其输出，断言输出等于 calues - 并存储订阅。

你可以通过多种方式在 Xcode 中运行单元测试：

1. 要运行单个测试，请单击方法定义旁边的菱形。
2. 要在单个测试类中运行所有测试，请单击类定义旁边的菱形。
3. 要在项目的所有测试目标中运行所有测试，请按 Command-U。请记住，每个测试目标可能包含多个测试类，每个测试类都可能包含多个测试。
4. 你还可以使用 Product ▸ Perform Action ▸ Run “TestClassName”——它也有自己的键盘快捷键：Command-Control-Option-U。

通过单击 test_collect() 旁边的菱形来运行此测试。 该项目将在执行测试时在模拟器中短暂构建和运行，然后报告它是成功还是失败。

正如预期的那样，测试将通过，你将看到以下内容：![image-20221026014631569](https://raw.githubusercontent.com/LLLLLayer/picture-bed/main/img/Kodeco/image-20221026014631569.png)

测试定义旁边的菱形也会变成绿色并包含一个复选标记。

你还可以通过 View ▸ Debug Area ▸ Activate Console 菜单项或按 Command-Shift-Y 来显示控制台以查看有关测试结果的详细信息（此处截断的结果）：

```
Test Suite 'Selected tests' passed at 2021-08-25 00:44:59.629.
     Executed 1 test, with 0 failures (0 unexpected) in 0.003 (0.007) seconds
```

要验证此测试是否正常工作，请将断言代码更改为：

```swift
XCTAssert(
  $0 == values + [1],
  "Result was expected to be \(values + [1]) but was \($0)"
)
```

你将 1 添加到与 collect() 发出的数组以及消息中的内插值进行比较的 values 数组。

重新运行测试，你会看到它失败了，并显示消息 Result 应该是 [0, 1, 2, 1] 但是是 [0, 1, 2]。你可能需要单击错误以展开并查看完整消息或显示控制台，完整消息也将打印在那里。

在继续之前撤消最后一组更改，然后重新运行测试以确保它通过。

> 注意：出于时间和空间的考虑，本章将侧重于编写测试符合条件的测试。但是，如果你有兴趣，我们鼓励你通过其他测试进行实验。请记住在继续之前将测试恢复到原始的通过状态。


这是一个相当简单的测试。下一个示例将测试一个更复杂的操作符。



#### 测试 flatMap(maxPublishers:)

正如你在第 3 节“转换操作符”中所了解的，flatMap 操作符可用于将多个上游发布者扁平化为单个发布者，你可以选择指定它将接收和扁平化的最大发布者数量。

通过添加以下代码为 flatMap 添加新的测试方法：

```swift
func test_flatMapWithMax2Publishers() {
  // Given
  // 1
  let intSubject1 = PassthroughSubject<Int, Never>()
  let intSubject2 = PassthroughSubject<Int, Never>()
  let intSubject3 = PassthroughSubject<Int, Never>()

  // 2
  let publisher = CurrentValueSubject<PassthroughSubject<Int, Never>, Never>(intSubject1)

  // 3
  let expected = [1, 2, 4]
  var results = [Int]()

  // 4
  publisher
    .flatMap(maxPublishers: .max(2)) { $0 }
    .sink(receiveValue: {
      results.append($0)
    })
    .store(in: &subscriptions)
}
```

你可以通过创建以下内容开始此测试：

1. 三个需要整数值的 PassthroughSubject 实例。

1. 一个本身接受和发布整数 PassthroughSubject 的 CurrentValueSubject，用第一个整数Subject 初始化。
2. 预期结果和一个数组来保存收到的实际结果。
3. 订阅发布者，使用最多两个发布者的 flatMap。在处理程序中，你将收到的每个值附加到结果数组中。

这照顾了Given。现在将此代码添加到你的测试中以创建操作：

```swift
// When
// 5
intSubject1.send(1)

// 6
publisher.send(intSubject2)
intSubject2.send(2)

// 7
publisher.send(intSubject3)
intSubject3.send(3)
intSubject2.send(4)

// 8
publisher.send(completion: .finished)
```

因为发布者是 CurrentValueSubject，所以它将当前值重播给新订阅者。因此，使用上面的代码，你可以继续该发布者的工作，并且：

5. 向第一个整数发布者发送一个新值。
6. 通过 CurrentValueSubject 发送第二个整数 Subject，然后向该 Subject 发送一个新值。
7. 对第三个整数 Subject 重复上一步，但这次传递两个值。
8. 通过当前值主题发送完成事件。

完成此测试剩下的就是断言这些操作将产生预期的结果。添加此代码以创建此断言：

```swift
// Then
XCTAssert(
  results == expected,
  "Results expected to be \(expected) but were \(results)"
)
```

通过单击其定义旁边的菱形来运行测试，你将看到它以绚丽的色彩通过！

如果你以前有响应式编程的经验，你可能熟悉使用测试调度程序，它是一种虚拟时间调度程序，可让你对基于时间的测试操作进行精细控制。

在撰写本文时，Combine 不包括正式的测试调度程序。 不过，一个名为 Entwine (https://github.com/tcldr/Entwine) 的开源测试调度程序已经可用，如果你需要一个正式的测试调度程序，那么值得一看。

但是，鉴于本书重点使用苹果原生的 Combine 框架，当你想测试 Combine 代码时，绝对可以使用 XCTest 的内置功能。 这将在你的下一个测试中展示。



#### 测试 publish(every_:on_:in:)

在下一个示例中，被测系统将是 Timer 发布者。

你可能还记得第 11 节“定时器”，这个发布器可用于创建重复定时器，而无需大量样板设置代码。为了测试这一点，你将使用 XCTest 的期望 API 来等待异步操作完成。

通过添加以下代码开始一个新的 test：

```swift
func test_timerPublish() {
  // Given
  // 1
  func normalized(_ ti: TimeInterval) -> TimeInterval {
    return Double(round(ti * 10) / 10)
  }

  // 2
  let now = Date().timeIntervalSinceReferenceDate
  // 3
  let expectation = self.expectation(description: #function)
  // 4
  let expected = [0.5, 1, 1.5]
  var results = [TimeInterval]()

  // 5
  let publisher = Timer
    .publish(every: 0.5, on: .main, in: .common)
    .autoconnect()
    .prefix(3)
}
```

在此设置代码中：

1. 定义一个辅助函数，通过四舍五入到小数点后标准化时间间隔。
2. 存储当前时间间隔。
3. 创建一个期望，用于等待异步操作完成。
4. 定义预期结果和一个数组来存储实际结果。
5. 创建一个自动连接的计时器发布者，并且只获取它发出的前三个值。请参阅第 11 章，“定时器”以重新了解此操作符的详细信息。

接下来，添加此代码以测试此发布者：

```swift
// When
publisher
  .sink(
    receiveCompletion: { _ in expectation.fulfill() },
    receiveValue: {
      results.append(
        normalized($0.timeIntervalSinceReferenceDate - now)
      )
    }
  )
  .store(in: &subscriptions)
```

在上面的订阅处理程序中，你使用辅助函数来获取每个发出日期的时间间隔的规范化版本，然后将其附加到结果数组中。

完成后，就该等待发布者完成其工作并完成，然后进行验证。

添加此代码以执行此操作：

```swift
// Then
// 6
waitForExpectations(timeout: 2, handler: nil)

// 7
XCTAssert(
  results == expected,
  "Results expected to be \(expected) but were \(results)"
)
```

你在这里：

6. 最多等待 2 秒。

7. 断言实际结果等于预期结果。

运行测试，你将再次通过测试——Apple 的联合团队 +1，这里的一切都像宣传的那样工作！

说到这，到目前为止，你已经测试了 Combine 内置的操作符。 为什么不测试一个自定义操作符，比如你在第 18 节“自定义发布者和处理背压”中创建的那个？



#### 测试 shareReplay(capacity:)

该操作符提供了一个常用的功能：与多个订阅者共享发布者的输出，同时还将最后 N 个值的缓冲区重播给新订阅者。此操作符采用指定滚动缓冲区大小的容量参数。再次，请参阅第 18 节，“自定义发布者和处理背压”以获取有关此操作符的更多详细信息。

你将在下一个测试中测试此操作符的共享和重播组件。添加此代码以开始使用：

```swift
func test_shareReplay() {
  // Given
  // 1
  let subject = PassthroughSubject<Int, Never>()
  // 2
  let publisher = subject.shareReplay(capacity: 2)
  // 3
  let expected = [0, 1, 2, 1, 2, 3, 3]
  var results = [Int]()
}
```

与之前的测试类似：

1. 创建一个 Subject 以向其发送新的整数值。
2. 使用容量为两个的 shareReplay 从该主题创建发布者。
3. 定义预期结果并创建一个数组来存储实际输出。

接下来，添加此代码以触发应产生预期输出的操作：

```swift
// When
// 4
publisher
  .sink(receiveValue: { results.append($0) })
  .store(in: &subscriptions)

// 5
subject.send(0)
subject.send(1)
subject.send(2)

// 6
publisher
  .sink(receiveValue: { results.append($0) })
  .store(in: &subscriptions)

// 7
subject.send(3)
```

从顶部：

4. 创建对发布者的订阅并存储任何发出的值。
5. 通过发布者分享重播的 Subject 发送一些值。
6. 创建另一个订阅并存储任何发出的值。
7. 通过 Subject 再发送一个值。

完成后，剩下的就是确保这个操作符是最新的，只需要创建一个断言。 添加此代码以结束此测试：

```swift
XCTAssert(
  results == expected,
  "Results expected to be \(expected) but were \(results)"
)
```

这是与前两个测试相同的断言代码。

运行这个测试，瞧，你有一个真正的成员值得在你的联合驱动项目中使用！

通过学习如何测试这种小范围的 Combine 操作符，你已经掌握了测试几乎任何 Combine 操作符所需要的技能。 在下一部分中，你将通过测试之前看到的 ColorCalc 应用程序来练习这些技能。



### 测试生产代码

在本章的开头，你发现了 ColorCalc 应用程序的几个问题。 现在是时候做点什么了。

该项目使用 MVVM 模式组织，你需要测试和修复的所有逻辑都包含在应用程序的唯一视图模型中：CalculatorViewModel。

> 注意：应用程序在 SwiftUI 视图文件等其他方面可能存在问题，但是 UI 测试不是本章的重点。 如果你发现自己需要针对你的 UI 代码编写单元测试，这可能表明你的代码应该重新组织以分离职责。 为此，MVVM 是一种有用的架构设计模式。 如果你想了解更多关于 MVVM 和 Combine 的信息，请查看[ MVVM 和 Combine 教程](https://www.kodeco.com/4161005-mvvm-with-combine-tutorial-for-ios)。


打开 ColorCalcTests/ColorCalcTests.swift，并在 ColorCalcTests 类定义的顶部添加以下两个属性：

```swift
var viewModel: CalculatorViewModel!
var subscriptions = Set<AnyCancellable>()
```

你将在每个测试之前重置两个属性的值，即在每个测试之前的 viewModel 和在每个测试之后的订阅。将 setUp() 和 tearDown() 方法更改为如下所示：

```swift
override func setUp() {
  viewModel = CalculatorViewModel()
}

override func tearDown() {
  subscriptions = []
}
```



#### 问题 1：显示的名称不正确

有了该设置代码，你现在可以针对视图模型编写第一个测试。添加此代码：

```swift
func test_correctNameReceived() {
  // Given
  // 1
  let expected = "rwGreen 66%"
  var result = ""

  // 2
  viewModel.$name
    .sink(receiveValue: { result = $0 })
    .store(in: &subscriptions)

  // When
  // 3
  viewModel.hexText = "006636AA"

  // Then
  // 4
  XCTAssert(
    result == expected,
    "Name expected to be \(expected) but was \(result)"
  )
}
```

这是你所做的：

1. 存储此测试的预期名称标签文本。
2. 订阅视图模型的 $name 发布者并保存接收到的值。
3. 执行应触发预期结果的操作。
4. 断言实际结果等于预期结果。

运行此测试，它将失败并显示以下消息：

```
Name expected to be rwGreen 66% but was Optional(ColorCalc.ColorName.rwGreen)66%. 
```

打开 View Models/CalculatorViewModel.swift。在类定义的底部是一个名为 configure() 的方法。这个方法在初始化器中被调用，它是所有视图模型订阅设置的地方。首先，创建一个 hexTextShared 发布者来共享 hexText 发布者。

设置 name 的订阅：

```swift
hexTextShared
  .map {
    let name = ColorName(hex: $0)
    

    if name != nil {
      return String(describing: name) +
        String(describing: Color.opacityString(forHex: $0))
    } else {
      return "------------"
    }

  }
  .assign(to: &$name)
```

现在返回 ColorCalcTests/ColorCalcTests.swift 并重新运行 test_correctNameReceived()。它通过了！

查看该代码。 你知道有什么问题吗？ 与其只检查 ColorName 的本地名称实例是否为 nil，不如使用可选绑定来解包非 nil 值。

将整 个map 代码块更改为以下内容：

```swift
.map {
  if let name = ColorName(hex: $0) {
    return "\(name) \(Color.opacityString(forHex: $0))"
  } else {
    return "------------"
  }
}
```

无需修复并重新运行项目一次来验证修复，你现在有一个测试，可以在每次运行测试时验证代码是否按预期工作。



#### 问题 2：点击退格键会删除两个字符

仍然在 ColorCalcTests.swift 中，添加这个新测试：

```swift
func test_processBackspaceDeletesLastCharacter() {
  // Given
  // 1
  let expected = "#0080F"
  var result = ""

  // 2
  viewModel.$hexText
    .dropFirst()
    .sink(receiveValue: { result = $0 })
    .store(in: &subscriptions)

  // When
  // 3
  viewModel.process(CalculatorViewModel.Constant.backspace)

  // Then
  // 4
  XCTAssert(
    result == expected,
    "Hex was expected to be \(expected) but was \(result)"
  )
}
```

与之前的测试类似：

1. 设置你期望的结果并创建一个变量来存储实际结果。
2. 订阅 viewModel.$hexText 并保存删除第一个重放值后获得的值。
3. 调用 viewModel.process(_:) 传递一个表示 ← 字符的常量字符串。
4. 断言实际结果和预期结果相等。

运行测试，如你所料，它失败了。这次的消息是 Hex 应该是#0080F 但是#0080。

回到 CalculatorViewModel 并找到 process(_:) 方法：

```swift
case Constant.backspace:
  if hexText.count > 1 {
    hexText.removeLast(2)
  }
```

这一定是在开发过程中被一些手动测试留下的。修复再简单不过了：删除 2 以便 removeLast() 只删除最后一个字符。

返回 ColorCalcTests，重新运行 test_processBackspaceDeletesLastCharacter()，它通过了！



#### 问题 3：背景颜色不正确

编写单元测试在很大程度上可能是一种反复进行的活动。下一个测试遵循与前两个相同的方法。将此新测试添加到 ColorCalcTests：

编写单元测试在很大程度上可能是一种反复进行的活动。下一个测试遵循与前两个相同的方法。将此新测试添加到 ColorCalcTests：

```swift
func test_correctColorReceived() {
  // Given
  let expected = Color(hex: ColorName.rwGreen.rawValue)!
  var result: Color = .clear

  viewModel.$color
    .sink(receiveValue: { result = $0 })
    .store(in: &subscriptions)

  // When
  viewModel.hexText = ColorName.rwGreen.rawValue

  // Then
  XCTAssert(
    result == expected,
    "Color expected to be \(expected) but was \(result)"
  )
}
```

这次你正在测试视图模型的 $color 发布者，当 viewModel.hexText 设置为 rwGreen 时，期望颜色的十六进制值是 rwGreen。起初这似乎没有做任何事情，但请记住，这是测试 $color 发布者是否为输入的十六进制值输出正确的值。

运行测试，它通过了！你做错什么了吗？绝对不！编写测试意味着尽可能主动，如果不是更被动的话。你现在有一个测试，可以验证输入的十六进制颜色是否正确。因此，请务必保持该测试以警惕未来可能出现的回归。

不过，回到这个问题的绘图板上。想想看。是什么导致了这个问题？是你输入的十六进制值，还是……等一下，又是那个←按钮！

添加此测试，以验证点击 ← 按钮时接收到正确的颜色：

```swift
func test_processBackspaceReceivesCorrectColor() {
  // Given
  // 1
  let expected = Color.white
  var result = Color.clear

  viewModel.$color
    .sink(receiveValue: { result = $0 })
    .store(in: &subscriptions)

  // When
  // 2
  viewModel.process(CalculatorViewModel.Constant.backspace)

  // Then
  // 3
  XCTAssert(
    result == expected,
    "Hex was expected to be \(expected) but was \(result)"
  )
}
```

从顶部：

1. 为预期和实际结果创建本地值，并订阅 viewModel.$color，与之前的测试相同。
2. 这次处理退格输入——而不是像之前的测试那样显式设置十六进制文本。
3. 验证结果是否符合预期。

运行此测试并失败并显示以下消息：

```
Hex was expected to be white but was red. The last word here is the most important one: red. 
```

你可能需要打开控制台才能查看整个消息。

跳回 CalculatorViewModel 并查看在 configure() 中设置颜色的订阅：

```swift
colorValuesShared
  .map { $0 != nil ? Color(values: $0!) : .red }
  .assign(to: &$color)
```

也许将背景设置为红色是另一个从未被预期值替换的快速开发时间测试？当无法从当前的十六进制值导出颜色时，该设计要求背景为白色。通过将地图实现更改为：

```swift
.map { $0 != nil ? Color(values: $0!) : .white }
```

返回 ColorCalcTests，运行 test_processBackspaceReceivesCorrectColor()，它通过了。

到目前为止，你的测试主要集中在测试正确条件上。接下来，你将对否定条件进行测试。



#### 测试错误输入

此应用程序的 UI 将阻止用户为十六进制值输入错误数据。

但是，事情可能会发生变化。例如，也许有一天你将十六进制 Text 更改为 TextField，以允许粘贴值。因此，现在添加一个测试来验证当为十六进制值输入错误数据时的预期结果是一个好主意。

将此测试添加到 ColorCalcTests：

```swift
func test_whiteColorReceivedForBadData() {
  // Given
  let expected = Color.white
  var result = Color.clear

  viewModel.$color
    .sink(receiveValue: { result = $0 })
    .store(in: &subscriptions)

  // When
  viewModel.hexText = "abc"

  // Then
  XCTAssert(
    result == expected,
    "Color expected to be \(expected) but was \(result)"
  )
}
```

本次测试与上一次几乎相同。唯一的区别是，这一次，你将错误数据传递给 hexText。

运行这个测试，它就会通过。但是，如果添加或更改了逻辑，从而可能为十六进制值输入错误数据，你的测试将在此问题交到用户手中之前发现它。

还有两个问题需要测试和修复。但是，你已经掌握了技能。因此，你将在下面的挑战部分解决剩余的问题。

在此之前，继续使用 Product ▸ Test 菜单运行所有现有测试，或者按 Command-U 通过测试。



### 挑战

#### 挑战 1：解决问题 4：点击清除不会清除十六进制显示

目前，点击⊗无效。它应该将十六进制显示清除为#。编写一个由于十六进制显示未正确更新而失败的测试，识别并修复有问题的代码，然后重新运行测试并确保它通过。

> 提示：常量 CalculatorViewModel.Constant.clear 可用于 ⊗ 字符。



**解决方案**

这个挑战的解决方案看起来几乎与你之前编写的 test_processBackspaceDeletesLastCharacter() 测试相同。唯一的区别是预期的结果只是#，并且动作是通过⊗而不是←。这个测试应该是这样的：

```swift
func test_processClearSetsHexToHashtag() {
  // Given
  let expected = "#"
  var result = ""

  viewModel.$hexText
    .dropFirst()
    .sink(receiveValue: { result = $0 })
    .store(in: &subscriptions)

  // When
  viewModel.process(CalculatorViewModel.Constant.clear)

  // Then
  XCTAssert(
    result == expected,
    "Hex was expected to be \(expected) but was \"\(result)\""
  )
}
```

按照你在本章中已经多次完成的相同分步过程，你将：

- 创建本地值以存储预期和实际结果。
- 订阅 $hexText 发布者。
- 执行应产生预期结果的操作。
- 断言预期等于实际。

按原样在项目上运行此测试将失败，并显示 Hex 应为 # 但为 "" 的消息。

研究视图模型中的相关代码，你会发现在 process(_:) 中处理 Constant.clear 输入的案例只有一个中断。

解决方法是将 break 更改为 hexText = "#"。



#### 挑战 2：解决问题 5：输入的十六进制的红绿蓝不透明度显示不正确

目前，将应用启动时显示的初始十六进制更改为其他内容后，红绿蓝不透明度 (RGBO) 显示不正确。这可能是那种从开发中得到“无法重现”响应的问题，因为它“在我的设备上运行良好”。幸运的是，你的 QA 团队在输入 006636 等值后提供了显示不正确的明确说明，这将导致 RGBO 显示设置为 0、102、54、170。

因此，你将创建的最初会失败的测试如下所示：

```swift
func test_correctRGBOTextReceived() {
  // Given
  let expected = "0, 102, 54, 170"
  var result = ""

  viewModel.$rgboText
    .sink(receiveValue: { result = $0 })
    .store(in: &subscriptions)

  // When
  viewModel.hexText = "#006636AA"

  // Then
  XCTAssert(
    result == expected,
    "RGBO text expected to be \(expected) but was \(result)"
  )
}
```

缩小到此问题的原因，你会在 CalculatorViewModel.configure() 中找到设置 RGBO 显示的订阅代码：

```swift
colorValuesShared
  .map { values -> String in
    if let values = values {
      return [values.0, values.1, values.2, values.3]
        .map { String(describing: Int($0 * 155)) }
        .joined(separator: ", ")
    } else {
      return "---, ---, ---, ---"
    }
  }
  .assign(to: &$rgboText)
```

此代码当前使用不正确的值来乘以发出的元组中返回的每个值。它应该是 255，而不是 155，因为每个红色、绿色、蓝色和不透明度字符串应该代表从 0 到 255 的基础值。

将 155 更改为 255 即可解决问题，随后测试将通过。



### 关键点

- 单元测试有助于确保你的代码在初始开发期间按预期工作，并且不会在以后引入回归。
- 你应该组织你的代码，将你将进行单元测试的业务逻辑与你将进行 UI 测试的表示逻辑分开。 MVVM 是非常适合此目的的模式。
- 它有助于使用 Given-When-Then 等模式来组织你的测试代码。
- 你可以使用期望来测试基于时间的异步组合代码。
- 测试正确和错误条件都很重要。



### 接下来去哪儿？

很棒的工作！你已经测试了几个不同的 Combine 操作符，并为以前未经测试和不守规矩的代码库带来了法律和秩序。

在你越过终点线之前还有一节。你将完成一个完整的 iOS 应用程序，该应用程序利用你在整本书中学到的知识，包括本节。
