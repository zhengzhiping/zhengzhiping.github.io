

## 前言

​        这是关于资源加载的第二篇内容，前一篇内容讲了，从StaticLoadObject出发，是如何把自身的数据以及依赖对象的数据给加载到内存中的，这一篇内容将继续讲文件读取的相关内容。

## 责任链设计模式

​        在进入主题之前必须先了解一下UE4为了文件读取所采用的设计模式，责任链设计模式。这是网上找的一篇讲解 [http://www.cnblogs.com/java-my-life/archive/2012/05/28/2516865.html](http://www.cnblogs.com/java-my-life/archive/2012/05/28/2516865.html)

　　来考虑这样一个功能:申请聚餐费用的管理。

　　很多公司都是这样的福利，就是项目组或者是部门可以向公司申请一些聚餐费用，用于组织项目组成员或者是部门成员进行聚餐活动。

　　申请聚餐费用的大致流程一般是：由申请人先填写申请单，然后交给领导审批，如果申请批准下来，领导会通知申请人审批通过，然后申请人去财务领取费用，如果没有批准下来，领导会通知申请人审批未通过，此事也就此作罢。

　　不同级别的领导，对于审批的额度是不一样的，比如，项目经理只能审批500元以内的申请；部门经理能审批1000元以内的申请；而总经理可以审核任意额度的申请。

　　也就是说，当某人提出聚餐费用申请的请求后，该请求会经由项目经理、部门经理、总经理之中的某一位领导来进行相应的处理，但是提出申请的人并不知道最终会由谁来处理他的请求，一般申请人是把自己的申请提交给项目经理，或许最后是由总经理来处理他的请求。

　　可以使用责任链模式来实现上述功能：当某人提出聚餐费用申请的请求后，该请求会在 **项目经理—〉部门经理—〉总经理** 这样一条领导处理链上进行传递，发出请求的人并不知道谁会来处理他的请求，每个领导会根据自己的职责范围，来判断是处理请求还是把请求交给更高级别的领导，只要有领导处理了，传递就结束了。

　　需要把每位领导的处理独立出来，实现成单独的职责处理对象，然后为它们提供一个公共的、抽象的父职责对象，这样就可以在客户端来动态地组合职责链，实现不同的功能要求了。

​        项目经理可以解决（完成 ），不可以解决->抛给部门经理去处理->部门经理不能解决再抛给总经理去处理。各人负责各自领域内的责任 。

## UE4文件责任链

在UE4中对文件读取也采用了责任链的设计模式。好处是将负责一类功能的（比如专门负责读取Pak文件）给单独抽取了出来，保持类的单一职责。在这个链上的各个处理器只负责分内的事，如果请求不是自己处理直接抛给下一个就好了。链上的每一个环都继承自IPlatformFile接口。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad2/2.jpg"/>

https://zhuanlan.zhihu.com/p/35925797

在加载uasset文件的时候调用到了方法

`Handle = FPlatformFileManager::Get().GetPlatformFile().OpenAsyncRead(InFileName); `

其中FPlatformFileManager 是一个单例的管理器，持有一个TopmostPlayformFile，也就是责任链的头部位置。每个具体实现的类在初始化的时候都会持有下一个IPlatformFile的引用。这样TopmostPlayformFile处理不了的文件可以一直往下传递，直到被处理或者传递到链的末尾。

相关方法如下

```c++
FPlatformFileManager 

/** Currently used platform file. */
class IPlatformFile* TopmostPlatformFile; //类里的唯一变量

FPlatformFileManager& FPlatformFileManager::Get() //单例模式
{
  static FPlatformFileManager Singleton;
  return Singleton;
}

IPlatformFile& FPlatformFileManager::GetPlatformFile() //获得链顶部的IPlatform处理器
{
  if (TopmostPlatformFile == NULL)
  {
     TopmostPlatformFile = &IPlatformFile::GetPlatformPhysical();
  }
  return *TopmostPlatformFile;
}
```



**如何生成责任链：**

以安卓为例，启动过程：

1. 在LaunchAndroid.cpp调用AndroidMain 程序入口

2. 调用FEngineLoop::PreInit

3. 在PreInit中调用 LaunchCheckForFileOverride

4. 在LaunchCheckForFileOverride生成读取文件的责任链

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad2/4.jpg"/>

最开始责任链只有一个physical platform file，随后加入读取PakFile的处理器。此时physical platform file为PakFile的LowerLevel，也就是处理器的下一环。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad2/5.jpg"/>

ConditionallyCreateFileWrapper在获得相应的IPlatformFile之后会调用Initialize初始化方法，CurrentPlatformFile也就成了新生成的IPlatformFile的LowerLevel。



**FPakPlatformFile的结构**

生成责任链后，链上的每一环各自负责自己的内容，其中FPakPlatformFile是专门处理Pak的一环。这里对FPakPlatformFile只做简要介绍，下一篇内容会详细讲解Pak格式相关。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad2/3.jpg" width="600px"/>

* LowerLevel 链上下一环处理器

* TArray<FPakListEntry> PakFiles存储所有Pak文件

FPakPlatformFile的PakFiles 将几个指定目录下所有pak为后缀的文件挂载到内存中，是挂载而不是加载，加载意味着整个pak文件数据都加载到内存中。而挂载不同，pak是打包了很多uasset、uexp文件的这样一个文件。挂载仅仅只是对这些uasset生成一个目录，加载某一个uasset文件的时候，就可以在这个目录下查找，这个pak是否含有这个uasset文件，如果有，才把pak中对应资源相关的数据给加载到内存。