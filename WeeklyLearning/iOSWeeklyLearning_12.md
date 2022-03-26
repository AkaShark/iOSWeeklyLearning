# iOS摸鱼周报 第十二期

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/iOS摸鱼周报模板.png)

iOS摸鱼周报，主要分享开发过程中遇到的经验教训、优质的博客、高质量的学习资料、实用的开发工具等。周报仓库在这里：https://github.com/zhangferry/iOSWeeklyLearning ，如果你有好的的内容推荐可以通过 issue 的方式进行提交。另外也可以申请成为我们的常驻编辑，一起维护这份周报。另可关注公众号：iOS成长之路，后台点击进群交流，联系我们，获取更多内容。

## 开发Tips

### Xcode统计耗时的几个小技巧

收集几个分析项目耗时的统计小技巧。

**统计整体编译耗时**

在命令行输入以下命令：

```bash
$ defaults write com.apple.dt.Xcode ShowBuildOperationDuration -bool YES
```

此步骤之后需要重启 Xcode 才能生效，之后我们可以在 Xcode 状态栏看到整个编译阶段的耗时。

**关键阶段耗时统计**

上面的耗时可能不够详细，Xcode 还提供了一个专门用于分析各阶段耗时的功能。

菜单栏：Product > Perform Action > Build with Timing Summary

此步骤会自动触发编译，耗时统计在编译日志导航的最底部。

其还对应一个 xcodebuild 参数`-buildWithTimingSummary`，使用该参数，编译日志里也会带上个阶段耗时的统计分析。

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522134121.png)

**Swift 耗时阈值设置**

Swift 编译器提供了以下两个参数：

- `-Xfrontend -warn-long-function-bodies=<millisecond>`
- `-Xfrontend -warn-long-expression-type-checking=<millisecond>`

配置位置如下：

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522133914.png)

分别对应了长函数体编译耗时警告和长类型检查耗时警告。

一般这里输入 100 即可，表示对应类型耗时超过 100ms 将会提供警告。

### Include of non-modular header inside framework module

在组件 Framework 化的时候，如果在 public 头文件引入了另一个未 Framework 化的组件（.a静态库）时就会触发该问题。报错日志提示 Framework 里包含了非 modular 的头文件，也就是说如果我们要做 Framework 化的话，其依赖的内容也都应该是 Framework 化的，所以这个过程应该是一个从底层库到高层逐步进行的过程。如果底层依赖无法轻易修改，可以使用一些别的手段绕过这个编译错误。

Build Settings 里搜索 non-modular，将以下`Allow Non-modular Includes In Framework Modules`选项设置为 Yes。

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522133020.png)

该选项进对 OC 模块代码有作用，对于 Swift 的引用还需要加另外一个编译参数：`-Xcc -Wno-error=non-modular-include-in-framework-module`。添加位置为：

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522133508.png)

注意这两处设置均是对项目的设置，而非组件库。另外这些方案均是临时方案，最好还是要将所有依赖库全部 modular 化。

## 那些Bug

### 解决 iOS 14.5 UDP 广播 sendto 返回 -1

