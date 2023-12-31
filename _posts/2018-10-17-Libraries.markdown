---
layout:     post
title:      "静态库和动态库的一些记录"
subtitle:   " \"静态库和动态库的一些讨论\""
date:       2018-10-17 17:00:00
author:     "QD"
header-img: "/img/in-post/Libraries/static_library.png"
catalog: true
tags:
    - 其它
---

> “😝”


今天同事遇到了一个动态库的问题，大家讨论了很久。很尴尬啊 遇到好多知识盲区啊 兄dei 默默的拿出我的小本本记录一下。

决定应用性能的最重要的因素包括2点
1.启动时间
2.运行内存中占用
尽可能减少可执行文件的大小和使用内存

大部分时间动态库都是优于静态库的


## 静态库
现在打成静态库的情况越来越少了。
都是用framework形式搞得
只要在设置的地方改成
![image.png](https://upload-images.jianshu.io/upload_images/1106106-604aa1d2a3f99743.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/1106106-242805653a513902.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当我们库文件打包生成 *静态链接器*，当主程序编译的时候，静态库会加载到主程序的源文件中。

但这里 静态库优于动态库的地方， *打出来ipa包 一般来说会小于动态库的。 因为主程序中相关的类会编译加载到主程序。*不相关的不会加载。

**坑**
`categoty`并不会加载。 要加载category 就要在`Other Linker Flags`中加入`-ObjC`来提示加入类目。如果只有类目的话，还是会加载不出来就要使用`-all_load`加载全部,或者`-force_load`强制加载某一个。


## 动态库


![image.png](https://upload-images.jianshu.io/upload_images/1106106-2ad4274e766d2dd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


和静态库的对比可以看出 最大的区别就是。
1 加载到可执行程序的是动态库的引用

在程序运行的时候，动态库会放在进程的内存地址中。
这样运行的时候，可执行程序的内存就会大大减小。

不过对于静态库的ipa包。动态库是全量放在ipa包中的。可能ipa包会大一点。

还要一个很重要的点，就是加载时间。
动态库不仅仅是在启动的时候加载。可以选择运行时加载。可以减少内存使用和启动时间。



## 引发讨论的问题
动态库在主程序打包的时候出现了`error: Bundle only contains bitcode-marker`
但是在模拟器build的时候没有问题

引发这个问题的原因是bitcode
1 Xcode在自己build的时候时候会使用`-fembed-bitcode-marker`Mode
2 在archive build和production build的时候需要使用`-fembed-bitcode`

我用通过run build生成的动态包都是使用`-fembed-bitcode-marker` 在打包的时候，明显不对。
![image.png](https://upload-images.jianshu.io/upload_images/1106106-22c3816c89a1cefa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决方案：
1 'Other C Flags'  to "-fembed-bitcode"
2 ` Build Settings` ` +` ` user-defined build setting` `BITCODE_GENERATION_MODE`,  Debug to marker, Release to bitcode

3 或者禁用bitcode


## 一些额外的命令

```
//查看信息
lipo -info xxx.framework/xxx 
// 合并库
lipo -create -output xxx xxx
```


