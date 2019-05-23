## 前言

这是关于资源加载的第一篇内容，主要从StaticLoadObject出发，讨论UE是如何把序列化的数据给加载到内存中的。了解加载过程前必须先了解UPackage、uasset文件格式、FLinkerLoad。了解这三个概念之后会介绍StaticLoadObject加载过程所经过的四个步骤。

## UPackage、uasset、FLinkerLoad

一个资源在文件中对应uasset，在内存中对应为UPackage。

#### UPackage

一个资源在内存中表现为一个UPackage的实例，比如一个SoundCue资源，SoundCue内部可能有很多个蓝图节点，就有一些节点的数据，比如Modulator、Mixer等等，这些数据是实例本身的数据。同时SoundCue也引用外部声音文件SoundWave。SoundWave也是一个资源，也是对应的一个UPackage实例。这样两个UPackage之间就存在依赖关系。

UPackage就好比一个班级，底下的数据UObject就好比学生，对于班级（UPackage）底下的同学（UObject）来说，UPackage是UObject的Outer。要知道资源自身数据UObject的内容，必须先知道UPackage才行。

#### uasset文件格式

UPackage序列化到本地之后就是uasset文件。uasset是本地的资源文件，文件格式如图

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/1.jpg" width="250px" />

* File Summary 文件头信息

* Name Table 包中对象的名字表

* Import Table 存放被该包中对象引用的其它包中的对象信息(路径名和类型)

* Export Table 该包中的对象信息(路径名和类型)

* Export Objects 所有Export Table中对象的实际数据。

前文提过，两个UPackage实例是可以存在依赖关系的，序列化到uasset文件的时候，这些依赖关系就存储为ImportTable。可以把ImportTable看做是这个资源所依赖的其他资源的列表，ExportTable就是这个资源本身的列表。Unity导出资源的时候是导出AssetBundle + 依赖表。每个资源所依赖的其他资源都记录在依赖表中 。这里的uasset可以看做是AssetBundle + 依赖表中这个资源的依赖文件记录。其中AssetBundle就是对应的ExportTable以及ExportObject的内容，依赖表中这个资源的依赖文件记录就是对应的ImportTable。

#### FLinkerLoad

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/2.jpg" />



FLinkerLoad是作为uasset和内存UPackage的中间桥梁。在加载内容生成UPackage的时候，UPackage会根据名字找到uasset文件，由FLinkerLoad来负责加载。FLinkerLoad主要内容如下：

* FArchive* Loader; 

  //Loader负责读取具体文件

* TArray<FObjectImport> ImportMap; 

  //将uasset的ImportTable加载到ImportMap中，FObjectImport是需要依赖（导入）的UObject

* TArray<FObjectExport> ExportMap; 

  //FObjectExport是这个UPackage所拥有的UObject（这些UObject都能提供给其他UPackage作为Import）

在了解了基本概念后，接下来进入主要部分，也就是StaticLoadObject加载，StaticLoadObject可以分成四个部分。

## 加载内容的四个步骤：

1. 根据文件名字创建一个空的包（没有任何文件相关的数据）

2. 建立一个LinkerLoad去加载对应的uasset文件 序列化。

3. 优先加载ImportMap

4. 加载ExportMap（本身的数据）

对应图中右边的四个步骤

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/3.jpg" width="800px" />



序列化uasset阶段中会序列化还原这个资源所需要的信息，例如ImportMap、ExportMap，但这两个Map中存储的信息仅仅是Import和Export的信息而已，可以理解为是知道了去加载的途径，但是还没有去加载。随后在VerifyImportInner才会实际上地把Import内容加载进内存，（LoadAllObject + EndLoad）把自身资源的数据加载到内存。

#### 1、建立一个UPackage

从StaticLoadObject方法逐步看即可，略过

#### 2、序列化uasset

在FLinkerLoad的Tick函数中，会把uasset的信息给加载出来。

`FLinkerLoad::ELinkerStatus FLinkerLoad::Tick( float InTimeLimit, bool bInUseTimeLimit, bool bInUseFullTimeLimit )`

* 读取文件 CreateLoader  

* 序列化FileSummary，SerializePackageFileSummary 

  FPackageFileSummary 主要存储 比如FolderName基本字段以及uasset其余信息在文件中的偏移信息，比如ExportOffset、ExportCount。

* 序列化uasset其他信息（除FileSummary、ExportObject）比如：    SerializeImportMap、SerializeExportMap。

* 生成必要信息，这些信息不需要序列化到uasset，可以通过其余序列化信息恢复生成 CreateExportHash

* FinalizeCreation  创建LinkerLoad的最后步骤，verify 加载外部依赖UObject

verify加载外部依赖的时候就进入了第三阶段，加载ImportMap的内容。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/4.jpg" width="250px" />



#### 3、加载ImportMap

ImportMap是一个FObjectImport的数组，存储依赖的UObject，对应的ExportMap也是FObjectExport的数组。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/5.jpg"/>



