# 性能优化-代码取差

## 代码瘦身方法

1. LinkMap 结合 Mach-O 找无用代码
2. 通过 AppCode 找出无用代码
3. 运行时检查类是否真正被使用过

## LinkMap 结合 Mach-O 找无用代码

1. 快速找到方法和类的全集:可以通过分析 LinkMap 来获得所有的代码类和方法的信息。获取 LinkMap 可以通过将 Build Setting 里的 Write Link Map File 设置为 Yes，然后指定 Path to Link Map File 的路径就可以得到每次编译后的 LinkMap 文件了。
2. LinkMap 文件分为三部分：Object File、Section 和 Symbols。
3. Object File 包含了代码工程的所有文件；
4. Section 描述了代码段在生成的 Mach-O 里的偏移位置和大小；
5. Symbols 会列出每个方法、类、block，以及它们的大小。
6. 通过 LinkMap ，你不光可以统计出所有的方法和类，还能够清晰地看到代码所占包大小的具体分布，进而有针对性地进行代码优化。
7. 需要找到已使用的方法和类:通过 Mach-O 取到使用过的方法和类。
8. 原理：iOS 的方法都会通过 objc_msgSend 来调用。而，objc_msgSend 在 Mach-O 文件里是通过 __objc_selrefs 这个 section 来获取 selector 这个参数的。所以，__objc_selrefs 里的方法一定是被调用了的。__objc_classrefs 里是被调用过的类，__objc_superrefs 是调用过 super 的类。通过 __objc_classrefs 和 __objc_superrefs，我们就可以找出使用过的类和子类。
9. 查看 Mach-O 文件的 __objc_selrefs、__objc_classrefs 和 __objc_superrefs。
10. 使用 MachOView 这个软件来查看 Mach-O 文件里的信息。
11. 将生成的 .app 包解开，取出 可执行文件。最后，我们就可以使用 MachOView 来查看 Mach-O 里的信息了。
12. 可以看到 __objc_selrefs、__objc_classrefs 和、__objc_superrefs 这三个 section。
13. 但是，这种查看方法并不是完美的，还会有些问题。原因在于， Objective-C 是门动态语言，方法调用可以写成在运行时动态调用，这样就无法收集全所有调用的方法和类。所以，我们通过这种方法找出的无用方法和类就只能作为参考，还需要二次确认。

## 通过 AppCode 找出无用代码

1. 如果工程量不是很大的话，我还是建议你直接使用 AppCode 来做分析。毕竟代码量达到百万行的工程并不多。而，那些代码量达到百万行的团队，则会自己通过 Clang 静态分析来开发工具，去检查无用的方法和类。
2. 用 AppCode 做分析的方法很简单，直接在 AppCode 里选择 Code->Inspect Code 就可以进行静态分析。
3. 静态分析完以后，我们可以在 Unused code 里看到所有的无用代码。
4. 无用代码的主要类型。
5. 无用类：Unused class 是无用类，Unused import statement 是无用类引入声明，Unused property 是无用的属性；
6. 无用方法：Unused method 是无用的方法，Unused parameter 是无用参数，Unused instance variable 是无用的实例变量，Unused local variable 是无用的局部变量，Unused value 是无用的值；
7. 无用宏：Unused macro 是无用的宏。
8. 无用全局：Unused global declaration 是无用全局声明。
9. AppCode 静态检查的问题：
10. JSONModel 里定义了未使用的协议会被判定为无用协议；
11. 如果子类使用了父类的方法，父类的这个方法不会被认为使用了；
12. 通过点的方式使用属性，该属性会被认为没有使用；
13. 使用 performSelector 方式调用的方法也检查不出来，比如 self performSelector:@selector(arrivalRefreshTime)；
14. 运行时声明类的情况检查不出来。比如通过 NSClassFromString 方式调用的类会被查出为没有使用的类，比如 layerClass = 15. NSClassFromString(@“SMFloatLayer”)。还有以 [[self class] accessToken] 这样不指定类名的方式使用的类，会被认为该类没有被使用。像 UITableView 的自定义的 Cell 使用 registerClass，这样的情况也会认为这个 Cell 没有被使用。
15. 基于以上种种原因，使用 AppCode 检查出来的无用代码，还需要人工二次确认才能够安全删除掉。

## 运行时检查类是否真正被使用过

1. 实际上，在 App 的不断迭代过程中，新人不断接手、业务功能需求不断替换，会留下很多无用代码。这些代码在执行静态检查时会被用到，但是线上可能连这些老功能的入口都没有了，更是没有机会被用户用到。也就是说，这些无用功能相关的代码也是可以删除的。
2. 通过 ObjC 的 runtime 源码，我们可以找到怎么判断一个类是否初始化过的函数。
3. 既然能够在运行中看到类是否初始化了，那么我们就能够找出有哪些类是没有初始化的，即找到在真实环境中没有用到的类并清理掉。
4. 具体编写运行时无用类检查工具时，我们可以在线下测试环节去检查所有类，先查出哪些类没有初始化，然后上线后针对那些没有初始化的类进行多版本监测观察，看看哪些是在主流程外个别情况下会用到的，判断合理性后进行二次确认，最终得到真正没有用到的类并删掉。

注意：以上三种代码瘦身方法，都需要进行二次确认。

1. 对于上线时间不长的新 App 和那些代码量不大的 App 来说，做些资源上的优化，再结合使用 AppCode 就能够有很好的收益。而且把这些流程加入工作流后，日常工作量也不会太大。
2. 对于代码量大，而且业务需求迭代时间很长的 App 来说，包大小的瘦身之路依然任道重远，这个领域的研究还有待继续完善。LinkMap 加 Mach-O 取差集的结果也只能作为参考，每次人工确认的成本是非常大的，只适合突击和应急清理时使用。
3. 最后日常采用的方案，可能还是用运行时检查类的方式，这种大粒度检查的方式精度虽然不高，但是人工工作量会小很多。