---
layout: post
title: 手游开发的一些技巧
categories:
- technology
tags:
- 游戏 技巧

---

本文总结手游开发实践中的一些技巧，下面分为如何秒打iOS包、如何快速升级引擎以及如何修复常见Crash三个部分进行介绍。
## 1. 如何秒打iOS包
---

众所周知，iOS打包非常慢，打一个版本需要15~20分钟之久，每次打phone版、pad版两个包需要30~40分钟的漫长等待。有时经常还会出现，打出来的包有问题，需要重复打几次，这样就更坑了。据目前所知，G21、G16、G17都是这样，会浪费不少时间。

打包时间很长的原因是XCode点击Achive打包后，会将所有引擎和业务代码全部重新编译一遍，没有任何增量编译。

一种可行的解决办法是将引擎相关工程和代码移除出主工程，然后将引擎编译成静态链接库，加入主工程中。当脚本有更新时，直接更新XCode主工程的资源文件即可。引擎有更新时，直接增量编译引擎代码，速度也非常快，无需全部重新编译。这样也可以强迫你把引擎代码和业务代码区分开来。具体的步骤很简单！

（1）将cocos2d_libs和cocos2d_lua_binding两个工程和代码移除主工程；

（2）将上述两个工程的静态库输出路径更改为项目相对路径，这样引擎重新编译时，主工程无需进行任何设置；

（3）生成上述两个工程的Debug和Release版本静态库，以备调试和发布使用；

（4）在主工程中添加静态库的引用。

这样处理之后，G21就可以1分钟之内打出一个包来了。

## 2. 如何快捷升级引擎
---

因为工作室使用SVN进行源码的管理。SVN的一个问题是没有本地提交功能，有时是临时代码还不需要上传到服务器，如果代码未保存、或者删除就会浪费时间去重写，也不方便查找之前的代码思路，在这方面Git做的比较好，支持频繁地本地提交。SVN的另一个问题是分支功能不太好用，打一个Branch需要复制所有工程代码，非常麻烦。如果有一个正在研发中、或者目前还不稳定的功能，能方便地打出Branch就更好了，在这方面Git基于文件差异进行Branch做的更好。所以，一种可行的源码管理的方案是将SVN和Git结合，双源码管理器管理源码。远程使用SVN，用于稳定功能代码的备份和提交，本地使用Git，Git主要用于本地的代码提交和分支。
这样做的另外一个好处是升级引擎非常方便，可以经常根据引擎官方的修改进行引擎的升级，不用担心最新修改的bug或性能优化没有加入自己的引擎中。具体地：

（1）当项目使用Cocos2d-x 3.0版本时，我们从github上pull出Cocos2d-x 3.0的代码；

（2）当引擎需要修改时进行相应修改，提交SVN进行源码管理；

（3）当需要更新最新官方引擎时，Git更新最新的引擎代码进行合并；

那么已经修改过很多的引擎如何使用这种办法升级呢？一种可行办法是：Git同步最初的引擎版本，然后用目前的代码覆盖这份代码进行合并，最后根据需要同步更新后面较新的引擎版本。

## 3. 如何修复常见Crash

一般来说，线程安全和资源释放始终是Crash的罪魁祸首。

（1）iOS Crash

Android版本不Crash、iOS版本Crash，常见的一种思路是检查是否新分配了资源，分配了资源是否Retain和Release，否则可能会被系统回收掉。举个例子：

代码（Objective-c）：

{% highlight objective-c %}
@implementation XXHelper
static NSString *_token = @"";

+ (void) setToken:(NSString *)token { //早早地设置了token
    _token = token;
    [_token retain]; //看这里、看这里
}

+(void) registerTokenObtained { //Lua将会调用我
    if(![_token isEqualToString:@""]){
            [XXHelper sendTokenToLua];
    }
}

+ (void) sendTokenToLua { //我将token发送给Lua
    //我在这里调用Lua函数
    [_token release]; //看这里、看这里
}
@end
{% endhighlight %}

（2）Android Crash

iOS版本不Crash、Android版本Crash，非常常见的一种原因是Android的线程问题。如果使用C++ 11 thread库创建线程，线程中通过JNI对Java代码进行了调用，需要把C++创建的线程attach到JVM虚拟机上。具体地，可以加入如下两段代码：

代码一（C++）：

{% highlight cpp %}
 #if defined(ANDROID)
    JavaVM *vm; 
    JNIEnv *env; 
    vm = JniHelper::getJavaVM();

    JavaVMAttachArgs thread_args;
    thread_args.name = "XXXX"; 
    thread_args.version = JNI_VERSION_1_4; 
    thread_args.group = NULL;
    vm-> AttachCurrentThread (&env, &thread_args); 
 #endif
 {% endhighlight %}

代码二（C++）：

{% highlight cpp %}
 #if defined(ANDROID)
     vm-> DetachCurrentThread (); 
 #endif
 {% endhighlight %}

（3）Cocos2d-x Crash

有时iOS版本、Android版本都莫名地Crash了，一般是引擎和脚本调用有问题。最常见的错误也是线程安全问题和资源释放问题。

1）注意不要在非GL线程中调用openGL相关的指令和函数代码。Cocos2d-x 3.0之后对渲染引擎进行了重构，先遍历整个UI树，并将相关的绘制命令加入绘制队列中，而不是立即绘制。所以一些绘制之后的逻辑操作被包装成命令加入队列中。注意将多线程异步操作包装成命令加入队列，并且不要包含GL相关指令函数的调用。

2）在Lua中调用Cocos2d-x时，注意使用Retain和Release。Cocos2d-x的垃圾回收机制是每帧之后，释放引用计数为0的对象。在多帧中公用的对象注意Retain，适当时机Release，如RenderTexture等。