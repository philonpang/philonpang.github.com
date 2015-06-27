---
layout: post
title: "IOS平台Hotfix框架:JSPatch学习笔记及和WaxPatch的集成"
date: 2015-06-26 14:59:32 +0800
comments: true
categories: 
---

## 关于HotfixPatch

在IOS开发领域，由于Apple严格的审核标准和低效率，IOS应用的发版速度极慢，稍微大型的app发版基本上都在一个月以上，所以代码热更新（HotfixPatch）对于IOS应用来说就显得尤其重要。

现在业内基本上都在使用WaxPatch方案，由于Wax框架已经停止维护四五年了，所以waxPatch在使用过程中还是存在不少坑(比如参数转化过程中的问题，如果继承类没有实例化修改继承类的方法无效, wax_gc中对oc中instance的持有延迟释放...)。另外苹果对于Wax使用的态度也处于模糊状态，这也是一个潜在的使用风险。

随着FaceBook开源React Native框架，利用JavaScriptCore.framework直接建立JavaScript（JS）和Objective-C(OC)之间的bridge成为可能，JSPatch也在这个时候应运而生。最开始是从唐巧的微信公众号推送上了解到，开始还以为是在React Native的基础上进行的封装，不过最近仔细研究了源代码，跟React Native半毛钱关系都没有，这里先对JSPatch的作者（不是唐巧，是Bang，[博客地址](http://blog.cnbang.net/)）赞一个。

深入了解JSPatch之后，第一感觉是这个方案小巧，易懂，维护成本低，直接通过OC代码去调用runtime的API，作为一个IOS开发者，很快就能看明白，不用花大精力去了解学习lua。另外在建立JS和OC的Bridge时，作者很巧妙的利用JS和OC两种语言的消息转发机制做了很优雅的实现，稍显不足的是JSPatch只能支持ios7及以上。

由于现在公司的部分应用还在支持ios6，完全取代Wax也不现实，但是一些新上应用已经直接开始支持ios7。个人觉得ios6和ios7的界面风格差别较大，相信应用最低支持版本会很快升级到ios7. 还考虑到JSPatch的成熟度不够，所以决定把JSPatch和WaxPatch结合在一起，相互补充进行使用。下面给大家说一些学习使用体会。


## JSPatch和WaxPatch对比

关于JSPatch对比WaxPatch的优势，下面摘抄一下JSPatch作者的话：

* [来源: JSPatch – 动态更新iOS APP](http://blog.cnbang.net/works/2767/)

#### 方案对比
目前已经有一些方案可以实现动态打补丁，例如WaxPatch，可以用Lua调用OC方法，相对于WaxPatch，JSPatch的优势：

* 1.**JS语言:** JS比Lua在应用开发领域有更广泛的应用，目前前端开发和终端开发有融合的趋势，作为扩展的脚本语言，JS是不二之选。

* 2.**符合Apple规则:** JSPatch更符合Apple的规则。[iOS Developer Program License Agreement](https://developer.apple.com/programs/terms/ios/standard/ios_program_standard_agreement_20140909.pdf)里3.3.2提到不可动态下发可执行代码，但通过苹果JavaScriptCore.framework或WebKit执行的代码除外，JS正是通过JavaScriptCore.framework执行的。

* 3.**小巧:** 使用系统内置的JavaScriptCore.framework，无需内嵌脚本引擎，体积小巧。

* 4.**支持block:** wax在几年前就停止了开发和维护，不支持Objective-C里block跟Lua程序的互传，虽然一些第三方已经实现block，但使用时参数上也有比较多的限制。


JSPatch的劣势：

* 相对于WaxPatch，JSPatch劣势在于不支持iOS6，因为需要引入JavaScriptCore.framework。另外目前内存的使用上会高于wax，持续改进中。


## JSPatch的实现原理理解

JSPatch的实现原理作者的博文已经很详细的介绍了，我这里就不多说了，贴一下学习之处：

* JSPatch实现原理详解 [http://blog.cnbang.net/tech/2808/](http://blog.cnbang.net/tech/2808/)
* JSPatch Git源码和使用说明 [https://github.com/bang590/JSPatch](https://github.com/bang590/JSPatch)


看实现原理详解的时候对照着源码看，比较好理解，我在这里说一下我对JSPatch的学习和理解：

#### （1）OC的动态语言特性

不管是WaxPatch框架还是JSPatch的方案，其根本原理都是利用OC的动态语言特性去动态修改类的方法实现。
OC的动态语言特性是在runtime system(全部用C实现，Apple维护了一份开源代码)上实现的，面向对象的Class和instance机制都是基于消息机制。我们平时认为的[object method]，正确的理解应该是[receiver sendMsg], 所有的消息发送会在编译阶段编译为runtime c函数的调用：_obj_sendMsg(id, SEL). 

详细介绍参考博文：

* [Objective-C Runtime详细介绍](http://justsee.iteye.com/blog/2163777)
* [Objective-C Runtime源码_Apple](http://www.opensource.apple.com/source/objc4/)

runtime提供了一些运行时的API

* 反射类和选择器

```
	Class class = NSClassFromString("UIViewController");
	SEL selector = NSSelectorFromString("viewDidLoad");
```

* 为某个类新增或者替换方法选择器（SEL）的实现（IMP）   

```
	BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
	IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types);
```

* 在runtime中动态注册类

```
	Class superCls = NSClassFromString(superClassName);
	cls = objc_allocateClassPair(superCls, className.UTF8String, 0);
	objc_registerClassPair(cls);
```


#### （2）JS如何调用OC

在JS运行环境中，需要解决两个问题，一个是OC类对象（objc_class）的获取，另一个就是使用对象提供的接口方法。 

对于第一个问题，JSPatch在实现中是通过Require调用在JS环境下创建一个class同名对象（js形式），当向OC发送alloc接收消息之后，会将OC环境中创建的对象地址保存到这个这个js同名对象中，js本身并不完成任何对象的初始化。关于JS持有OC对象的引用，其回收的解释在JSPatch作者的博文中有介绍，没有具体测试。详见JSPatch.js代码：

```
	//请求OC类对象
	UIView = require("UIView");
	
	//缓存JS class同名对象
	var _require = function(clsName) {
	    if (!global[clsName]) {
	      global[clsName] = {
	        __isCls: 1,
	        __clsName: clsName
	      }
	    } 
	    return global[clsName]
  	}

	//调用class方法，返回OC实例化对象进行封装
	var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
                         _OC_callC(clsName, selectorName, args)
                         
    //OC创建后返回对象
    return@{@"__clsName": NSStringFromClass([obj class]), @"__obj": obj};
        

    //JS中解析OC对象
    return _formatOCToJS(ret)
    
    //_formatOCToJS
    if (obj instanceof Object) {
        var ret = {}
        for (var key in obj) {
          ret[key] = _formatOCToJS(obj[key])
        }
        return ret
     }

	
```

对于第二个问题，JSPatch在JS环境中通过中心转发方式，所有OC方法的调用均是通过新增Object（js）原型方法_c(methodName)完成调用，在通过JavaScriptCore执行JS脚本之前，先将所有的方法调用字符替换
_c('method')的方式； 在_c函数中通过JSContex建立的桥接函数传入参数和返回参数即完成了调用；

```
	//字符替换
	static NSString *_regexStr = @"\\.\\s*(\\w+)\\s*\\(";
	static NSString *_replaceStr = @".__c(\"$1\")(";
	
	NSString *formatedScript = [NSString stringWithFormat:@"try{@}catch(e){_OC_catch(e.message, e.stack)}", [_regex stringByReplacingMatchesInString:script options:0 range:NSMakeRange(0, script.length) withTemplate:_replaceStr]];
	
	
	//__c()向OC转发调用参数
	Object.prototype.__c = function(methodName) {
	    
	    ...
	    
	    return function(){
	      var args = Array.prototype.slice.call(arguments)
	      return _methodFunc(self.__obj, self.__clsName, methodName, args, self.__isSuper)
	    }
	 }
	  
	//_methodFunc调用桥接函数
	var _methodFunc = function(instance, clsName, methodName, args, isSuper) {
		
		...
		
	    var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
	                         _OC_callC(clsName, selectorName, args)
	
	    return _formatOCToJS(ret)
	 }


	//OC中的桥接函数，JS和OC的桥接函数都是通过这样定义
	context[@"_OC_callI"] = ^id(JSValue *obj, NSString *selectorName, JSValue *arguments, BOOL isSuper) {
        return callSelector(nil, selectorName, arguments, obj, isSuper);
    };
    
    context[@"_OC_callC"] = ^id(NSString *className, NSString *selectorName, JSValue *arguments) {
        return callSelector(className, selectorName, arguments, nil, NO);
    };

```

#### （3）JS如何替换OC方法

JSPatch的主要作用还是通过脚本修复一些线上bug，希望能够达到替换OC方法的目标。JSPatch的实现巧妙之处在于：利用了OC的[消息转发机制](http://bugly.qq.com/blog/?p=64)。

* 1:替换原有selector的IMP实现为一个空的IMP实现，这样当objc_class接受到消息之后，就会进行消息转发, 另外需要将selector的初始实现进行保存；

```
	//selector指向空实现
	IMP msgForwardIMP = getEmptyMsgForwardIMP(typeDescription, methodSignature);
    class_replaceMethod(cls, selector, msgForwardIMP, typeDescription);


	//保存原有实现，这里进行了修改，增加了恢复现场的支持
	NSString *originalSelectorName = [NSString stringWithFormat:@"ORIG@", selectorName];
    SEL originalSelector = NSSelectorFromString(originalSelectorName);
    if(class_respondsToSelector(cls, selector)) {
        if(!class_respondsToSelector(cls, originalSelector)){
            class_addMethod(cls, originalSelector, originalImp, typeDescription);
        } else {
            class_replaceMethod(cls, originalSelector, originalImp, typeDescription);
        }
    }

```


* 2:将替换的JS方法构造一个JPSelector及其IMP实现（根据返回参数构造），添加到当前class中，并通过cls＋selecotr全局缓存JS方法（全局缓存并没有多大用途，但是对于后面恢复现场比较有用）;

```
	if (!_JSOverideMethods[clsName][JPSelectorName]) {
        _initJPOverideMethods(clsName);
        _JSOverideMethods[clsName][JPSelectorName] = function;
        const char *returnType = [methodSignature methodReturnType];
        IMP JPImplementation = NULL;
        
        //根据返回类型构造
        switch (returnType[0]){
         ...
        }
     
		if(!class_respondsToSelector(cls, JPSelector)){
            class_addMethod(cls, JPSelector, JPImplementation, typeDescription);
        } else {
            class_replaceMethod(cls, JPSelector, JPImplementation,typeDescription);
        }
    }

```

* 3:然后改写每个替换方法类的forwadInvocation的实现进行拦截，如果拦截到的Invocation的selctor转化成JPSelector能够响应，说明是一个替换方法，则从Invocation中取参数后调用JPSelector的IMP；

```
	static void JPForwardInvocation(id slf, SEL selector, NSInvocation *invocation)
	{
	    NSMethodSignature *methodSignature = [invocation methodSignature];
	    NSInteger numberOfArguments = [methodSignature numberOfArguments];
	    
	    NSString *selectorName = NSStringFromSelector(invocation.selector);
	    NSString *JPSelectorName = [NSString stringWithFormat:@"_JP@", selectorName];
	    SEL JPSelector = NSSelectorFromString(JPSelectorName);
	    
	    if (!class_respondsToSelector(object_getClass(slf), JPSelector)) {
	    	...
	    }
	    
	    NSMutableArray *argList = [[NSMutableArray alloc] init];
	    [argList addObject:slf];
    
	    for (NSUInteger i = 2; i < numberOfArguments; i++) {
	    	...
	    }
	    
	    //获取参数之后invoke JPSector调用JSFunction的实现
	    @synchronized(_context) {
	        _TMPInvocationArguments = formatOCToJSList(argList);
	
	        [invocation setSelector:JPSelector];
	        [invocation invoke];
	        
	        _TMPInvocationArguments = nil;
	    }
	}

```

## Patch现场复原的补充
 
Patch现场恢复的功能主要用于连续更新脚本的应用场景。由于IOS的App应用按Home键或者被电话中断的时候，应用实际上是首先进入到后台运行阶段（applicationWillResignActive），当我们下次再次使用App的时候，如果后台应用没有被终止（applicationWillTerminate），那么App不会走appliation:didFinishLaunchingWithOptions方法，而是会走（applicationWillEnterForeground）。 对于这种场景如果我们连续更新线上脚本，那么第二次脚本更新则无法保留最开始的方法实现，另外恢复现场功能也有助于我们撤销线上脚本能够恢复应用的本身代码功能。

#### （1）WaxPatch的现场恢复

在Wax中通过wax_start("init lua file")启动框架并通过脚本替换类方法，通过wax_end()停止使用waxPatch环境。 Wax的官方维护版本中也没有现场恢复功能，替换方法的现场保存也是我们维护团队自行加入的，其实就是在替换方法或者新增方法的时候对原始现场（替换类／新增类＋替换方法/新增方法）进行cache，当需要恢复patch现场的时候通过cache进行复原。 下面简单说一下wax中的现场保存和恢复功能：

首先是在方法替换的时候，记录替换类＋替换方法

```
	static BOOL overrideMethod(lua_State *L, wax_instance_userdata *instanceUserdata){
		...
		
		if(instImp) {
            if(!class_respondsToSelector(klass, newSelector)) {
                class_addMethod(klass, newSelector, prevImp, typeDescription);
            } else {
                class_replaceMethod(klass, newSelector, prevImp, typeDescription);
            }
            success = YES;
            
            NSDictionary *dict = @{@"class" : klass,
                                   @"sel" : selector ? NSStringFromSelector(selector) : [NSNull null],
                                   @"sel_objc" : newSelector ? NSStringFromSelector(newSelector) : [NSNull null], // objcXXXX
                                   @"typeDesc" : typeDescription ? [NSString stringWithUTF8String:typeDescription] : [NSNull null],
                                   @"identifier" : identifier
                                   };
            addMethodReplaceDict(dict);
        } 
		
		...
	}
		
```

当Wax_end调用的时候恢复现场：

```
	//调用wax_end
	void wax_end() {
	    wax_clear();
	    [wax_gc stop];
	    lua_close(wax_currentLuaState());
	    currentL = 0;
	}
	
	//wax_clear()恢复现场

	/// 重置所有被wax修改的方法和类
	void wax_clear() {
	    // methods rollback
	    for (NSDictionary *dict in replacedMethodArray) {
	        Class class = dict[@"class"];
	        NSString *sel_str = dict[@"sel"];
	        NSString *sel_objc_str = dict[@"sel_objc"];
	        NSString *typeDesc = dict[@"typeDesc"];
	        NSString *identifier = dict[@"identifier"];
	        if (identifier) {
	            [[LDAOPAspect instance] removeAnInterceptorWithIdentifier:identifier];
	        }
	        
	        if (sel_str && ![sel_str isKindOfClass:[NSNull class]]
	            && sel_objc_str && ![sel_objc_str isKindOfClass:[NSNull class]]
	            && typeDesc && ![typeDesc isKindOfClass:[NSNull class]]) {
	            SEL sel = NSSelectorFromString(sel_str);
	            SEL sel_objc = NSSelectorFromString(sel_objc_str); // objcXXXX
	            IMP imp = class_getMethodImplementation(class, sel_objc);
	            class_replaceMethod(class, sel, imp, typeDesc.UTF8String);
	        }
	    }
	    
	    [replacedMethodArray removeAllObjects];
	    [replacedMethodArray release];
	    replacedMethodArray = nil;
	    
	    // class rollback
	    for (NSDictionary *dict in modifiedClassArray) {
	        NSString *className = dict[@"class"];
	        NSInteger version = [dict[@"version"] intValue];
	        Class class = NSClassFromString(className);
	        class_setVersion(class, version);
	    }
	    [modifiedClassArray removeAllObjects];
	    [modifiedClassArray release];
	    modifiedClassArray = nil;
	}

```

#### （2）JSPatch的现场恢复

为了跟Wax配合使用，本文在JSPatch添加了一样的调用方式和现场恢复功能；源码地址参考：

* 增加现场恢复的JSPatchDemo:[https://github.com/philonpang/JSPatch.git](https://github.com/philonpang/JSPatch.git)

说明如下：

（1）在JPEngine.h 中添加了两个启动和结束的调用函数如下：

```
	void js_start(NSString* initScript);
	void js_end();
```

(2) JPEngine.m 中调用函数的实现以及恢复现场对部分代码的修改：主要是利用了替换方法和新增方法的cache（_JSOverideMethods, 主要是这个）

```
	//处理替换方法,selector指回最初的IMP，JPSelector和ORIGSelector都指向未实现IMP
     if([JPSelectorName hasPrefix:@"_JP"]){
         if (class_getMethodImplementation(cls, @selector(forwardInvocation:)) == (IMP)JPForwardInvocation) {
             SEL ORIGforwardSelector = @selector(ORIGforwardInvocation:);
             IMP ORIGforwardImp = class_getMethodImplementation(cls, ORIGforwardSelector);
             class_replaceMethod(cls, @selector(forwardInvocation:), ORIGforwardImp, "v@:@");
             class_replaceMethod(cls, ORIGforwardSelector, _objc_msgForward, "v@:@");
         }
         
         
         NSString *selectorName = [JPSelectorName stringByReplacingOccurrencesOfString:@"_JP" withString:@""];
         NSString *ORIGSelectorName = [JPSelectorName stringByReplacingOccurrencesOfString:@"_JP" withString:@"ORIG"];
         
         SEL JPSelector = NSSelectorFromString(JPSelectorName);
         SEL selector = NSSelectorFromString(selectorName);
         SEL ORIGSelector = NSSelectorFromString(ORIGSelectorName);
         
         if(class_respondsToSelector(cls, ORIGSelector) &&
            class_respondsToSelector(cls, selector) &&
            class_respondsToSelector(cls, JPSelector)){
             NSMethodSignature *methodSignature = [cls instanceMethodSignatureForSelector:ORIGSelector];
             Method method = class_getInstanceMethod(cls, ORIGSelector);
             char *typeDescription = (char *)method_getTypeEncoding(method);
             IMP forwardEmptyIMP = getEmptyMsgForwardIMP(typeDescription, methodSignature);
             IMP ORIGSelectorImp = class_getMethodImplementation(cls, ORIGSelector);
             
             class_replaceMethod(cls, selector, ORIGSelectorImp, typeDescription);
             class_replaceMethod(cls, JPSelector, forwardEmptyIMP, typeDescription);
             class_replaceMethod(cls, ORIGSelector, forwardEmptyIMP, typeDescription);
         }
     }
     
     //处理添加的新方法
     else {
         isClsNew = YES;
         SEL JPSelector = NSSelectorFromString(JPSelectorName);
         if(class_respondsToSelector(cls, JPSelector)){
             NSMethodSignature *methodSignature = [cls instanceMethodSignatureForSelector:JPSelector];
             Method method = class_getInstanceMethod(cls, JPSelector);
             char *typeDescription = (char *)method_getTypeEncoding(method);
             IMP forwardEmptyIMP = getEmptyMsgForwardIMP(typeDescription, methodSignature);
             
             class_replaceMethod(cls, JPSelector, forwardEmptyIMP, typeDescription);
         }
     }
```


## HotfixPatch的那些坑

WaxPatch之前被一些同事抱怨有不少坑，JSPatch在使用过程中也会遇到不少坑，所以虽然这两个框架现在虽然都能够做到新增可执行代码，但是将其应用到开发功能组件还不太可取。 

比如说我在第一次使用JSPatch遇到了一个坑：（后面想单写一个博客收集一下我们团队使用Patch遇到的坑～～）

* 在JS脚本改写派生类中未实现的继承类的 optional protocol方法时，tableView reload的时候不会调用JS的补丁方法，但是在tableView中显式调用可以调用替换的selector方法；另外如果在派生类中重写这个protocol方法，则可以调起；

* ...

先写这么多了，本来想写一下我们的patch管理方案，觉得没有什么可说了，就不写了～ 