整理编辑：[FBY展菲](https://juejin.cn/user/3192637497025335/posts)

### 1. 问题背景

1. 手机系统升级到 iOS 14.5 之后，UDP 广播发送失败
2. 项目中老版本使用到 socket 
3. 项目中新版本使用 CocoaAsyncSocket
4. 两种 UDP 发包方式都会报错 No route to host

**报错具体内容如下：**

```
sendto: -1
client: sendto fail, but just ignore it
: No route to host
```

### 2. 问题分析

##### 2.1  sendto 返回 -1 问题排查

我们知道发送广播 sendto 返回 -1，正常情况sendto 返回值大于 0 。
首先判断 socket 连接是否建立

```objectivec
self._sck_fd4 = socket(AF_INET,SOCK_DGRAM,0);
if (DEBUG_ON) {
     NSLog(@"client init() _sck_fd4=%d",self._sck_fd4);
}
```

self._sck_fd4 打印：

```
server init(): _sck_fd4=12
```

socket 连接正常，接下来判断数据发包

```objectivec
sendto(self._sck_fd4, bytes, dataLen, 0, (struct sockaddr*)&target_addr, addr_len) = -1
```

数据发送失败

##### 2.2  增加 NSLocalNetworkUsageDescription 权限

1. Info.plist 添加 `NSLocalNetworkUsageDescription`

2. 发送一次UDP广播，触发权限弹框，让用户点击好，允许访问本地网络。

发现问题依旧存在

##### 2.3 发送单播排查

由于项目中发送广播设置的 hostName 为 `255.255.255.255`，为了排查决定先发送单播看是否能成功。

将单播地址改为 `192.168.0.101` 之后发现是可以发送成功的，然后在新版本 `CocoaAsyncSocket` 库中发送单播也是可以成功的。

UDP 广播推荐使用 `192.168.0.255` ，将广播地址改了之后，问题解决了，设备可以收到 UDP 广播数据。

### 3. 问题解决

由于 `192.168.0.255` 广播地址只是当前本地地址，App 中需要动态改变前三段 192.168.0 本地地址，解决方法如下：

```objectivec
NSString *localInetAddr4 = [ESP_NetUtil getLocalIPv4];
NSArray *arr = [localInetAddr4 componentsSeparatedByString:@"."];
NSString *deviceAddress4 = [NSString stringWithFormat:@"%@.%@.%@.255",arr[0], arr[1], arr[2]];
```

发包过滤，只需要过滤地址最后一段是否为 255

```objectivec
bool isBroadcast = [targetHostName hasSuffix:@"255"];
```

参考：[解决 iOS 14.5 UDP 广播 sendto 返回 -1 - 展菲](https://mp.weixin.qq.com/s/2SmIYq6qCTFXHDL3j6LoeA "解决 iOS 14.5 UDP 广播 sendto 返回 -1")

## 编程概念

整理编辑：[师大小海腾](https://juejin.cn/user/782508012091645)，[zhangferry](https://zhangferry.com)

这期是对 [程序必知的硬核知识大全](https://github.com/zhangferry/iOSWeeklyLearning/blob/main/Resources/Books/xuan-computer-basic.pdf "程序必知的硬核知识大全") 的第二部分总结，有兴趣的读者可以下载全文阅读。

### 什么是二进制

计算机内部是由 IC 电子元件组成的，其中 CPU 和 内存 也是 IC 电子元件的一种。IC 内部由引脚组成，所有引脚只有两种电压：0V 和 5V，该特性决定了计算机的信息处理只能用 0 和 1 表示，也就是二进制来处理。二进制由几个重要的概念，这里简要介绍下：

**补数**

如果用8位表示一个有符号整数，最高位为符号位，1 的表示为`0000 0001`。那 -1 该如何表示呢？答案是：`1111 1111`。如果你首次看到这个表示法可能很奇怪，为什么不是`1000 0001`呢，其实它是基于加法运算推演出来的结果。我们用让计算机计算`1 - 1`，即`1 + (-1)`，即`0000 0001 + 1111 1111` 结果为 `1 0000 0000`，舍去溢出的高位 1，结果就是 0 。所以 -1 对应到二进制就成了`1111 1111`。从正数到负数的表示，产生了补数的概念，它的计算是这样的：

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522135424.png)

这个就是 1 到 -1 的计算方式。

**移位运算**

移位分两种，逻辑移位和算数移位，看下面示例：

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522223515.png)

逻辑移位会在空缺部分补 0，算数右移会在空缺部分补符号位。左移的话都是补 0，没有区别。

代码中通常使用的 `>>`、`<<`符号都是算数移位，因为右移一位会让数值缩小为原来的一半，所以某些二分查找的写法会使用该技巧。

### 什么是大端小端

计算机硬件主要有两种数据存储方式，大端字节序（Big-Endian）和小端字节序（Little-Endian），其实还有一个中端字节序（Middle- Endian），因为用的很少这里不再介绍。

大端和小端的差别是什么呢，看一个例子。比如 32bit 宽的数`0x12345678`在 小端模式内存中的存放方式（假设从地址0x4000开始存放）为：

| 内存地址 | 0x4000 | 0x4001 | 0x4002 | 0x4003 |
| ---- | ------ | ------ | ------ | ------ |
| 存放内容 | 0x78   | 0x56   | 0x34   | 0x12   |

而在大端模式内存中的存放方式则为：

| 内存地址 | 0x4000 | 0x4001 | 0x4002 | 0x4003 |
| ---- | ------ | ------ | ------ | ------ |
| 存放内容 | 0x12   | 0x34   | 0x56   | 0x78   |

他们的区别是，在低字节处，小端模式存储的是低位，而大端存储的是高位。因为我们的习惯是先看高位后看低位，所以大端更符合人们的直觉；但是处理器在处理数据时先从低位处理效率会更高一些。这就是存在大端小端两种模式的原因。

目前 iOS 和macOS应用都使用的小端模式，可以通过以下方法验证：

```objectivec
if (NSHostByteOrder() == NS_BigEndian) {
     NSLog(@"BigEndian");
} else if (NSHostByteOrder() == NS_LittleEndian) {
     NSLog(@"LittleEndian");
}
```

### 什么是缓存

内存的内部是由各种 IC 电路组成的，它的种类很庞大，但是其主要分为三种存储器：随机存储器（RAM）、只读存储器（ROM）和高速缓存（Cache）。高速缓存通常又会分为一级缓存（L1 Cache）、二级缓存（L2 Cache）、三级缓存（L3 Cache），它位于内存和 CPU 之间，是一个读写速度比内存更快的存储器。以上分类从前往后速度越来越快。

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522223549.png)

为什么会有这么多缓存呢？主要是有两方面的考虑：速度和成本。读取速度越高的设备成本也会越高，为了在这两者之间进行平衡才加入了各个缓存。后者都可以作为前者的缓存，比如可以把主存作为硬盘的缓存，也可以把高速缓存作为主存的缓存。

以下是各种设备的读取速度对比：

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522143424.png)

