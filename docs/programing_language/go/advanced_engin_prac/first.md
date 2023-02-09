# 【1】Go程序是怎么跑起来的

## 开场闲聊 
### 课程介绍
课程主要分如下三方面进行讲解：
1. 语言深度：在Go语言上，希望通过调度将Go语言的代码以及底层实现串联起来，后面会讲解调试技巧（包括如何用Debugger调代码）、汇编以及反汇编（只讲工具，也就是用工具找到上层业务代码和底层实现之间的联系，不会很难，是业务关联到底层的手段，）、
   1. 调度原理：为什么放在第一课就是因为可以对Go的**启动**和**执行流程**建立简单的宏观认识
   2. 调试技巧
   3. 汇编反汇编
   4. 内部数据结构实现
   5. 常见syscall：Go语言和操作系统交互是通过系统调用，这里会讲一些系统调用的知识
   6. 函数调用规约（待定）
   7. 内存管理与垃圾回收
   8. 并发编程：介绍常见的并发手段与工具，让大家用工具写一些代码，争取可以对并发有一定了解，解决常见的并发Bug，知道内部并发库的问题
2. 应用深度：主要是web方面的知识，比如如何实现一个web框架
   1. 框架原理：trpc也会讲，会讲社区star比较多的框架，如何实现以及为什么这么实现
   2. 社区框架分析
   3. 模块分层
   4. linter规范
   5. 中台场景实现
   6. 性能调优
3. 架构广度
   1. 模块拆分
   2. CI/CD实战（coding平台）
   3. 监控与可观测性
   4. 服务发现/信息检索/定时任务/MQ等基础设施
   5. 稳定性保证
   6. 未来架构
   7. 语言前言


