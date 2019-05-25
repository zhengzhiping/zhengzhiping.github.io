---
layout:     post
title:      UE4资源加载（四）UnrealPak打包
date:       2019-04-25
author:     BAJIAObujie
header-img: img/Heaven's_Feel.jpg
catalog: true


---

## 前言

上一篇文章讲完了从pak中加载出资源文件的过程，那么pak是如何打包生成的呢？pak是分为游戏主体包和后续补丁包两种的，两种又有什么区别呢？这是本篇文章的内容。在UE4打包的时候会启动UAT打包工具，UAT会经历烘焙内容等阶段，最后调用UnrealPak的方法将资源文件打包成pak文件。上一篇文章提到过pak打包与读取是序列化与反序列化的过程，两者是可以互相印证的。本篇文章是pak打包的过程，所以在读代码的时候，不妨与FPakPlatformFile.cpp的内容互相对比。

在4.18版本，Unreal Pak的相关代码是放在UnrealPak.cpp底下的。在4.21版本则是写到了PakFileUtilities.cpp里。文章以4.21版本为准。



## UnrealPak打包过程

以下主要以打游戏的主体包为例。

#### 1、UAT命令

打包的时候可以在对应的UE的控制台找到UnrealPak的相关日志，文章是用的ProjectLauncher来打包，在ProjectLauncher.log 打包日志中可以看到一句

Running: D:\Program Files\Epic Games\UE_4.21\Engine\Binaries\Win64\UnrealPak.exe D:\MyUnrealProjects\second\second.uproject -batch="D:\Program Files\Epic Games\UE_4.21\Engine\Programs\AutomationTool\Saved\UnrealPak-Commands.txt"

这个txt包含了对UnrealPak如何打包的具体命令。

相关的UAT代码如下，其中RunAndLog即启动UnrealPak.exe。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/2.jpg"/>

参考官网文档，文章用电脑打包以创建Release版本，其中电脑的txt命令内容如下

C:\Users\ZL0032\Desktop\UE_Launcher\NoIncludeEditorContetn\WindowsNoEditor\second\Content\Paks\second-WindowsNoEditor.pak -create="C:\Users\ZL0032\AppData\Roaming\Unreal Engine\AutomationTool\Logs\D+Program+Files+Epic+Games+UE_4.21\PakList_second-WindowsNoEditor.txt" -cryptokeys=D:\MyUnrealProjects\second\Saved\Cooked\WindowsNoEditor\second\Metadata\Crypto.json -order=D:\MyUnrealProjects\second\Build\WindowsNoEditor\FileOpenOrder\CookerOpenOrder.log -patchpaddingalign=2048 

命令里首先是打包完成后pak的存储位置，没有以"-”开头。随后是

* -create 
* -cryptokeys 
* -order 
* -patchpaddingalign=2048

-create是获取到一个txt文件，这个txt文件保存了当前版本的资源文件列表。

-order 保存烘焙文件和烘焙文件对应的order顺序

接下来就是具体的打包过程，其中可以分为两步骤，**收集需要打包的文件**以及**生成pak文件**的过程。



#### 2、收集打包文件

收集打包文件是将每个资源文件整理成FPakInputPair，保存在一个TArray<FPakInputPair> FilesToAdd里，每个FPakInputPair保存打包时候的具体信息，比如是否压缩，烘焙文件在电脑上的地址等。

前文提过命令行中的-create之后txt文件就是当前版本资源文件的列表。发布Release版本也就是主体包直接用这个资源列表就可以了，打补丁包的时候会有所不同，将在后面补充。

**读取create的txt文件**

首先读取txt中存储的内容，txt以每行对应一个文件的形式的存储，每一行又分为三部分，前一部分保存烘焙后的文件在电脑上的地址，第二部分是记录到pak中的地址（在new FPakFile会构建文件夹路径-文件名-索引的结构，这个路径正是用来构建这个结构的），第三部分是这个资源对应的一个标签，例如-compress，表明这个资源打包的时候是需要压缩的，这样有助于减少pak的文件大小。如下

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/3.jpg"/>

"D:\MyUnrealProjects\second\Saved\Cooked\WindowsNoEditor\second\AssetRegistry.bin" "../../../second/AssetRegistry.bin" -compress

