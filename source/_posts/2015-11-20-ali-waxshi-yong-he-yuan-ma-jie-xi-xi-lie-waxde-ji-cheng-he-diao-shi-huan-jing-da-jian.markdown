---
layout: post
title: "Ali-Wax使用和源码解析系列-Wax的集成和调试环境搭建"
date: 2015-11-20 11:10:11 +0800
comments: true
categories: Ali-Wax
---

## 那些多余的话

在ios平台的基础框架中，代码的直接动态部署一直是一个无法解决的问题，所以我们只能退而求其次，通过建立脚本语言和OC的bridge来实现动态更新的目标。在这个方向上，wax-lua框架是第一个解决方案，但是随着wax-lua作者的放弃维护，wax框架逐渐被降级为做补丁修复的工具。之后随着objc语言的完善和SDK中JavaScriptCore（webkit）framework api的开放，另外一种脚本语言js以一种更为被苹果认可的姿态进入了这个方向，先有国外FaceBook发布的ReactNative框架，让更多的前端开发者可以通过js语言完成可以毗邻native效果的app，后有国内腾讯的bang开源JSPatch，专注于ios平台Patch修复。 

本人最近一直在维护公司内部的的wax框架，也一直在探索ios热更新方向上的新技术。先是wax框架的兼容64位维护，wax框架集成到产品后多个组件交叉使用runtime的bug修复；再是探索Tianium跨平台JS框架；再到FaceBook出品开源的ReactNative框架；再到最后的JSPatch... 这些框架的源码和Demo我都有涉及，从希望到失望...首先是自己维护的wax框架稳健性不够，特别是通过AOP+forward兼容64位后bug频现，用作patch热更新尚且能够应付，但是想用来做组件动态部署成熟度还不够。其次Tianium框架，关于jscore部分没有开源不敢用，另外其bind oc api部分需要时刻跟进新版本的SDK是一个low点。 再其次是ReactNative框架，源码我没有怎么看，但是我写了一个稍微复杂的页面，当图片在同一个页面加载较多的时候，卡顿的现象特别明显，让我对js的效率产生了深深的怀疑。 JSPatch同样是利用了forward消息转发机制完成了js和oc的bind，但是社区和框架成熟度仍然不够。

最近阿里开源了增强版的wax框架，我连续看了两天源码，对其增强部分的实现有了一个大致的了解，也让我对lua-bind的方案有了很大的信心。之前受Andfix android平台热更新框架思路的影响，自己很想在这条路上再继续折腾一番：如果能够把OC代码通过一个转换器转化成lua代码，ios平台的准直接动态部署应该是可以实现的。 所有的开发人员仍然用自己最熟悉的objc开发需求，测试和发版，比较版本代码的不同可以让低版本的产品用户使用高版本的功能。我们可以不按照版本发布，转而按照模块开发，所有的新需求和新运营方案都可以在不发版的情况推送给用户。

## Ali-Wax简介

