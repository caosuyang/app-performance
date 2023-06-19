# 性能优化-AsyncDisplayKit

## AsyncDisplayKit

1. AsyncDisplayKit 是 Facebook 开源的一个用于保持 iOS 界面流畅的库。
2. AsyncDisplayKit(Public archive) -> Texture

## ASDK 的基本原理

1. ASDK 认为，阻塞主线程的任务，主要分为上面这三大类。文本和布局的计算、渲染、解码、绘制都可以通过各种方式异步执行，但 UIKit 和 Core Animation 相关操作必需在主线程进行。ASDK 的目标，就是尽量把这些任务从主线程挪走，而挪不走的，就尽量优化性能。
2. ASDK 尝试对 UIKit 组件进行封装，ASDK 为此创建了 ASDisplayNode 类，包装了常见的视图属性（比如 frame/bounds/alpha/transform/backgroundColor/superNode/subNodes 等），然后它用 UIView->CALayer 相同的方式，实现了 ASNode->UIView 这样一个关系。
3. 当不需要响应触摸事件时，ASDisplayNode 可以被设置为 layer backed，即 ASDisplayNode 充当了原来 UIView 的功能，节省了更多资源。
4. 与 UIView 和 CALayer 不同，ASDisplayNode 是线程安全的，它可以在后台线程创建和修改。Node 刚创建时，并不会在内部新建 UIView 和 CALayer，直到第一次在主线程访问 view 或 layer 属性时，它才会在内部生成对应的对象。当它的属性（比如frame/transform）改变后，它并不会立刻同步到其持有的 view 或 layer 去，而是把被改变的属性保存到内部的一个中间变量，稍后在需要时，再通过某个机制一次性设置到内部的 view 或 layer。
5. 通过模拟和封装 UIView/CALayer，开发者可以把代码中的 UIView 替换为 ASNode，很大的降低了开发和学习成本，同时能获得 ASDK 底层大量的性能优化。为了方便使用， ASDK 把大量常用控件都封装成了 ASNode 的子类，比如 Button、Control、Cell、Image、ImageView、Text、TableView、CollectionView 等。利用这些控件，开发者可以尽量避免直接使用 UIKit 相关控件，以获得更完整的性能提升。

## ASDK 的图层预合成

1. 有时一个 layer 会包含很多 sub-layer，而这些 sub-layer 并不需要响应触摸事件，也不需要进行动画和位置调整。ASDK 为此实现了一个被称为 pre-composing 的技术，可以把这些 sub-layer 合成渲染为一张图片。开发时，ASNode 已经替代了 UIView 和 CALayer；直接使用各种 Node 控件并设置为 layer backed 后，ASNode 甚至可以通过预合成来避免创建内部的 UIView 和 CALayer。
2. 通过这种方式，把一个大的层级，通过一个大的绘制方法绘制到一张图上，性能会获得很大提升。CPU 避免了创建 UIKit 对象的资源消耗，GPU 避免了多张 texture 合成和渲染的消耗，更少的 bitmap 也意味着更少的内存占用。

## ASDK 异步并发操作

1. 自 iPhone 4S 起，iDevice 已经都是双核 CPU 了，现在的 iPad 甚至已经更新到 3 核了。充分利用多核的优势、并发执行任务对保持界面流畅有很大作用。
2. ASDK 把布局计算、文本排版、图片/文本/图形渲染等操作都封装成较小的任务，并利用 GCD 异步并发执行。如果开发者使用了 ASNode 相关的控件，那么这些并发操作会自动在后台进行，无需进行过多配置。

## Runloop 任务分发