缓存的使用逻辑大致是，没有对应数据时向上一级寻找，如果找到了就在当级缓存下来，下次再寻找相同的内容就可以直接从当级的缓存调用了。因为作为缓存的部分一般都比其上一级容量更小，所以缓存内容就不可能一直存在，需要按照一定规则进行移除，这就引出了LRU（Least Recently Used，最近最少使用）算法，它是将最近一段时间内最少被访问过的数据淘汰出缓存，提高缓存的利用率。

图片来源：[bowdoin.edu lec18.pdf](https://www.bowdoin.edu/~sbarker/teaching/courses/systems/18spring/lectures/lec18.pdf "bowdoin.edu lec18.pdf")

### 什么是压缩算法

压缩算法 (compaction algorithm) 指的就是数据压缩的算法，主要包括压缩和还原 (解压缩) 的两个步骤。其实就是在不改变原有文件属性的前提下，降低文件字节空间和占用空间的一种算法。 

根据压缩算法的定义，我们可将其分成不同的类型： 

**有损和无损** 

无损压缩：能够无失真地从压缩后的数据重构，准确地还原原始数据。可用于对数据的准确性要求严格的场合，如可执行文件和普通文件的压缩、磁盘的压缩，也可用于多媒体数据的压缩。该方法的压缩比较小。如差分编码、RLE、Huffman 编码、LZW 编码、算术编码。 

有损压缩：有失真，不能完全准确地恢复原始数据，重构的数据只是原始数据的一个近似。可用于对数据的准确性要求不高的场合，如多媒体数据的压缩。该方法的压缩比较大。例如预测编码、音感编码、分形压缩、小波压缩、JPEG/MPEG。 

**对称性** 

如果编解码算法的复杂性和所需时间差不多，则为对称的编码方法，多数压缩算法都是对称的。但也有不对称的，一般是编码难而解码容易，如 Huffman 编码和分形编码。但用于密码学的编码方法则相反，是编码容易，而解码则非常难。 

**帧间与帧内** 

在视频编码中会同时用到帧内与帧间的编码方法，帧内编码是指在一帧图像内独立完成的编码方法，同静态图像的编码，如 JPEG；而帧间编码则需要参照前后帧才能进行编解码，并在编码过程中考虑对帧之间的时间冗余的压缩，如 MPEG。 

**RLE编码**

RLE 是 run-length encoding的缩写，中文翻译为游程编码，是一种根据编码字符次数进行统计的无损压缩算法。举例来说，一组资料串"AAAABBBCCDEEEE"，由4个A、3个B、2个C、1个D、4个E组成，经过 RLE 可将资料压缩为4A3B2C1D4E（由14个单位转成10个单位）。

**哈弗曼编码**

哈夫曼编码是一种使用变长编码表对源符号进行编码的无损压缩编码方式。其特征是对于出现频率较高的字符使用较短的编码符号。对于 AAAAAABBCDDEEEEEF 这几个字符使用哈夫曼编码的结果如下：

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522213007.png)

