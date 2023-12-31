# 性能优化-包体积

## 包瘦身出发点

1. 对 App 包大小做优化的目的，就是节省用户流量，提高用户下载速度
2. App Store 规定了安装包大小超过 150MB 的 App 不能使用 OTA（over-the-air）环境下载，也就是只能在 WiFi 环境下下载。所以，150MB 就成了 App 的生死线，一旦超越了这条线就很有可能会失去大量用户。
3. 如果你的 App 要再兼容 iOS7 和 iOS8 的话，苹果官方还规定主二进制 text 段的大小不能超过 60MB。如果没有达到这个标准，你甚至都没法提交 App Store。
4. App 包体积过大，对用户更新升级率也会有很大影响。

## 包大小瘦身方法

1. App Thinning
2. 无用图片资源
3. 图片资源压缩
4. 代码瘦身

## App Thinning

1. 按照 Asset Catalog 的模板添加图片资源即可，添加的 2x 分辨率的图片和 3x 分辨率的图片，会在上传到 App Store 后被创建成不同的变体以减小 App 安装包的大小。
2. 而芯片指令集架构文件只需要按照默认的设置， App Store 就会根据设备创建不同的变体，每个变体里只有当前设备需要的那个芯片指令集架构文件。
3. 使用 App Thining 后，你可以将 2x 图和 3x 图区分开，从而达到减小 App 安装包体积的目的。如果我们要进一步减小 App 包体积的话，还需要在图片和代码上继续做优化。

## 无用图片资源

1. 删除无用图片的过程，可以概括为下面这 6 大步。
2. 通过 find 命令获取 App 安装包中的所有资源文件，比如 find /Users/daiming/Project/ -name。
3. 设置用到的资源的类型，比如 jpg、gif、png、webp。
4. 使用正则匹配在源码中找出使用到的资源名，比如 pattern = @"@"(.+?)""。
5. 使用 find 命令找到的所有资源文件，再去掉代码中使用到的资源文件，剩下的就是无用资源了。
6. 对于按照规则设置的资源名，我们需要在匹配使用资源的正则表达式里添加相应的规则，比如 @“image_%d”。
7. 确认无用资源后，就可以对这些无用资源执行删除操作了。这个删除操作，你可以使用 NSFileManger 系统类提供的功能来完成。
8. 不想自己重新写一个工具的话，可以选择开源的工具直接使用。目前最好用的是 LSUnusedResources，特别是对于使用编号规则的图片来说，可以通过直接添加规则来处理。使用方式也很简单

## 图片资源压缩

1. 对于 App 来说，图片资源总会在安装包里占个大头儿。对它们最好的处理，就是在不损失图片质量的前提下尽可能地作压缩。目前比较好的压缩方案是，将图片转成 WebP。
2. WebP 是 Google 公司的一个开源项目。
3. 图片压缩完了并不是结束，我们还需要在显示图片时使用 libwebp 进行解析。
4. WebP 在 CPU 消耗和解码时间上会比 PNG 高两倍。所以，我们有时候还需要在性能和体积上做取舍。
5. 如果图片大小超过了 100KB，你可以考虑使用 WebP；而小于 100KB 时，你可以使用网页工具 TinyPng或者 GUI 工具ImageOptim进行图片压缩。这两个工具的压缩率没有 WebP 那么高，不会改变图片压缩方式，所以解析时对性能损耗也不会增加。

## 代码瘦身

1. App 的安装包主要是由资源和可执行文件组成的
2. 可执行文件就是 Mach-O 文件，其大小是由代码量决定的。通常情况下，对可执行文件进行瘦身，就是找到并删除无用代码的过程。
3. 查找无用代码时，我们可以按照找无用图片的思路，即：
4. 首先，找出方法和类的全集；
5. 然后，找到使用过的方法和类；
6. 接下来，取二者的差集得到无用代码；
7. 最后，由人工确认无用代码可删除后，进行删除即可。

## 代码瘦身方法

1. LinkMap 结合 Mach-O 找无用代码
2. 通过 AppCode 找出无用代码
3. 运行时检查类是否真正被使用过
