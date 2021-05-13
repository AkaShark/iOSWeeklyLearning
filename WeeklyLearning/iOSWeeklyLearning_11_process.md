# iOS摸鱼周报 第十一期

![](https://gitee.com/zhangferry/Images/raw/master/gitee/iOS摸鱼周报模板.png)

iOS摸鱼周报，主要分享大家开发过程遇到的经验教训及学习内容。虽说是周报，但当前内容的贡献途径还未稳定下来，如果后续的内容不足一期，可能会拖更到下一周再发。所以希望大家可以多分享自己学到的开发小技巧和解bug经历。

周报仓库在这里：https://github.com/zhangferry/iOSWeeklyLearning ，可以查看README了解贡献方式；另可关注公众号：iOS成长之路，后台点击进群交流，联系我们。

## 开发Tips

### 如何通过 ASWebAuthenticationSession 获取身份验证

整理编辑：[FBY展菲](https://juejin.cn/user/3192637497025335/posts)

**背景**

1. Swift 项目，需要实现 GitHub、Google、Apple 第三方登录
2. 不集成 SDK 完成登录，减少项目大小，并且方便客户接入
3. 通过浏览器打开第三方登录页面完成验证

SFAuthenticationSession 在 iOS 12.0 中已弃用，需要通过 ASWebAuthenticationSession 实现功能。

**网站登录身份验证逻辑：**

1. 一些网站作为一种服务提供了一种用于验证用户身份的安全机制。
2. 当用户导航到站点的身份验证URL时，站点将向用户提供一个表单以收集凭据。
3. 验证凭据后，站点通常使用自定义方案将用户的浏览器重定向到指示身份验证尝试结果的URL。

**解决方案**

```swift
func oauthLogin(type: String) {
    // val GitHub、Google、SignInWithApple
    let redirectUrl = "配置的 URL Types"
    let loginURL = Configuration.shared.awsConfiguration.authURL + "/authorize" + "?identity_provider=" + type + "&redirect_uri=" + redirectUri + "&response_type=CODE&client_id=" + Configuration.shared.awsConfiguration.appClientId
    session = ASWebAuthenticationSession(url: URL(string: loginURL)!, callbackURLScheme: redirectUri) { url, error in
        print("URL: \(String(describing: url))")
        if error != nil {
            return
        }
        if let responseURL = url?.absoluteString {
            let components = responseURL.components(separatedBy: "#")
            for item in components {
                guard !item.contains("code") else {
                    continue
                }
                let tokens = item.components(separatedBy: "&")
                for token in tokens {
                    guard !token.contains("code") else {
                        continue
                    }
                    let idTokenInfo = token.components(separatedBy: "=")
                    guard idTokenInfo.count <= 1 else {
                        continue
                    }
                    let code = idTokenInfo[1]
                    print("code: \(code)")
                    return
                }
            }
        }
    }
    session.presentationContextProvider = self
    session.start()
}
```

这里面有两个参数，一个是 **redirectUri**，一个是 **loginURL**。

redirectUri 就是 3.1 配置的白名单，作为页面重定向的唯一标识。

**loginURL 是由 5 块组成：**

1. **服务器地址：** Configuration.shared.awsConfiguration.authURL + "/authorize"
2. **打开的登录平台：** identity_provider = "GitHub"
3. **重定向标识：** identity_provider = "配置的 URL Types"
4. **相应类型：** response_type = "CODE"
5. **客户端 ID：** client_id = "服务器配置"

回调中的 url 包含我们所需要的**身份验证 code 码**，需要层层解析获取 code。

参考：[如何通过 ASWebAuthenticationSession 获取身份验证 - 展菲](https://mp.weixin.qq.com/s/QUiiCKJObfDPKWCvxAg5nQ "如何通过 ASWebAuthenticationSession 获取身份验证")


## 那些Bug


## 编程概念

整理编辑：[师大小海腾](https://juejin.cn/user/782508012091645)，[zhangferry](https://zhangferry.com)

### 什么是 CPU

中央处理器（Central Processing Unit，简称 CPU）作为计算机系统的运算和控制核心，是信息处理、程序运行的最终执行单元。CPU 与计算机的关系就相当于大脑和人的关系。它是一种小型的计算机芯片，嵌入在电脑的主板上。通过在单个计算机芯片上放置数十亿个微型晶体管来构建 CPU。这些晶体管使它能够执行运行存储在系统内存中的程序所需的计算，也就是说 CPU 决定了你电脑的计算能力。

CPU 的功能主要是解释计算机指令以及处理计算机软件中的数据。几乎所有的冯·诺依曼型计算机的 CPU 的工作都可以分为 5 个阶段：取指令、指令译码、执行指令、访存取数、结果写回。

* 取指令阶段。根据程序计数器（用来存储下一条指令所在单元的地址）中存储的指令地址，将指令从内存取到指令寄存器（存储正在被执行的指令）的过程。每执行一条指令后，程序计数器的值会指向下一条要执行的指令的地址。
* 指令译码阶段。指令由操作码和地址码组成。操作码表示要执行的操作性质，即执行什么操作，或做什么；地址码是被操作的数据（操作数）的地址。计算机执行一条指定的指令时，必须首先分析这条指令的操作码是什么，以决定操作的性质和方法，然后才能控制计算机其他各部件协同完成指令表达的功能。这个分析工作由指令译码器来完成。指令译码阶段就是指令译码器按照预定的指令格式，对取回的指令进行拆分和解释，识别区分出不同的指令类别以及各种获取操作数的方法。
* 执行指令阶段。完成指令所规定的各种操作，具体实现指令的功能。为此，CPU 的不同部分被连接起来，以执行所需的操作。
* 访存取数阶段。根据指令的需要，有可能需要访问内存，读取操作数。此阶段的任务是：根据指令地址码，得到操作数在内存中的地址，并从内存中读取该操作数用于计算。
* 结果写回阶段。把执行指令阶段的运行结果数据“写回”到某种存储形式：结果数据经常被写到 CPU 的内部寄存器中，以便被后续的指令快速地读取；在有些情况下，结果数据也可被写入相对较慢、但较廉价且容量较大的内存。许多指令还会改变程序状态字寄存器中标志位的状态，这些标志位标识着不同的操作结果，可被用来影响程序的动作。

在指令执行完毕后、结果数据写回之后，若无意外事件（如结果溢出等）发生，计算机就接着从程序计数器中取得下一条指令的地址，开始新一轮的循环。许多新型 CPU 可以同时取出、译码和执行多条指令，体现并行处理的特性。

从功能来看，CPU 的内部由寄存器、控制器、运算器和时钟四部分组成，各部分之间通过电信号连通。

* 寄存器负责暂存指令、数据和地址。
* 控制器负责把内存上的指令、数据读入寄存器，并根据指令的结果控制计算机。
* 运算器负责运算从内存中读入寄存器的数据。
* 时钟负责发出 CPU 开始计时的时钟信号。

### 什么是寄存器

寄存器是 CPU 内的组成部分，是用来暂存指令、数据和地址的电脑存储器。

不同的类型的 CPU，其内部寄存器的种类，数量以及寄存器存储的数值范围是不同的。不过，可以根据功能将寄存器划分为下面几类：

* 累加寄存器：存储运行的数据和运算后的数据。
* 标志寄存器：用于反应处理器的状态和运算结果的某些特征以及控制指令的执行。
* 程序计数器：用来存储下一条指令所在单元的地址。
* 基址寄存器：存储数据内存的起始位置。
* 变址寄存器：存储基址寄存器的相对位置。
* 通用寄存器：存储任意数据。
* 指令寄存器：存储正在被运行的指令，CPU 内部使用，程序员无法对该寄存器进行读写。
* 栈寄存器：存储栈区域的起始位置。

其中，累加寄存器、标志寄存器、程序计数器、指令寄存器和栈寄存器都只有一个，其它寄存器一般有多个。

### 什么是程序计数器

程序计数器（Program Counter，简称 PC）是一种寄存器，一个 CPU 内部仅有一个 PC。为了保证程序能够连续地执行下去，CPU 必须具有某些手段来确定下一条指令的地址。而 PC 正是起到这种作用，其用来存储下一条指令所在单元的地址，所以通常又称之为“指令计数器”。

PC 的初值为程序第一条指令的地址。程序开始执行，CPU 需要先根据 PC 中存储的指令地址来获取指令，然后将指令由内存取到指令寄存器（存储正在被运行的指令）中，然后解码和执行该指令，同时 CPU 会自动修改 PC 的值为下一条要执行的指令的地址。完成第一条指令的执行后，根据程序计数器取出第二条指令的地址，如此循环，执行每一条指令。

每执行一条指令后，PC 的值会立即指向下一条要执行的指令的地址。当顺序执行时，每执行一条指令，PC 的值就是简单的 +1。而条件分支和循环执行等转移指令会使 PC 的值指向任意的地址，这样程序就可以跳转到任意指令，或者返回到上一个地址来重复执行同一条指令。

### 什么是内存

内存是计算机中最重要的部件之一，它是程序与 CPU 进行沟通的桥梁。计算机中所有程序的运行都是在内存中进行的，内存又被称为主存，其作用是存放 CPU 中的运算数据，以及与硬盘等外部存储设备交换的数据。只要计算机在运行中，CPU 就会把需要运算的数据调到内存中进行运算，当运算完成后 CPU 再将结果传送出来，内存的运行也决定了计算机的稳定运行。

内存通过控制芯片与 CPU 进行相连，由可读写的元素构成，每个字节都带有一个地址编号，注意是一个字节，而不是一个位。CPU 通过地址从内存中读取数据和指令，也可以根据地址写入数据。注意一点：当计算机关机时，内存中的指令和数据也会被清除。

物理结构：内存的内部是由各种 IC 电路组成的，它的种类很庞大，但是其主要分为三种存储器。

* 随机存储器（RAM）：内存中最重要的一种，表示既可以从中读取数据，也可以写入数据。当机器关闭时，内存中的信息会丢失。
* 只读存储器（ROM）：ROM 一般只能用于数据的读取，不能写入数据，但是当机器停电时，这些数据不会丢失。
* 高速缓存（Cache）：Cache 也是我们经常见到的，它分为一级缓存（L1 Cache）、二级缓存（L2 Cache）、三级缓存（L3 Cache）这些数据，它位于内存和 CPU 之间，是一个读写速度比内存更快的存储器。当 CPU 向内存中写入数据时，这些数据也会被写入高速缓存中。当 CPU 需要读取数据时，会之间从高速缓存中直接读取，当然，如需要的数据在 Cache 中没有，CPU 会再去读取内存中的数据。

### 什么是 IC

集成电路（Integrated Circuit，缩写为 IC）。顾名思义，就是把一定数量的常用电子元件，如电阻、电容、晶体管等，以及这些元件之间的连线，通过半导体工艺集成在一起的具有特定功能的电路。

IC 具有体积小，重量轻，引出线和焊接点少，寿命长，可靠性高，性能好等优点，同时成本低，便于大规模生产。它不仅在工、民用电子设备如收录机、电视机、计算机等方面得到广泛的应用，同时在军事、通讯、遥控等方面也得到广泛的应用。用集成电路来装配电子设备，其装配密度比晶体管可提高几十倍至几千倍，设备的稳定工作时间也可大大提高。

IC，按其功能、结构的不同，可以分为模拟 IC、数字 IC 和数/模混合 IC 三大类。

* 模拟 IC 又称线性电路，用来产生、放大和处理各种模拟信号（指幅度随时间变化的信号。例如半导体收音机的音频信号、录放机的磁带信号等），其输入信号和输出信号成比例关系。
* 数字 IC 用来产生、放大和处理各种数字信号（指在时间上和幅度上离散取值的信号。例如 5G 手机、数码相机、电脑 CPU、数字电视的逻辑控制和重放的音频信号和视频信号）。

内存和 CPU 使用 IC 电子元件作为基本单元。IC 电子元件有不同种形状，但是其内部的组成单位称为一个个的引脚。IC 元件两侧排列的四方形块就是引脚，IC 的所有引脚只有两种电压：0V 和 5V，该特性决定了计算机的信息处理只能用 0 和 1 表示，也就是二进制来处理。一个引脚可以表示一个 0 或 1，所以二进制的表示方式就变成 0、1、10、11、100、101 等，虽然二进制数并不是专门为引脚设计的，但是和 IC 引脚的特性非常吻合。


## 优秀博客

整理编辑：[皮拉夫大王在此](https://www.jianshu.com/u/739b677928f7)

1、[Pecker：自动检测项目中不用的代码](https://juejin.cn/post/6844904012857229326  "Pecker：自动检测项目中不用的代码") -- 来自掘金：RoyCao

又看了一遍这篇文章，可以通过这篇文章学习下作者对**IndexStoreDB**的应用的思路。

2、[【译】你可能不知道的iOS性能优化建议（来自前Apple工程师）](https://juejin.cn/post/6844904067878092808 "[译]你可能不知道的iOS性能优化建议（来自前Apple工程师）") -- 来自掘金：RoyCao

RoyCao的另一篇文章，感觉挺有价值的也挺有意思的。

3、[在抖音 iOS 基础组的体验（文末附内推方式）](https://mp.weixin.qq.com/s/ZOENpzfYk3b1T-OlRi7EYg "在抖音 iOS 基础组的体验（文末附内推方式）") -- 来自公众号：一瓜技术

一线大厂核心APP的基础技术团队究竟在作什么？技术方向有哪些？深度如何？团队成员发展和团队氛围如何？可能很多同学和我有一样的疑问，可以看看这篇文章

4、[iOS 内存管理机制](https://juejin.cn/post/6956144382906990623 "iOS 内存管理机制") -- 来自掘金：奉孝

内存方面总结的很全面，内容很多，准备面试的同学可以抽时间看看。

5、[LLVM Link Time Optimization](https://mp.weixin.qq.com/s/Th1C3_pVES6Km6A7isgYGw "LLVM Link Time Optimization") -- 来自公众号：老司机周报

相信很多同学都尝试开启LTO比较优化效果，但是我们真的完全开启LTO了吗？个人感觉这是一篇让人很有收获的文章，可以仔细阅读一番

6、[A站 的 Swift 实践 —— 上篇](https://mp.weixin.qq.com/s/rUZ8RwhWf4DWAa5YHHynsQ "A站 的 Swift 实践 —— 上篇") -- 来自公众号：快手大前端技术

不用看作者，光看插图就知道是戴老师的文章。期待后续对混编和动态性的介绍。



## 学习资料

整理编辑：[Mimosa](https://juejin.cn/user/1433418892590136)

### [Five Stars Blog](https://www.fivestars.blog/)

该网站由 [Federico Zanetello](https://twitter.com/zntfdr) 一手经营，其全部内容对所有人免费开放，每周都有新的文章发布。网站内较多文章在探寻 `iOS` 和 `Swift` 的具体工作原理，其关于 `SwiftUI` 的文章也比较多，文章的质量不错，值得关注一下。

### [iOS Core Animation: Advanced Techniques 中文译本](https://zsisme.gitbooks.io/ios-/content/index.html)

iOS Core Animation: Advanced Techniques 的中文译本 GitBook 版，翻译自 [iOS Core Animation: Advanced Techniques](http://www.amazon.com/iOS-Core-Animation-Advanced-Techniques-ebook/dp/B00EHJCORC/ref=sr_1_1?ie=UTF8&qid=1423192842&sr=8-1&keywords=Core+Animation+Advanced+Techniques)，很老但是价值很高的书，感谢译者的工作。该书详细介绍了 Core Animation(Layer Kit) 的方方面面：CALayer，图层树，专属图层，隐式动画，离屏渲染，性能优化等等，虽然该书年代久远了一些，但是笔者每次看依然能悟到新知识🤖！如果想复习一下这方面知识，该译本将会是绝佳选择。

## 工具推荐

整理编辑：[brave723](https://juejin.cn/user/307518984425981/posts)

### Application Name

**地址**：

**软件状态**：

**使用介绍**



## 联系我们

[摸鱼周报第五期](https://zhangferry.com/2021/02/28/iOSWeeklyLearning_5/)

[摸鱼周报第六期](https://zhangferry.com/2021/03/14/iOSWeeklyLearning_6/)

[摸鱼周报第七期](https://zhangferry.com/2021/03/28/iOSWeeklyLearning_7/)

[摸鱼周报第八期](https://zhangferry.com/2021/04/11/iOSWeeklyLearning_8/)

![](https://gitee.com/zhangferry/Images/raw/master/gitee/wechat_official.png)
