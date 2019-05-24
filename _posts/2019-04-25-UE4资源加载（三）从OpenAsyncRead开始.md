---
layout:     post
title:      UE4资源加载（三）从OpenAsyncRead开始
date:       2019-04-25
author:     BAJIAObujie
header-img: img/Heaven's_Feel.jpg
catalog: true

---

## 前言

这篇文章主要是要解决一个问题。在文件责任链模式下，调用OpenAsyncRead这个方法后，是怎么具体读取到文件的。

解决这个问题首先可以看看这个方法在FPakPlatformFile中的实现

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/1.jpg" />

首先先简单介绍一下概念，FPakPlatformFile是责任链中专门负责处理pak文件的一环处理器。FPakEntry对应具体要读取的uasset文件，FPakFile对应具体的磁盘上的pak文件。FPakPlatformFile持有一个FPakFile的数组，而FPakFile有包含有多个FPakEntry。（pak可以理解为压缩包，压缩包里有至少一个资源文件，这个文件就是对应FPakEntry）

然后从代码可以了解到，在FindFileInPakFiles方法中传入三个参数，一是读取的文件名字，二三是空的FPakEntry和FPakFile。这个方法返回一个bool值表明是否找到对应的文件，FileEntry被赋值为对应的文件。PakFile被赋值为包含这个FileEntry的pak文件。在找到这个具体的文件FileEntry后，返回一个异步读取的句柄。

**这个问题不关心是如何异步读取的。关注点是FindFileInPakFiles这个方法。**

在了解这个方法前必须先了解四点相关内容

1. pak文件格式

2. FPakPlatformFile、FPakFile、FPakEntry的代码结构。

3. FPakPlatformFile的初始化过程

4. 挂载Mount

## 四点相关内容

#### 1、pak文件格式

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/2.jpg" />

参考文章：<https://zhuanlan.zhihu.com/p/54531649>

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/3.jpg" />

在打包成pak的时候是按照从上到下的顺序依次写入的，而在读取pak的时候则是从下至上的顺序。读取pak文件的时候，因为FPakInfo的大小是固定的，只需要用整个pak文件的大小减去FPakInfo的大小就可以知道FPakInfo的偏移，这样就可以完整的读取到FPakInfo信息，继而也就知道了文件索引区的偏移以及大小。序列化文件索引区之后，通过FPakEntryPair就可以知道这个pak中存储了哪些文件，这些文件在pak中的偏移、大小情况。

在4.21版本中，关于打包pak的相关代码在PakFileUtilities.cpp中的CreatePakFile方法中，关于读取pak的相关代码在IPlatformFilePak.cpp中的Initialize和LoadIndex中。

打包以及读取是序列化与反序列化相反的过程，在以上两个方法中可以找到很多对应的部分。

#### 2、FPakPlatformFile、FPakFile、FPakEntry的组织结构。

* FPakPlatformFile是文件责任链中负责处理pak相关的一环处理器

* FPakFile对应磁盘上的一个pak文件

* FPakEntry对应pak中存储的单个资源文件
* <img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/4.jpg" />



**FPakPlatformFile**

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/5.jpg" />

LowerLevel不必说，是指向责任链的下一环。

FPakListEntry是由两部分组成，一个是ReadOrder，另一个是FPakFile，也就是具体的Pak文件。    TArray<FPakListEntry> PakFiles也就是保存所有挂载的Pak文件，同时这些pak文件根据ReadOrder从大到小排序。



**FPakFile**

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/6.jpg" />

Pak中存储了很多资源文件，FPakEntry对应一个具体的资源文件，在TArray<FPakEntry> Files中保存了所有的文件。每个文件在Files中都有一个对应的索引。

TMap<FString, FPakDirectory> Index；是一个路径名到FPakDirectory的映射，而FPakDirectory定义为TMap<FString, int32>。这又是一个文件名到一个具体数字的映射。这个具体的数字就是这个文件在Files中的索引。综上来看，在读取一个文件的时候，也就是传入一个文件名字的时候，是将文件名分解为文件夹路径名+文件具体名字。然后通过文件夹路径名和文件具体名字的映射得到一个索引，这样就能够知道文件对应哪一个FPakEntry。



