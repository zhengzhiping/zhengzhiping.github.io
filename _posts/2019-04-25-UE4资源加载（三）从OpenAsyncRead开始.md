## 前言

这篇文章主要是要解决一个问题。在文件责任链模式下，调用OpenAsyncRead这个方法后，是怎么具体读取到文件的。

解决这个问题首先可以看看这个方法在FPakPlatformFile中的实现



首先先简单介绍一下概念，FPakPlatformFile是责任链中专门负责处理pak文件的一环处理器。FPakEntry就是具体要读取的uasset文件，FPakFile对应具体的磁盘上的pak文件。FPakPlatformFile持有一个FPakFile的数组，而FPakFile有包含有多个FPakEntry。（pak可以理解为压缩包，压缩包里有至少一个文件，文件就是对应FPakEntry）

然后从代码可以了解到，在FindFileInPakFiles方法中传入三个参数，一是读取的文件名字，二三是空的FPakEntry和FPakFile。这个方法返回一个bool值表明是否找到对应的文件，FileEntry被赋值为对应的文件。PakFile被赋值为包含这个FileEntry的pak文件。在找到这个具体的文件FileEntry后，返回一个异步读取的句柄。

这个问题不关心是如何异步读取的。关注点是FindFileInPakFiles这个方法。



在了解这个方法前必须先了解几点

1. pak文件格式

2. FPakPlatformFile、FPakFile、FPakEntry的代码结构。

3. FPakPlatformFile的初始化过程

4. 挂载Mount



**一、pak文件格式**

参考文章：<https://zhuanlan.zhihu.com/p/54531649>