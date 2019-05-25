在UAT的打包阶段，有打包成.pak格式的。在ProjectLauncher.log 打包日志中可以看到一句

Running: D:\Program Files\Epic Games\UE_4.21\Engine\Binaries\Win64\UnrealPak.exe D:\MyUnrealProjects\second\second.uproject -batch="D:\Program Files\Epic Games\UE_4.21\Engine\Programs\AutomationTool\Saved\UnrealPak-Commands.txt"

这一句话调用了UnrealPak程序，其中-batch之后的txt文件存储的是本次打包的相关命令

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>



<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad4/1.jpg"/>

按照官网文档创建Release版本，命令内容如下

C:\Users\ZL0032\Desktop\UE_Launcher\NoIncludeEditorContetn\WindowsNoEditor\second\Content\Paks\second-WindowsNoEditor.pak -create="C:\Users\ZL0032\AppData\Roaming\Unreal Engine\AutomationTool\Logs\D+Program+Files+Epic+Games+UE_4.21\PakList_second-WindowsNoEditor.txt" -cryptokeys=D:\MyUnrealProjects\second\Saved\Cooked\WindowsNoEditor\second\Metadata\Crypto.json -order=D:\MyUnrealProjects\second\Build\WindowsNoEditor\FileOpenOrder\CookerOpenOrder.log -patchpaddingalign=2048 



补丁包：

C:\Users\ZL0032\Desktop\UE_Launcher\1\WindowsNoEditor\second\Content\Paks\second-WindowsNoEditor_0_P.pak -create="C:\Users\ZL0032\AppData\Roaming\Unreal Engine\AutomationTool\Logs\D+Program+Files+Epic+Games+UE_4.21\PakList_second-WindowsNoEditor_0_P.txt" -cryptokeys=D:\MyUnrealProjects\second\Saved\Cooked\WindowsNoEditor\second\Metadata\Crypto.json -order=D:\MyUnrealProjects\second\Build\WindowsNoEditor\FileOpenOrder\CookerOpenOrder.log -generatepatch=D:\MyUnrealProjects\second\Releases\1.1\WindowsNoEditor\second-WindowsNoEditor*.pak -tempfiles="D:\Program Files\Epic Games\UE_4.21\TempFilessecond-WindowsNoEditor_0_P" -patchpaddingalign=2048 



命令里首先是打包的pak的名字，没有-开头。随后是

-create 

-cryptokeys 

-order 

-generatepatch（补丁专用，指定了要比较的Release文件）

-tempfiles

-patchpaddingalign    

Patch版本相比Release版本，打包文件的名字多了_0_P结尾，命令也多了一个-generatepatch



-create是获取到一个txt文件，这个txt文件保存了所有烘焙后的文件的地址和保存在pak中的地址

-order 保存烘焙文件和烘焙文件对应的order顺序

-generatepatch这个是表明这次要生成的是补丁文件，其后是Release版本的文件路径，新版本的内容将与先前Release版本的内容逐个资源进行比较，判断新加入、修改、删除的内容打出一个补丁包。



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