**FPakEntry**

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/7.jpg" />

通过注释就可以知道 一个FPakEntry对应保存在pak file中的单个文件。FPakEntry记录了这个资源文件的大小，以及在pak中的偏移位置等。



#### 3、FPakPlatformFile的初始化过程（找到pak文件并挂载）

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/8.jpg" />

如图所示初始化主要分为三个步骤：

* 指定这个处理器的LowerLevel下一环处理器。

* 找到所有Pak文件夹并挂载所有Pak文件。

* 指定一些委托函数。例如想手动挂载某个pak或者卸载pak那么将会触发对应的方法。

这三个步骤中重点是第二个步骤找到pak文件并挂载。首先是找到对应的Pak文件夹

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/9.jpg" />

!UE_BUILD_SHIPPING 表明如果不是发行版本那么还可以额外指定文件夹。



<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/10.jpg" />

UE4将通过硬编码的形式，指定三个文件夹，其中第一个文件夹 ProjectContentDir + "Paks/"是我们打出来的包存放的文件夹。如图。找到所有PAK对应的文件夹后是挂载这些文件夹中的pak文件

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/11.jpg" />

挂载所有pak的过程分为三个步骤：

* 找到文件夹下的pak文件

* 找到已挂载的pak文件

* 具体挂载单个pak文件

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/12.jpg" />

挂载单个pak文件，首先是与已经挂载的pak文件进行比较，过滤重复挂载的pak文件。接着每一个pak文件必须有读取的先后顺序，得到先后顺序后，通过Mount方法挂载上。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/13.jpg" />

pak的顺序与所在文件目录有关，在ProjectContentDir下的文件优先级最高。随后将pak文件名字与pak优先级传入Mount方法里。



#### 4、挂载Mount

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/14.jpg" />

在找到pak文件后接着就是挂载的过程，挂载可以理解为把pak的包含了哪些内容生成一个目录，并加入到PakFiles中，查找文件的时候，遍历这些PakFiles，根据生成的目录可以快速索引是否这个pak的目录包含查找文件。对之前的FPakPlatformFile的结构图来说也就是构造红线的部分。新生成一个FPakFile，指定优先级顺序，加入到PakFiles中。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/15.jpg" />

挂载主要也主要分成三步骤：

一、生成FPakFile。

加载pak文件信息区以及文件索引区。知道这个pak的相关信息，存储了多少文件等等。对这些文件构建对应的路径-文件名-索引的结构。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/16.jpg" />

在Initialize函数里，首先需要读取pak文件中pak文件信息区的内容（PAK文件格式分成三部分，文件资源区，文件索引区，pak文件信息区）。pak文件信息区(对应一个FPakInfo)存放整个PAK的信息，FPakInfo是固定格式的。把整个文件大小减去FPakInfo的大小就是FPakInfo在pak文件中的偏移，（因为FPakInfo在文件的最后部分），如下图。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/17.jpg" />

在得到了pak文件信息区的内容后，接着就可以知道索引区的数量，偏移位置等等。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/18.jpg" />

在得到Pak文件信息区的内容后，接着读取文件索引区的内容。将这一部分的内容加载到内存了，首先读取MountPoint和NumEntries。接着读取TArray<FPakEntryPair> Index的内容。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/19.jpg" />

这一部分相当于是把索引区的内容给读取进来，然后读取挂点以及索引区的数量

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/20.jpg" />

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/21.jpg" />

首先是读取到资源文件的名字，然后是根据这个名字构建对应的路径-文件名-索引的关系

首先获得这个名字的路径Path，在保存的Index中寻找是否已存在文件夹，如果已存在文件夹则直接把直接把索引和文件名字放入这个文件夹（对应if中的部分）

如果不存在这个文件夹，那么依次构建文件夹，并放入（对应else的部分）

Ps：

FPaths::GetPath(Filename); 获得文件的路径

FPaths::GetCleanFilename(Filename)  获得文件的名字 包含后缀

Filename = 路径 + 名字 = FPaths::GetPath(Filename) + FPaths::GetCleanFilename(Filename)

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/22.jpg" />