将这个txt读取到TArray<FString> Lines中，每一行即代表一个资源，Lines.Num()即代表资源的数量。遍历Lines，把每一行的第一部分（本地烘焙文件地址）保存为Source，第二部分（保存到pak中的地址）保存为Dest，第三部分（-compress标签）保存为Switches。这三部分信息保存到一个FPakInputPair里，在把这个Pair保存进TArray中。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/4.jpg"/>

**读取order的txt文件**

这样就读取了-create的所有信息，并生成了FPakInputPair的数组，接着再取出-order保存的txt文件。如图

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/5.jpg"/>

其中second是我这个工程的目录，以六个省略号开头的是我本地的资源。

order命令的txt文件的读取和create命令的txt文件读取是类似的，每一行代表一个资源，左边是资源打到pak路径，相当于之前的FPakInputPair的Dest。右边是这个资源的SuggestOrder，建议顺序。每一个资源如果路径匹配则赋予对应的SuggestOrder，如果找不到则SuggestOrder默认为MAX_uint64。在对所有FPakInputPair赋予SuggestOrder后，将会对这个数组重新按照从小到大的顺序排序。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/6.jpg"/>

PS: 这个order命令其实让人很难捉摸，因为按照或者不按照这个顺序打入pak包，对最后似乎是没有太大影响的。而且对SuggestOrder赋值，似乎仅仅是对系统自带的资源有效，就是指“../../../Engine”开头的资源，而六个省略号开头的项目内的资源反而是没有赋值的。这样的结果就是优先打入引擎自带的资源，项目资源被最后才被打入包里。在我看我似乎影响不大



#### 3、生成pak文件

收集打包文件之后就可以开始生成pak文件了。

``````
bool bResult = CreatePakFile(*PakFilename, FilesToAdd, CmdLineParameters, SigningKey, EncryptionKey);
``````

* PakFilename是pak打包完成后的保存地址
* FileToAdd保存了本次pak打包的每个资源的信息。由之前的收集打包文件过程收集而来。

之前的文章曾经提过pak文件的格式，如下图。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/7.jpg"/>

打包的过程其实就是依次从上往下写入的过程，CreatePakFile函数首先建立一个FPakInfo和TArray<FPakEntryPair> Index的数组，随后将资源文件逐个写入，每个写入完毕之后，就将文件的偏移、大小记录到FPakInputPair中并加入到Index，等所有的资源全部写入后，也就是完成了文件内容区的写入后，就开始写入Index，文件索引区。写入文件索引区后，再次记录文件索引区的偏移情况以及大小到FPakInfo中，最后写入FPakInfo。这样就完成了一个pak文件。在之前的文章中有提到过new FPakFile构建文件夹路径-文件名-索引的过程，在这个过程中就是因为FPakInfo的大小固定，而且在文件末尾，所以非常方便可以读取到这部分信息，从而读取到文件索引区的。序列化与反序列化是一一对应的！

如图，在代码一开始就建立一个FPakInfo和TArray<FPakEntryPair> Index的数组。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/8.jpg"/>

在把资源文件打入文件内容区的时候，还会根据是否是本体包或者补丁包而有所区别，也在补丁包模块中讲。



## UnrealPak打补丁包

下面将介绍打补丁包与打主体包的不同

* UAT传入的命令不同
* 收集文件步骤不同

* CreatePakFile对已删除的文件的处理不同

#### 1、UAT命令不同

在打补丁包的时候，启动UnrealPak的命令如下：

C:\Users\ZL0032\Desktop\UE_Launcher\1\WindowsNoEditor\second\Content\Paks\second-WindowsNoEditor_0_P.pak -create="C:\Users\ZL0032\AppData\Roaming\Unreal Engine\AutomationTool\Logs\D+Program+Files+Epic+Games+UE_4.21\PakList_second-WindowsNoEditor_0_P.txt" -cryptokeys=D:\MyUnrealProjects\second\Saved\Cooked\WindowsNoEditor\second\Metadata\Crypto.json -order=D:\MyUnrealProjects\second\Build\WindowsNoEditor\FileOpenOrder\CookerOpenOrder.log -generatepatch=D:\MyUnrealProjects\second\Releases\1.1\WindowsNoEditor\second-WindowsNoEditor*.pak -tempfiles="D:\Program Files\Epic Games\UE_4.21\TempFilessecond-WindowsNoEditor_0_P" -patchpaddingalign=2048 