verify主要是调用到FLinkerLoad::VerifyImportInner，这个函数主要分为两种情况，加载的UObject是Asset实际资产和非Asset（MemoryOnly），这两种情况还要区别是加载UObject还是UPackage。就是说加载Asset的时候可能只是加载这个资产底下的一个UObject而已，也可能是加载整个UPackage。加载非Asset的时候也有可能是加载UObject或者UPackage。**（UClass和UPackage都是继承自UObject的）**

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/6.jpg"/>

一、Import是一个MemoryOnly。

 1、Import是MemoryOnlyPackage 

加载代码里主要用到了这两个函数。

* LoadPackageInternal （主要是加载AssetPackage，不加载MemoryOnlyPackage）

* CreatePackage（优先在内存中找，否则创建）

当在LoadPackageInternal加载不到的时候，会继续在CreatePackage中查找，这个函数优先在内存中查找，而MemoryOnly正是提前存在于内存中的。找到UPackage对应的包返回即可，Import.XObject = FindPackage。

（就目前的理解来看，可能有错，加载的MemoryOnly一般是UClass或者包含UClass的UPackage，像SoundWave也是一个UClass，这些UClass包含在 /Script/UnrealEngine的UPackage中，这些UClass类以及UPackage都是在引擎启动的时候就已经加载到内存中的）

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/7.jpg"/>

 2、Import是一个UObject，那必定是Class对象（类似Java Class对象），找到TopLevelPackage，也就是这个Class对象所在的UPackage，在TopLevelPackage找到这个Class对象并赋值给Import的XObject。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/8.jpg"/>



比如加载一个SoundWave音频资源文件，在ExportMap中只有一个UObject就是音频数据。但这个音频数据需要SoundWave的类对象，所以在ImportMap中有一个UObject储存类对象，这个SoundWave类对象是属于 /Script/UnrealEngine UPackage里的，/Script/UnrealEngine就相当于保存类对象定义的地方，所以ImportMap中总共有两个UObject，一个是Class对象，一个是Package。加载的时候在UPackage中找到类对象

```
Sound.uasset
{
  ImportMap[0] SoundWave Class对象
  ImportMap[1] /Script/UnrealEngine UPackage包
  ExportMap[0] 音频数据
}
```

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/9.jpg" />

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/10.jpg"/>

* ClassName：Class表明这是一个Class对象、Package表明是一个UPackage

* ObjectName:SoundWave表明这个SoundWave类对象

* OuterIndex：-2表明这个类对象是放在索引为1的包内

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/11.jpg" />

索引换算方法：

* PackageIndex > 0,表示在ExportMap中的索引 实际索引 Index = PackageIndex -1；

* PackageIndex < 0,表示在ImportMap中的索引 实际索引 Index = - PackageIndex -1；

*  PackageIndex = 0,表示当前UPackage对象；

看ExportMap[0]的classIndex为-1，也就是说这个数据的类的数据保存在ImportMap[0]的位置。ImportMap[0]的ClassName为Class表明这是一个类，如果为Package则表明这个Import是一个UPackage。ImportMap[0]是U Package底下的一个UObject，通过ImportMap[0].OuterIndex 可以知道这个UObject的Outer的位置，OuterIndex是-2，也就是说Outer是ImportMap[1]，ImportMap[1]是一个UPackage。

二、Import是一个Asset资源。

 1、Import对应的是一个UPackage，那么会调用LoadPackageInternal，在这个函数里又会根据名字去找到对应的具体文件，然后创建一个新的UPackage。这个步骤有点类似于递归。（先假设已经完成加载Import的AssetPackage，因为LoadPackage过程是一样）, 接着让Import.SourceLinker = NewUPackage.LinkerLoad让Import持有NewUPackage的FLinkerLoad。

 2、Import表示一个UObject，UObject必定属于另一个UPackage(必定有Outer)，先去加载对应的Outer。加载完Outer之后，才加载UObject。一个包对应一个FLinkerLoad，让包1中Import.SourceLinker = 包2的Outer.SourceLinker。同时可以知道，NewUPackage（当作包2）的ExportMap肯定有一个UObject是对应着包1的ImportMap。因为两者存在引用关系。为了加载UObject，通过HashName找到对应的资源的索引，包1的Import.SourceIndex = 包2的ExportMap中对应资源的索引

附：如何寻找正确的SourceIndex 

一个资源引用另一个资源，那么必然这个资源的ObjectName（资源名字）、ClassName（类名字，比如SoundWave）、ClassPackage（类所在的Package）。这三点必然是相等的。

在读取文件序列化uasset的过程中，就有一个HashName的过程，即一个Export的ObjectName、ClassName、ClassPackage通过HashName得到一个0-255之间的一个索引，记录在一个0-255的数组中的对应位置记录上这个Export在ExportMap中的索引。

Import中也利用同样的三个值ObjectName、ClassName、ClassPackage，计算出一个同样的索引，在0-255中对应索引的位置上找到这个导出在ExportMap中的位置

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/12.jpg" />

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/13.jpg" />

总结：

以上就对四种情况分别做了介绍。