### 工程师的学习与进步
1. 多写代码，积累代码量(至少积累几十万的代码量，才能对设计有自己的观点)
2. 要总结和思考，如何对过去的工作进行改进(如自动化/系统化)
3. 积累自己的代码库、笔记库、开源项目。
4. 读好书，建立知识体系(比如像 Designing Data-Intensive Application 这种书，应该读好多遍）《设计数据密集型应用》相当于分布式领域的索引书
5. 关注一些靠谱的国内外新闻源，通过问题出发，主动使用 Google
6. 主动去 reddit、 hackernews 上参与讨论，避免被困在信息茧房中。建议读国外一手的资料，如果阅读英文书籍非常吃力可以背一下托福单词
7. 锻炼口才和演讲能力，内部分享 ＞ 外部分享。在公司内，该演要演，不要只是闷头干活。
   1. 费曼学习法的理论：通过输出促进自己的输入，如果可以给别人把这件事情讲清楚，说明我们就明白了。
8. 通过输出促进输入(博客、公众号、分享)，打造个人品牌，通过读者的反馈，循环提升自己的认知。
9. 信息源：Github Trending、 reddit（有专门的语言社区）、 medium、 hacker news, morning paper(作者不干了），acm.org， oreily，国外的领域相关大会（如 OSDI,SOSP，VLDB)论文，国际一流公司的技术博客，YouTube 上的国外工程师演讲。（每天花十几分钟浏览一下）

## 本节课程讲解内容目录
1. 理解可执行文件
2. Go进程的启动与初始化
3. 调度组件与调度循环
4. 处理阻塞
5. 调度器的发展历史
6. 与调度有关的常见问题

## 第一部分：理解可执行文件

### 环境安装

本节课涉及到的工具都在Dockerfile里面，可以自行实验：

1. 编写Dockerfile
```Dockerfile
FROM centos
RUN yum install golang -y \
    && yum install dlv -y \
    && yum install binutils -y \
    && yum install vim -y \
    && yum install gdb -y
```
2. build镜像出来：`docker build -t test .`
   1. `docker build`命令用于从Dockerfile文件中构建镜像，按照`docker build  -t ImageName:TagName dir`进行使用，之后可以通过命令`docker images`查看到Docker镜像
   2. `-t`参数表示给镜像增加一个Tag
   3. `.`表示Dockerfile文件所在目录，这里表示从当前目录的Dockerfile文件构建镜像
3. 运行镜像并执行bash进程，此时就可以用到课程所需要的所有命令了：`docker run -it --rm test bash`


### 编译过程：
文本 -> 编译（文本代码 -> 目标文件(.o 或者 .a) ） -> 链接（目标文件合并成二进制可执行文件）


`go build -x *.go` 可以观察到编译过程


可执行文件在不同的操作系统上面的规范是不一样的


|   Linux   |   Windows   |  Mac  |
| ---- | ---- | ---- |
|   ELF   |   PE   |   Mach-o   |

Linux 的可执行文件 ELF(Executable and Linkable Format) 为例，ELF 由几部分构成：
- ELF header
- Section header
- Sections

这里我们只是想找到Go语言的入口，所以可以只用看ELF Header就可以了

是Github上的一个同学做的二进制ELF文件的分布图（Linux、Windows、Mac都有），会详细讲解每一部分有哪些内容。

对于找入口来说，只要在ELF Header中可以找到就行。


操作系统执行可执行文件的步骤：（通过ELF Header中的内容找到相应的段，然后将二进制的段加载到内存中，最后通过entry point执行代码）
1. 解析ELF Header
2. 加载文件内容至内存
3. 从entry point开始执行代码

通过`readelf -h 可执行文件`找到可执行文件的头中的入口地址，

dlv是Go语言专用的Debug工具，通过dlv打一个断点，新加载的地址就可以看到断点打在某一个函数
![193614-KSn5Hb](https://cdn.jsdelivr.net/gh/sivanWu0222/UpicImageHosting@dev/uPic/2023-01-12/193614-KSn5Hb.png)


生成的是二进制，里面只有汇编指令，有一个通用的PC寄存器（PC 寄存器用于存放指令的地址），CPU靠PC寄存器指向的位置来执行代码，每执行完一条指令，PC寄存器都会指向下一条继续执行。


汇编学习，通过国外有一个工程师做的游戏《人力资源机器》学习，玩这个游戏就是在学习汇编，可以学习到对模块的划分，

前面讲解了计算机如何执行我们的程序，那么Go语言的二进制在执行起来之后是由什么组成的呢？由两部分组成，一部分是Go语言的runtime，另一部分是用户代码。

Runtime describes software/instructions that are executed
while your program is running, especially those
instructions that you did not write explicitly, but are
necessary for the proper execution of your code.
Low-level languages Like C have very small (if any)
runtime. More complex Languages Like Objective-C, which
allows for dynamic message passing, have a much more
extensive runtime.
可以认为 runtime 是为了实现额外的功能，而在程序运行时自动
加载/运行的一些模块。

![211149-3jZq02](https://cdn.jsdelivr.net/gh/sivanWu0222/UpicImageHosting@dev/uPic/2023-01-12/211149-3jZq02.png)
通过上面这个图我们可以看到Go二进制程序包括两部分：
1. Go程序
2. runtime


Go代码通过goroutine创建、或者channel分配、或者内存分配与runtime进行交互，runtime负责管理提交的任务，然后和操作系统通过系统调用与线程调用进行交互。

runtime相当于Go的Kernel

Go语言的runtime主要包含如下模块：
1. **Scheduler**：调度器管理所有的G，M，P，在后台执行调度循环。
2. Netpoll：网络轮询负责管理网络 FD 相关的读写、就绪事。为了支持高并发，原生的在runtime中实现了网络fd的管理流程。
3. Memory：当代码需要内存时，负责内存分配工作。
4. Garbage：当内存不再需要时，负责回收内存。

上面这些模块中，最核心的就是Scheduler，负责串联所有的runtime流程。这4个部分也是Go的4座大山，当我们跨过这4座大山之后，就没有特别难的东西了，再要去看其他代码就是一些lib的简单逻辑。

今天我们就讲解scheduler，串联起来所有逻辑