1. iOS 的显示系统是由 VSync 信号驱动的，VSync 信号由硬件时钟生成，每秒钟发出 60 次（这个值取决设备硬件，比如 iPhone 真机上通常是 59.97）。iOS 图形服务接收到 VSync 信号后，会通过 IPC 通知到 App 内。App 的 Runloop 在启动后会注册对应的 CFRunLoopSource 通过 mach_port 接收传过来的时钟信号通知，随后 Source 的回调会驱动整个 App 的动画与显示。
2. Core Animation 在 RunLoop 中注册了一个 Observer，监听了 BeforeWaiting 和 Exit 事件。这个 Observer 的优先级是 2000000，低于常见的其他 Observer。当一个触摸事件到来时，RunLoop 被唤醒，App 中的代码会执行一些操作，比如创建和调整视图层级、设置 UIView 的 frame、修改 CALayer 的透明度、为视图添加一个动画；这些操作最终都会被 CALayer 捕获，并通过 CATransaction 提交到一个中间状态去（CATransaction 的文档略有提到这些内容，但并不完整）。当上面所有操作结束后，RunLoop 即将进入休眠（或者退出）时，关注该事件的 Observer 都会得到通知。这时 CA 注册的那个 Observer 就会在回调中，把所有的中间状态合并提交到 GPU 去显示；如果此处有动画，CA 会通过 DisplayLink 等机制多次触发相关流程。
3. ASDK 在此处模拟了 Core Animation 的这个机制：所有针对 ASNode 的修改和提交，总有些任务是必需放入主线程执行的。当出现这种任务时，ASNode 会把任务用 ASAsyncTransaction(Group) 封装并提交到一个全局的容器去。ASDK 也在 RunLoop 中注册了一个 Observer，监视的事件和 CA 一样，但优先级比 CA 要低。当 RunLoop 进入休眠前、CA 处理完事件后，ASDK 就会执行该 loop 内提交的所有任务。具体代码见这个文件：ASAsyncTransactionGroup。
4. 通过这种机制，ASDK 可以在合适的机会把异步、并发的操作同步到主线程去，并且能获得不错的性能。

## 其他高级的功能

1. ASDK 中还有封装很多高级的功能，比如滑动列表的预加载、V2.0添加的新的布局模式等。
2. ASDK 是一个很庞大的库，它本身并不推荐你把整个 App 全部都改为 ASDK 驱动，把最需要提升交互性能的地方用 ASDK 进行优化就足够了。

## AsyncDisplayKit(RunLoop 的实际应用举例)

1. AsyncDisplayKit 是 Facebook 推出的用于保持界面流畅性的框架，其原理大致如下：
2. UI 线程中一旦出现繁重的任务就会导致界面卡顿，这类任务通常分为3类：排版，绘制，UI对象操作。
3. 排版通常包括计算视图大小、计算文本高度、重新计算子式图的排版等操作。绘制一般有文本绘制 (例如 CoreText)、图片绘制 (例如预先解压)、元素绘制 (Quartz)等操作。UI对象操作通常包括 UIView/CALayer 等 UI 对象的创建、设置属性和销毁。
4. 其中前两类操作可以通过各种方法扔到后台线程执行，而最后一类操作只能在主线程完成，并且有时后面的操作需要依赖前面操作的结果 （例如TextView创建时可能需要提前计算出文本的大小）。ASDK 所做的，就是尽量将能放入后台的任务放入后台，不能的则尽量推迟 (例如视图的创建、属性的调整)。
5. 为此，ASDK 创建了一个名为 ASDisplayNode 的对象，并在内部封装了 UIView/CALayer，它具有和 UIView/CALayer 相似的属性，例如 frame、backgroundColor等。所有这些属性都可以在后台线程更改，开发者可以只通过 Node 来操作其内部的 UIView/CALayer，这样就可以将排版和绘制放入了后台线程。但是无论怎么操作，这些属性总需要在某个时刻同步到主线程的 UIView/CALayer 去。
6. ASDK 仿照 QuartzCore/UIKit 框架的模式，实现了一套类似的界面更新的机制：即在主线程的 RunLoop 中添加一个 Observer，监听了 kCFRunLoopBeforeWaiting 和 kCFRunLoopExit 事件，在收到回调时，遍历所有之前放入队列的待处理的任务，然后一一执行。
7. 具体的代码可以看这里：_ASAsyncTransactionGroup。