代码地址：[https://github.com/alibaba/wax.git](https://github.com/alibaba/wax.git)

**wax-lua 的语言优势：**

* (1)自动垃圾回收：再也不用使用alloc，retain和release；
* (2)代码少：没有头文件，没有static类型、array常量、dictionary常量；
* (3)能够使用任何一个framework，例如cocoa、UITouch、Fondation等，任何用oc写的framework，wax自动将其暴露给lua使用；
* (4)超级简单的http请求： 和REST的web service一起交互使用；
* (5)lua也有函数闭包，也就是所谓的blocks；
* (6)lua有内置的Regex-like 模式匹配library；


**相比原Wax框架：**

* 64位支持；
* 线程安全；
* 其他一些特性：

>
 1. lua function 转化成 oc block，
 2. 在lua中调用oc block，
 3. getting/setting 私有成员变量，
 4. 内置通用的C函数，
 5. 支持lua代码debug；

在接下来的博客系列中，我将结合Ali—Wax源码，介绍如何实现这些增强特性的，敬请期待。下面我们先来看看如何使用和集成wax框架和其debug环境。

## Ali-Wax的Podfile集成

Wax框架只是一个热更新实现方案，我们用于具体的产品进行补丁修复或者功能组件发布还需要实现一个版本管理方案。在实现版本方案的时候，肯定涉及到在wax框架上的功能定制，比如说补丁现场恢复功能等（这部分功能我会开源到我的[fork代码](https://github.com/philonpang/wax.git)中）。另外我们现在用CocoaPods管理我们的代码组件版本，为了方便各自的产品修复bug，发放版本。 我建议大家fork一份代码到自己的github中，进行版本管理。

fork Ali-Wax: [https://github.com/philonpang/wax.git](https://github.com/philonpang/wax.git)

另外Ali-Wax框架中集成了lua的debug方案：mobdebug，位于wax/tools/mobdebug处，作者配置了Podspec，可以直接pod到你的调试工程中。 这部分代码也不会有什么变化，已建议作者单独列一个git工程，目前我放到了自己的git空间里，大家可以共享。

mobdebug: [https://github.com/philonpang/mobdebug.git](https://github.com/philonpang/mobdebug.git)

这样我们就可以在项目工程中通过Pofile直接集成wax框架和代码调试环境了(当然你也可以通过path指向wax clone项目的本地)；podifle如下：

```
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, '6.0'
inhibit_all_warnings!

workspace 'WLDWaxService'
xcodeproj 'LDWaxService'

target :LDWaxService do
    pod 'wax', :git => 'https://github.com/philonpang/wax.git'
    pod 'mobdebug', :git  => 'https://github.com/philonpang/mobdebug.git'
    link_with 'LDWaxService'
end

```

## Ali-Wax的luaDebug

作者通过编译luasocket源代码支持了lua代码的ios环境调试。作者刚发布的时候的调试环境也折腾了我很久，我这里说一下我踩过的那些坑和注意事项。 官方调试安装步骤见[https://github.com/alibaba/wax/tree/master/examples/LuaCodeDebug](https://github.com/alibaba/wax/tree/master/examples/LuaCodeDebug)

* download [ZeroBraneStudio](https://github.com/pkulchenko/ZeroBraneStudio) (lua代码调试IDE，直接git clone到本地)
* run ZeroBraneStudio: 进入ZeroBraneStudio/zbstudio文件夹，直接双击ZeroBraneStudio.app文件即可运行（将app拷贝到application中不能运行）
* import lua code: click the 6th button![Smaller icon](https://github.com/pkulchenko/ZeroBraneStudio/blob/master/zbstudio/res/24/DIR-SETUP.png?raw=true), choose your lua code's root directory
* start debug server: click Project->Start Debugger Server. （每次打开ZeroBraneStudio的时候，记得打开这个选项）
* run this code before you enter debug

```
    wax_start(nil, nil);//must start before debug
    extern void luaopen_mobdebug_scripts(void* L);
    luaopen_mobdebug_scripts(wax_currentLuaState());
```
* add```require('mobdebug').start('YOUR_MAC_IP_ADDRESS')```to your lua code. if you use simulator `'YOUR_MAC_IP_ADDRESS'` can be empty 
* launch your app，when `require('mobdebug').start()` is invoked, ZeroBraneStudio's dock will become active, then you should add breakpoint. (这种情况下，当你在模拟器中运行app后，会自动跳在ZeroBraneStudio中启动lua文件中断点下来)


### 坑1:import luaCode 的tip

官方截图上看到的截图是整个xcode工程的目录，**如果你import了整个工程，在目录和文件较多的情况下，ZeroBraneStudio中的断点是无效的**。
其实你只需要import存放lua文件的目录即可。

### 坑2:xcode工程中的lua代码文件

xcode添加文件有两种方式：Create groups 和 Create folder reference两种。 按照前一种方式，你引入目录的所有文件在运行时会被拷贝到“XX.app/”目录下，而后面一种方式目录会以引用方式加入，运行时文件仍然放在“XX.app/拷贝目录/”下，而且当引入目录中有新文件添加的时候，不用再执行一遍添加操作。

所以我们在本地调试代码的时候，建议以第二种方式引入目录，这样我们可以随时在ZeroBraneStudio中添加文件（而不用在xcode工程中再进行一次add操作）进行调试。 但是执行lua代码中的require代码行时，会提示找不到对应的require文件。这其实是lua 默认search 文件的路径没有设置照成的，所以我们需要设置lua的search路径环境变量。 代码如下：

```
#import "wax.h"
#import "wax_http.h"
#import "wax_json.h"
#import "wax_xml.h"


    //设置lua的search路径环境变量
    NSString *patchPath = [[NSBundle mainBundle] pathForResource:@"patchDemo" ofType:nil];
    NSString *pp = [NSString stringWithFormat:@"%@/?.lua;", patchPath];
    setenv(LUA_PATH, [pp UTF8String], 1);

    //启动wax框架和debug环境
    wax_start(nil, luaopen_wax_http, luaopen_wax_json, luaopen_wax_xml, nil);//must start
    extern void luaopen_mobdebug_scripts(void* L);
    void * p = wax_currentLuaState();
    luaopen_mobdebug_scripts(p);

    //运行启动文件
    wax_runLuaString("require('patch')");

```

完成以上步骤之后，你就可以在你的项目工程中开心的调试lua代码了。如下图所示：

{% img /images/luadebugdemo.jpg %}