构建路径-文件名-索引这种结构是为了方便在寻找一个文件的时候能够利用Hash快速索引至正确位置。这样就正确生成了FPakFile。

二、如果是补丁文件则提高补丁文件的优先级

​       补丁文件的以 _ 数字_P.pak 这种形式结尾，读取到其中的数字，再加上1，和值乘上100加到原先的优先级上。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/23.jpg" />

仍然以此图为例，本体包优先级为4，补丁包为 （0+1）*100 + 4。如果后面出新的补丁包，后缀为1、2、3。优先级就是204、304、404。





<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/24.jpg" />

三、将FPakFile添加到FPakPlatformFile上，排序。**（指定顺序与补丁包的使用有关，最后再说）**

每挂载上一个PAK文件，就是添加到FPakPlatformFile的数组上，并且按照从大到小顺序排。按照从大到小的顺序与资源加载是有关的。如果在本体包中的一个文件，在日后的更新中删除的时候，那么这个文件也会打到补丁包上，相当于只是记录到补丁包中，这个文件已经被删除了。这样读取的时候，优先读取补丁包，被删除的文件也就不会被读取到。（注：这里不是真正的删除，只是因为读取顺序的关系，读取到补丁包，这样本体包中的文件虽然没有被删除，但是却永远不会被访问到，效果等同于删除）

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/25.jpg" />

通过生成FPakFile（读取Pak文件信息区和文件索引区再构建路径-文件名-索引的结构），指定优先级顺序，加入到PakFiles并排序，这样就完成了挂载Mount的过程。

网上有问题探讨Mount挂载和加载到内存的区别，通过以上分析，可以知道Mount其实是只加载部分pak的文件信息到内存，这部分信息是这个pak存储了多少文件，文件在什么位置等等。并没有加载具体的文件信息。只有用到了具体的文件信息才会去读取。



讲完了以上四点之后，接着就是回到开头的问题了，从文件责任链开始的OpenAsyncRead之后是如何找到正确的文件的。

依据FPakPlatformFile的结构图简单来说就是，遍历所有挂载的Pak文件，每个Pak文件都有一个路径-文件名-索引的结构，我们将文件的名字拆分成这种路径加文件名，快速取得索引，这样就可以在Files中取得正确的FPakEntry。

下面是具体的代码

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/26.jpg" />

FindFileInPaiFiles方法如下，找到所有已经挂载的pak，并且从pak中寻找正确的文件

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/27.jpg" />

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/28.jpg" />其核心代码是这一段

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/29.jpg" />

对挂载的每一个PAK做一个遍历操作，每一个PAK都调用Find方法寻找文件

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad3/30.jpg" />

在Find方法中，通过FileName分割为对应的文件夹路径和具体文件资源的名字

先通过文件夹路径找到指定文件夹FPakDirectory，接着再通过具体资源名字在FPakDirectory找到具体索引。通过具体索引就可以在TArray存储所有FPakEntry的数组中找到具体索引的资源文件，这样就找到了正确的FPakEntry。



至此文章的主题部分就结束了，已经明白了是如何找到FPakEntry的。可以接着思考UE4的这种设计结构如何来配合UE4的打包。Unity的策略是，每一个文件一个Bundle，手动管理依赖关系。加载的时候加载依赖文件，这样来加载整个物体。UE4也是可以做到一样的，每一个文件单独打成一个PAK，然后AssetRegistry是可以导出文件的依赖与被依赖关系的。但是结合之前说的代码，这样的操作是非常低效的。如果我们在游戏初始化的时候就挂载所有PAK文件，那么读取的时候逐个遍历这些PAK是比较低效的。手动管理PAK的挂载与释放可以减少挂载的PAK，但也是有些麻烦的，而且也有挂载释放的消耗。应该说Pak中更适合放多个文件的。因为PAK减少了，文件的读取也可以用hash完成。其实就是遍历和哈希的时间复杂度的比较。



pak不仅适合放多个文件，UE4在ProjectLauncher里也提供了非常方便的打包工具，接下来将会介绍UE4的打包相关的内容。