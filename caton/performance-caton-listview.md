# 性能优化-列表卡顿

## 列表卡顿如何优化

cpu
1. 尽量使用轻量级对象，如果用不到事件处理，使用calayer代替uiview
2. 避免频繁调用view相关属性，比如布局相关frame、bounds等
3. 不要多次修改属性，尤其是属性内图片宽高布局，cell高度，提前计算好，需要时一次性设置
4. 把耗时操作比如文本尺寸计算、绘制，图片解码、绘制，放到子线程处理
5. 控制线程的最大并发数量，并发过多也会造成性能损耗
6. 考虑使用frame代替autolayout，后者消耗更多cpu资源

gpu
1. 避免短时间大量图片显示
2. 多张图片考虑合成一张图片显示
3. 注意不要超过gpu处理的最大纹理尺寸
4. 尽量减少视图数量和层次
5. 减少透明的视图
6. 圆角阴影处理，避免出现离屏渲染

## 列表卡顿优化技巧

1. 预排版
2. 预渲染
3. 异步绘制
4. 全局并发控制
5. 更高效的异步图片加载

## 如何监视CPU卡顿

1. 对于 CPU 的卡顿，它可以通过内置的 CADisplayLink 检测出来
2. FPS 指示器：FPSLabel 只有几十行代码，仅用到了 CADisplayLink 来监视 CPU 的卡顿问题。
1. 优化前：应用的列表视图在滑动时已经有很严重的卡顿了，25~35 FPS，感受到明显卡顿。
2. 优化后：Demo 列表在快速滑动时仍然能保持 50~60 FPS 的流畅交互，快速滑动感受不到卡顿，足够流畅。

## 监控卡顿手段

1. 用 Instuments 的 GPU Driver 预设，能够实时查看到 CPU 和 GPU 的资源消耗。在这个预设内，你能查看到几乎所有与显示有关的数据，比如 Texture 数量、CA 提交的频率、GPU 消耗等，在定位界面卡顿的问题时，这是最好的工具。

## 离屏渲染场景？

1. 列表 cell 处理圆角和阴影
2. 头像、按钮圆角阴影
3. 遮罩、圆角、 阴影

## 离屏渲染过程？

1. gpu有两种渲染方式，当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作
2. 当某些操作比如光栅化、遮罩、圆角、阴影，会触发离屏渲染
3. 离屏渲染，在当前屏幕缓冲区以外新开辟一个缓冲区用于渲染操作
4. 渲染过程中，需要多次切换上下文环境，先从on-screen切换到off-screen，渲染结束后，将离屏缓冲区渲染结果显示到屏幕上，又需要将上下文环境从off-screen切换回on-screen
5. 创建和切换环境带来了性能上的消耗