对于MemoryOnly来说，是在Import.XObject中直接记录UPackage指针 或者 UClass对象指针，Import里有一个SourceLinker表示依赖的资源所需要的FArchive，对于MemoryOnly来说是不需要依赖Asset文件的，所以是这个值是NULL。

对于Asset来说，是在Import.SourceLinker中记录资源的Loader，在Import.SourceIndex中记录资源在ExportMap中的位置。这样就可以找到Export.Object



<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/14.jpg" width="600px" />

其实方法上来讲是很相似的，加载UObject（Import加载UClass Export加载UPackage下的UObject）的时候都会先去要求Outer已经被加载，再从Outer中获取UObject。

#### 4、加载ExportMap自身数据

加载ExportMap自身数据的部分可以分成两个主要部分，**一是根据CDO类默认对象生成一个模板，二修改差异性的数据。**

**一、塑造模板的过程如下：**

1. 获得Export.Object的Archetype

2. 根据Class对象、Outer、Name、Template构建模板对象

3. 设置Linker

**获得Export.Object的Archetype**

1. 是UPackage，则取得CDO (Class Default Object），相当于类默认构造函数所构建的一个对象，一个类会在内存中放置一个CDO。

2. 不是UPackage，则应该是UPackage下的一个UObject，必须先加载到Outer，从Outer中加载原型。加载Outer的时候会一直追溯到UPackage。最后取得的UObject就相当于是CDO中对应的部分。

如果是UPackage则返回一个CDO。

如果有Outer也就是说不是UPackage 则从outer中找到原型 再从原型中找到对应的component，因为outer->getArchetype最终一定有一个Top-Level Package，这样必定返回一个类的默认对象。

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/15.jpg"  />

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/16.jpg"  />

**根据Class Outer Name Template构建模板对象**

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/17.jpg"  />

在内存中重新构建出来一个UObject

LoadClass 这个Object对应的类

ThisParent 这个Object对应的Outer

Template 这个Object对应的模板



**设置Linker**

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/18.jpg"  />

设置Export.Object对应的Linker，并添加到ObjLoaded中，在EndLoad中重新拿到ObjLoaded（需要加载的所有Export）随后真正的序列化这个物体。



**二、EndLoad调用PreLoad方法实现序列化**

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/19.jpg"  />

FAA2即FArchive下的Loader对象，与uasset文件直接关联。

Export包含了这个Object导出所存储的必要信息，在文件中的起始偏移值，文件大小。将内容加载至内存随后序列化

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/20.jpg"/>

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/21.jpg"  />

<img src="https://raw.githubusercontent.com/BAJIAObujie/BAJIAObujie.github.io/master/img/UE4ResourceLoad1/22.jpg" />

<https://yuedu.163.com/book_reader/abb2cf428b244522b17aa2ec9eeea88c_4/dfd26a58c22643bf95b8a473352d5b4c_4>



**总结：**

至此四个步骤就已经结束了，第一部分创建了一个UPackage。第二部分将读取的uasset部分数据加载到FLinkerLoad中，此时FLinkerLoad就已经知道了这个资源依赖哪些资源，自身又有哪些资源。第三部分加载ImportMap。第四部分加载ExportMap，其中加载自身数据的这个过程又分为两步，先是依据类模板对象生成一个模板，随后才序列化差异的数据。类模板对象UClass总是在ImportMap中，这可能也是为什么要先加载ImportMap的原因。



## 补充：

当资源1依赖于资源2的时候，也就是加载包1的过程中必须加载包2，例如一个SoundCue依赖于一个SoundWave，加载资源2时是根据名字去Pak中搜索对应的uasset。找到对应的uasset之后，包1ImportMap与包2ExportMap中对应的UObject 建立关联需要保证三个值不变ObjectName ClassName ClassPackage。

场景1：要更改一个资源，比如只是简单的音乐替换的话，那么ClassName ClassPackage肯定是不变的,只需要保证ObjectName不变即可。打包后，将修改后的音频资源的Pak直接替换，那么游戏的音乐就修改成功。如果需要改变ClassName ClassPackage的话，那么必须同时修改两个包才可以！ 

场景2：如果给SoundCue增加了很多功能，比如蓝图中的Mixer，Modulator。这样会增加SoundCue的依赖的Class对象，以及自身Export中的数据，与依赖的文件资源时没有关系的，这种情形下直接替换Pak即可



## 参考文章：

* UE4 Pak 文件格式  https://zhuanlan.zhihu.com/p/54531649>

* UE4 文件系统 https://zhuanlan.zhihu.com/p/35925797>

* 大象无形 <https://yuedu.163.com/book_reader/abb2cf428b244522b17aa2ec9eeea88c_4/dfd26a58c22643bf95b8a473352d5b4c_4>

* UE4对象系统_序列化和uasset文件格式 https://www.jianshu.com/p/9fea500aaa4d>

* UE4 Pak 相关知识总结 https://arcecho.github.io/2017/07/02/UE4-Pak-相关知识总结/

* UE4 APK内置资源包加载流程  https://blog.csdn.net/jiangdengc/article/details/68064895>