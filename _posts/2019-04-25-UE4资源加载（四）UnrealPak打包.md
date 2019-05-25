## 前言

上一篇文章讲完了从pak中加载出资源文件的过程，那么pak是如何打包生成的呢？pak是分为游戏主体包和后续补丁包两种的，两种又有什么区别呢？这是本篇文章的内容。在UE4打包的时候会启动UAT打包工具，UAT会经历烘焙内容等阶段，最后调用UnrealPak的方法将资源文件打包成pak文件。上一篇文章提到过pak打包与读取是序列化与反序列化的过程，两者是可以互相印证的。本篇文章是pak打包的过程，所以在读代码的时候，不妨与FPakPlatformFile.cpp的内容互相对比。

在4.18版本，Unreal Pak的相关代码是放在UnrealPak.cpp底下的。在4.21版本则是写到了PakFileUtilities.cpp里。文章以4.21版本为准。



## UnrealPak打包过程

以下主要以打游戏的主体包为例。打包的时候可以在对应的UE的控制台找到UnrealPak的相关日志，文章是用的ProjectLauncher来打包，在ProjectLauncher.log 打包日志中可以看到一句

Running: D:\Program Files\Epic Games\UE_4.21\Engine\Binaries\Win64\UnrealPak.exe D:\MyUnrealProjects\second\second.uproject -batch="D:\Program Files\Epic Games\UE_4.21\Engine\Programs\AutomationTool\Saved\UnrealPak-Commands.txt"

启动UnrealPak的相关命令就是存储在这个txt中，这一句话调用了UnrealPak程序，其中-batch之后的txt文件存储的是本次打包的相关命令

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>



<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>

按照官网文档创建Release版本，命令内容如下

C:\Users\ZL0032\Desktop\UE_Launcher\NoIncludeEditorContetn\WindowsNoEditor\second\Content\Paks\second-WindowsNoEditor.pak -create="C:\Users\ZL0032\AppData\Roaming\Unreal Engine\AutomationTool\Logs\D+Program+Files+Epic+Games+UE_4.21\PakList_second-WindowsNoEditor.txt" -cryptokeys=D:\MyUnrealProjects\second\Saved\Cooked\WindowsNoEditor\second\Metadata\Crypto.json -order=D:\MyUnrealProjects\second\Build\WindowsNoEditor\FileOpenOrder\CookerOpenOrder.log -patchpaddingalign=2048 



补丁包：

C:\Users\ZL0032\Desktop\UE_Launcher\1\WindowsNoEditor\second\Content\Paks\second-WindowsNoEditor_0_P.pak -create="C:\Users\ZL0032\AppData\Roaming\Unreal Engine\AutomationTool\Logs\D+Program+Files+Epic+Games+UE_4.21\PakList_second-WindowsNoEditor_0_P.txt" -cryptokeys=D:\MyUnrealProjects\second\Saved\Cooked\WindowsNoEditor\second\Metadata\Crypto.json -order=D:\MyUnrealProjects\second\Build\WindowsNoEditor\FileOpenOrder\CookerOpenOrder.log -generatepatch=D:\MyUnrealProjects\second\Releases\1.1\WindowsNoEditor\second-WindowsNoEditor*.pak -tempfiles="D:\Program Files\Epic Games\UE_4.21\TempFilessecond-WindowsNoEditor_0_P" -patchpaddingalign=2048 



命令里首先是打包完成后pak的存储位置，没有以"-”开头。随后是

* -create 

* -cryptokeys 

* -order 

* -generatepatch（补丁专用，指定了要比较的Release文件）

* -tempfiles

* -patchpaddingalign    

Patch版本相比Release版本，打包文件的名字多了_0_P结尾，命令也多了一个-generatepatch

-create是获取到一个txt文件，这个txt文件保存了所有烘焙后的文件在电脑上的地址和保存在pak中的地址

-order 保存烘焙文件和烘焙文件对应的order顺序

-generatepatch这个是表明这次要生成的是补丁文件，其后是Release版本的文件路径，补丁包是基于Release包为基础打出的，所以如果本次打包是补丁包，那么必须指明具体的Release版本，在发布Release版本的时候，除了生成对应的pak外，还会在电脑项目工程的Release目录下生成一个以ReleaseVersion为名的文件夹。在打补丁包的时候，此时已经是新版本了，新版本的内容将与先前Release版本的内容逐个资源进行比较，判断新加入、修改、删除的内容，从而打出一个补丁包。



调用UnrealPak后会执行到ExecuteUnrealPak这个函数

打包的过程可以分为**收集需要打包的文件**以及**生成pak文件**的过程。



**收集需要打包的文件**

命令行中的-create之后txt文件就是本次打包文件的列表。Release版本直接用这个列表就可以了。Patch版本需要用这个文件列表与之前的pak中的资源进行对比，重新整理出一份打包文件列表。

Release和Patch资源对比。

Patch首先读取到之前的pak文件，遍历读取所有pak文件，pak文件中保存许多资源文件，每一个资源文件生成一个对应的FFileInfo （包含文件大小、md5值）。如下

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>



这个函数传入先前Release文件夹路径，接着找到所有pak文件，对所有资源文件生成FFileInfo

保存在TMap<FString, FFileInfo> SourceFileHashes 中，这是一个资源文件名字到FFileInfo的映射。



接着会调用TArray<FPakInputPair> DeleteRecords = GetNewDeleteRecords(FilesToAdd, SourceFileHashes); 



<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>



FilesToAdd是txt中保存的本次打包的所有文件，得到的DeleteRecords 经过对比Release包需要删除的内容。这个DeleteRecords 会加入到本次打包的pak文件中。

先讲一下pak对删除文件的打包策略。对release包存在的内容，新版本中已经不存在的内容。仍然需要打进补丁包的内容，但是这个内容就相当于一个覆盖。对删除文件会在补丁包中的文件索引区中加入，但是不会把实际资源内容保存到文件内容区。因为补丁包的优先关系，在读取文件的时候，优先读取到补丁包中的内容，知道这个内容被标记为删除，那么就不会继续访问。这样release包中虽然资源并没有被删除，但是可以说等效于删除，因为不会被访问到。



<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>

DeleteRecords就是打在文件索引区的删除标记。

在这个函数里会遍历之前release的pak文件，判断先前的文件在本次打包中是否仍然存在，如果不存在就生成一个DeleteRecord并记录到DeleteRecords。



对已经删除掉的资源的处理方式是在补丁包中加入一个删除标记。此时FilesToAdd里还有两次版本都没有变化的内容。需要移除掉这些重复的内容。调用RemoveIdenticalFiles(FilesToAdd, CmdLineParameters.SourcePatchDiffDirectory, SourceFileHashes); 判断是否是重复内容的方法是先判断资源的大小，如果大小一致接着判断资源的md5值。只有同时满足才表明资源是一样的。如果不一样就会被打入补丁包中。



**生成pak文件**

bool bResult = CreatePakFile(*PakFilename, FilesToAdd, CmdLineParameters, SigningKey, EncryptionKey);

就是生成上图文件格式的过程。































































<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>







<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>







<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>