## 优秀博客

整理编辑：[皮拉夫大王在此](https://www.jianshu.com/u/739b677928f7)

1、[普通技术人的成长路径 - 一位客户端老兵的经验之谈](https://mp.weixin.qq.com/s/IrSQyyc0J3SXBuWs9M3SYA "普通技术人的成长路径 - 一位客户端老兵的经验之谈") -- 来自公众号： 老司机周报

青衫不负踏歌行，莫忘曾经是书生。很认同其中的一些观点和思考。不知道大家是否跟我一样，存在各种各样的焦虑：客三消、内卷、35岁危机... 抽空可以读读此文，让内心的焦躁得到暂时的缓解。

2、[Swift 汇编（一）Protocol Witness Table 初探](https://mp.weixin.qq.com/s/lRKVZk5c1tX7AtWVgD56OA "Swift 汇编（一）Protocol Witness Table 初探") -- 来自公众号：Swift 社区

之前关注过 Protocol Witness Table，但是没有抽时间去了解。很有深度的一篇文章，值得阅读，如果有时间可以跟着作者的思路亲自动手调试下。

3、[Wakeup in XNU](https://mp.weixin.qq.com/s/8OBAmyCLa6_eFYqIJgoCQw "Wakeup in XNU") -- 来自公众号： 网易云音乐大前端团队

去年年底的时候在群里帮一位同学解析了一个 wakeup 日志。wakeup 日志看起来比较奇怪，可能很多同学并没有遇到类似的问题。通过这篇专业的文章可以让大家对 wakeup 有个初步了解。

4、[快手客户端稳定性体系建设](https://blog.csdn.net/Kwai_tech/article/details/107964806 "快手客户端稳定性体系建设") -- 来自CSDN：快手技术团队

这里面就提到了快手遇到了 wakeup 崩溃以及如何定位相关问题的。

5、[iOS技能拓展 初识符号与链接](https://juejin.cn/post/6961576195332309006 "iOS技能拓展 初识符号与链接") -- 来自掘金：我是好宝宝

熟悉Mach-O与链接会成为面试的加分项，正在面试的同学可以关注下。

6、[了解和分析iOS Crash](https://segmentfault.com/a/1190000016411126 "了解和分析iOS Crash") -- 来自segmentfault：腾讯WeTest

iOS crash相关很好很全面的文章，作者加了注解帮助我们理解，已收藏。

7、[A站 的 Swift 实践 —— 下篇](https://mp.weixin.qq.com/s/EIPHLdxBMb5MiRDDfxzJtA "A站 的 Swift 实践 —— 下篇") -- 来自公众号：快手大前端技术

这个是戴铭老师的 Swift 实践的下篇，相对于上篇更偏向于宏观的介绍 Swift，这篇则更加贴近实际开发场景。Swift 实战的推进有两个重要问题需要解决，一个是 Module 化，处理组件及混编问题，一个是 Swift 的 Hook 方案，处理各种 Hook 场景。

## 学习资料

整理编辑：[Mimosa](https://juejin.cn/user/1433418892590136)

### Swift by Sundell

地址：https://www.swiftbysundell.com/

[John Sundell](https://twitter.com/johnsundell) 的博客网站（同时他也是 [Publish](https://github.com/JohnSundell/Publish) 的作者），网站主要的内容是关于 `Swift` 开发的文章、播客和新闻。其文章清晰易懂，难度范围也比较广，各个水平的开发者应该都能找到适合自己水平的文章。其网站上部分关于 `SwiftUI` 的文章中，还能实时预览 `SwiftUI` 代码所对应的运行效果，贼舒服😈。

### 100 Days of SwiftUI from Paul Hudson

地址：https://www.hackingwithswift.com/100/swiftui

[Paul Hudson](https://twitter.com/twostraws) 是一个免费的 `SwiftUI` 课程，比较基础，是一个绝佳的新手教程。他会简单教一下 `Swift` 语言，然后用 `SwiftUI` 开始构建 iOS App。课程对应的网站 [Hacking with Swift](https://www.hackingwithswift.com/) 上也有很多 `iOS` 开发中比较基础的教程和解答，总的来说比较适合刚接触 `iOS` 开发的人群🤠。

## 工具推荐

整理编辑：[brave723](https://juejin.cn/user/307518984425981/posts)

### SwiftFormat for Xcode

**地址**：https://github.com/nicklockwood/SwiftFormat

**软件状态**：免费 ，开源

**使用介绍**

SwiftFormat 是用于重新格式化 Swift 代码的命令行工具。它会在保持代码意义的前提下，将代码进行格式化

很多项目都有固定的代码风格，统一的代码规范有助于项目的迭代和维护，而有的程序员却无视这些规则。同时，手动强制的去修改代码的风格又容易出错，而且没有人愿意去做这些无聊的工作。

如果有自动化的工具能完成这些工作，那几乎是最完美的方案了。在代码 review 时就不需要每次都强调无数遍繁琐的代码格式问题了。

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522213832.png)

### Notion

**地址**: https://www.notion.so/desktop

**软件状态**：个人免费，团队收费

**使用介绍**

Notion 是一款极其出色的个人笔记软件，它将“万物皆对象”的思维运用到笔记中，让使用者可以天马行空地去创造、拖拽、链接；Notion 不仅是一款优秀的个人笔记软件，其功能还涵盖了项目管理、wiki、文档等等。

##### 核心功能

* 支持导入丰富的文件和内容 
* 内置丰富的模板
* 简洁的用户界面、方便的拖动和新建操作
* 支持 Board 视图，同时可以添加任意数量的其他类型视图并自定义相关的过滤条件
* 复制图片即完成上传，无需其他图床 
* 保存历史操作记录并记录相关时间
* 强大的关联功能，比如日历与笔记，笔记与文件以及网页链接

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/20210522213919.png)

## 联系我们

[iOS摸鱼周报 第七期](https://zhangferry.com/2021/03/28/iOSWeeklyLearning_7/)

[iOS摸鱼周报 第八期](https://zhangferry.com/2021/04/11/iOSWeeklyLearning_8/)

[iOS摸鱼周报 第九期](https://zhangferry.com/2021/04/24/iOSWeeklyLearning_9/)

[iOS摸鱼周报 第十期](https://zhangferry.com/2021/05/05/iOSWeeklyLearning_10/)

[iOS摸鱼周报 第十一期](https://zhangferry.com/2021/05/16/iOSWeeklyLearning_11/)

![](http://r9ccmp2wy.hb-bkt.clouddn.com/Images/wechat_official.png)