- -create 

- -cryptokeys 

- -order 

- -generatepatch（补丁专用，指定了要比较的Release文件）

- -tempfiles

- -patchpaddingalign    

Patch版本相比Release版本，主要有两点不同：

* 打包文件的名字多了_0_P结尾，这种名字结尾的pak在FPakPlatformFile读取的时候将获得更高的优先级。

* -generatepatch是表明这次要生成的是补丁文件，其后是Release版本的文件路径，补丁包是基于Release包为基础打出的，所以如果本次打包是补丁包，那么必须指明具体的Release版本，在发布Release版本的时候，除了生成对应的pak外，还会在电脑项目工程的Release目录下生成一个以ReleaseVersion为名的文件夹。在打补丁包的时候，此时已经是新版本了，新版本的内容将与先前Release版本的内容逐个资源进行比较，判断新加入、修改、删除的内容，从而打出一个补丁包。



#### 2、收集文件步骤不同

前文提过命令行中的-create之后txt文件就是当前版本资源文件的列表。Release版本也就是主体包直接用这个列表就可以了。Patch版本会有所不同，新版本与旧版本资源比较后，相同的文件是不需要重复打包的，而有变化的文件，包括删除、新增都需要打包进补丁包。那么如何分析哪些文件是相同的，哪些文件是有变化的呢？

generatepatch后面带了一个地址，这个地址的文件是先前本体pak的一个备份。UE4会找到本体pak，生成FPakFile，并对每一个文件进行分析,记录文件大小以及生成的md5值到FileInfo中，并把结果记录到 TMap<FString, FFileInfo> SourceFileHashes中。FileInfo是用来比较文件的，判断是否有变化。如下

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/9.jpg"/>

主要方法就是GenerateHashesFromPak，分析pak生成FileInfo。具体代码就不展示了，感兴趣的去看看这个方法即可。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/10.jpg"/>

在对每个文件生成了FileInfo之后，就可以来判断文件是否相同了。

* GetNewDeleteRecords方法 遍历旧版本的文件，查询文件是否在新版本还存在，如果不存在就表明被删除了。
* 将GetNewDeleteRecords记录到的已删除的文件的添加到FileToAdd中，以便打包的时候记录。此时FileToAdd中的已删除文件的FPakInputPair的bDeleteRecord已经被标记为true
* RemoveIdenticalFiles方法 移除相同的文件。

RemoveIdenticalFiles比较复杂。需要遍历每一个资源文件。一个资源文件是对应uasset/uexp组合的或者umap/uexp组合，（还有一些其他格式，如ubulk，这些我不太清楚怎么回事）在遍历资源文件的时候，只判断uasset、umap部分，就是**非uexp的部分**。通过FileIsIdentical方法。这个方法首先去判断新旧版本文件的大小，如果大小不一致，那肯定是有变化的。如果大小一致，那么判断md5值是否一致，只有一致了才判断文件是相同的。才可以从补丁包中移除。

对于文件来说除了自身数据变化，还有可能是移动、删除、新增三种状态。移动其实可以看做是删除和新增的组合吧。不过我没有试验过，应该是没有问题的。这样就完成了收集文件的步骤。



#### 3、CreatePakFile对已删除的文件的处理不同

对于本体pak来说是不存在已删除文件的，只有新版本与旧版本比较产生的补丁包才会有。

```
bool bResult = CreatePakFile(*PakFilename, FilesToAdd, CmdLineParameters, SigningKey, EncryptionKey);
```

在执行CreatePakFile代码的时候，FileToAdd保存了本次pak打包的每个资源的信息。由之前的收集打包文件过程收集而来。FileToAdd不仅仅有新增加的资源，应该说有变化的资源都会被打包，包括被删除的。当资源是被删除的时候，FileToAdd里对应的FPakInputPair的一个标志位bDeleteRecord会被标记为true，那么在把资源文件打入文件内容区的时候，在知道了这个资源被删除的时候，资源不会再被写入到文件内容区，而是只会记录到index中。这样，在读取资源的时候因为补丁包的优先级，总是优先读取到补丁包，那么索引到这个资源已经是删除的了，就不会继续往本体pak中去加载。这样就完成了不改变本体包的情况下，也可以让资源“删除”的效果，因为本体包虽然存在这个资源，但是却无法被读取到。

