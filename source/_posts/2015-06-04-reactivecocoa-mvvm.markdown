---
layout: post
title: "基于ReactiveCocoa的MVVM探索"
date: 2015-06-04 15:49:37 +0800
comments: true
categories: 
---
## 前言

一直觉得对于具体功能模块的设计模式来说，没有什么值得研究和讨论的。觉得iOS提供的MVC模式已经能够很好的利用到产品的开发中去，对于大多数开发人员来说，最熟悉最得心应手的设计模式莫过于此。

但是随着业务需求的膨涨、页面交互的体验优化、业务模式的逐渐丰富，我们一直习以为常的MVC模式中，某些Controller的UI交互代码＋业务逻辑代码已经达到上千行的规模。当我们维护app、或进行新功能的扩充、或提供其他模块的调用接口时，复杂混乱的controller逻辑让我们头疼不已。

于是我们开始将Controlller的业务逻辑代码进行拆分，拆分部分逐渐形成了viewModel层，于是在现下移动客户端领域开始兴起了MVVM模式(V-VM-M)。随着FaceBook的“[React Native](https://github.com/facebook/react-native)“跨平台开发框架，以及之前就出现的[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)响应式编程模式的推出，MVVM设计模式在IOS移动端正逐渐被得心应手的开始使用。

本文在在学习了解React Native和ReactiveCocoa的基础上，对在IOS开发框架中如何采用MVVM模式进行了一定的探索和实践，并在实际的工程项目中如何利用MVC和MVVM做混合模式开发给出一定的建议，部分内容可能偏理论一点，不妥之处，望指教更正。

关于ReactiveCocoa和MVVM的学习资料：[点击这里-Chrome书签](https://www.google.com/stars/mzm33xlrlwxgw/profile/folio/ssf_68299747347db8bc?hl=zh-CN)

通过学习资料了解完ReactiveCocoa的基本概念之后，我们可以从下面一张图对ReactiveCocoa有一个整体理解：
{% img /images/FRP_ReactiveCocoa_large.png %}

核心概念是RACSignal，相当于一个信号管道，接收信号源的信号，将信号依次发送给订阅者，在RACSignal基础上我们在MVVM模式下用得比较多的是RACCommand，信号到来开始执行，每次执行产生一个回调Signal。 具体的理解可以参考学习资料，讲的很清楚。

目前最新版本的ReactiveCocoa开始支持Swift，要求最低发布版本为IOS8，所以我们可以使用支持IOS6的最高版本， 当接入ReactiveCocoa之后，在性能上会慢1～2倍，但不影响app的直接体验，另外在调试部分也需要程序员看懂信号的来源和出处，打断点进行调试。

```
platform :ios, '6.0'
inhibit_all_warnings!

workspace 'WTestReactiveCocoa'
xcodeproj 'TestReactiveCocoa'

target :TestReactiveCocoa do
    pod 'ReactiveCocoa', '~> 2.5'

    #设置pod target需要link的工程target
    link_with 'TestReactiveCocoa'
end


```


本文主要介绍通过ReactiveCocoa进行MVVM模式的开发实践，具体的应用模式如下：

{% img /images/MVVMReactiveCocoa.png %}


## 一、关于view层和viewModel层的Binding

对于某个业务功能（这里指某个controller），从代码集来看，肯定是一堆系统原生view组件和自定义view组件在controller.view中的聚集。所有的叶子view组件节点都是以controller.view为根节点，作为其subview存在于controller中。所以view层其实可以看作是以controller.view为根节点的树状结构，负责view的创建、布局、和viewModel的binding、以及页面的交互效果（包括动画、跳转、切换等），只涉及到页面交互处理，不涉及业务逻辑的处理。

而对于viewModel来说，一般和controller是一一对应的，view层可以强依赖viewModel层，而viewModel层一定不能引用view层的任何对象。 view层中涉及业务逻辑处理的任何交互均需要通过signal信号告知viewModel层或者直接调用viewModel的command命令执行，同时view层也可以监听viewModel中定义的signal或者command执行发出的signal完成页面交互逻辑的处理。


###具体处理过程

在controller初始化时，为该controller初始化一个对应的viewModel对象。这个viewModel可以只是个壳，viewModel中的具体属性对象可以通过前一个controller赋值初始化，也可以在当前controller中发出某个signal之后通过Model层提供的service进行初始化。

```
- (instancetype)init
{
    if (self = [super init]) {
        _fhcViewModel = [CPArenaFootHallContainerViewModel new];
        [self setupContent];
    }
    return self;
}

```


完成初始化之后，需要在viewDidLoad方法中在view组件创建之后，完成view层和viewModel层的binding。

```
- (void)viewDidLoad
{
    [super viewDidLoad];
    [self setupContent];
    [self bindViewModel];
}

/**
 * 将controller中的view和ViewModel进行binding
 */
-(void) bindViewModel{
    //监控初始化hallModel command的执行
    @weakify(self);
    [[self.fhcViewModel.loadHallModel execute:@(YES)] subscribeNext:^(id statusCode){
    	//to do something
    }];
 }

```

{% blockquote %}
tip: 

(1) 并不是所有的view组件都需要和viewModel绑定，通常如果view组件的显示属性是常量定义（如显示标题、某个业务组件的icon图片名称), 这些属性没有必要放入viewModel层，也就没有必要进行binding。

(2) 对于controller中跟viewModel中的对象息息相关的各个view对象（获取textField的输入值作为viewModel中某个operation的输入，点击某个按钮执行viewModel的某个command）, 则需要在controller中进行绑定。

{% endblockquote %}


不同的情景其binding的方法也不一样：

* 情景1:

如果subview是UIkit自带的view组件（如label，button, textfield, textview, imageView, alert, action sheet等等），则可以直接通过ReactiveCocoa直接进行view和viewModel之间的绑定。

ReactiveCocoa提供绑定category的的基本View组件包括：MKAnnotationView、UIActionSheet、UIAlertView、UIBarButtonItem、UIButton、UICollectionReusableView、UIControl、UIDatePicker、UIGestureRecognizer、UIImagePickerController、UIRefreshControl、UISegmentedControl、UISlider、UIStepper、UISwitch、UITableViewCell、UITableViewHeaderFooterView、UITextField、UITextView。

而我们平常经常用到的可能会有UITextField，UITextView、UIControl、UIButton、UIActionSheet、UIAlertView这几个，具体的使用方法在ReativeCoccoa的[示例](http://www.itiger.me/?p=38)中可以查看。其封装核心是监控view组件的delegate方法或者selector方法执行，每当执行时发送一个执行信号；view层将该信号和viewModel层进行绑定，自己也可以监控这些信号完成某些交互行为。

```
- (void)bindViewModel {
    self.title = self.viewModel.title;
    RAC(self.viewModel, searchText) = self.searchTextField.rac_textSignal;
    self.searchButton.rac_command = self.viewModel.executeSearch;
    RAC([UIApplication sharedApplication], networkActivityIndicatorVisible) = self.viewModel.executeSearch.executing;
    RAC(self.loadingIndicator, hidden) = [self.viewModel.executeSearch.executing not];
    
    [self.viewModel.executeSearch.executionSignals
     subscribeNext:^(id x) {
         [self.searchTextField resignFirstResponder];
     }];
}


```

* 情景2:

情景2主要是针对subview是通过单独文件封装的view组件，如UITableView的自定义tableviewCell，某些在基础view组件上进行封装的view组件等。如果是新写这些view组件，可以考虑通过ReactiveCocoa的方式完成；如果是已经成型的组件，则需要根据具体情况看是否改造成响应式模式；

（1）如果组件中只绑定一两个viewModel中非集合类型的属性值，则直接通过开放view组件的属性值，在controller中完成binding； 具体的binding方法和情景1类似。

（2）如果组件中有较多的基础view组件，且都和viewModel相关，则考虑为该view组件定义一个view对象，并作为view组件的属性。controller将view组件的viewModel属性和controller对应的viewModel相应属性绑定，当controller中对应的viewModel发生变化，将会把变化传递给view 组件，在view组件中根据viewModel的属性变化去改变具体的基础view组件显示值；
 
***这个跟Reative Native的props属性传递的思想类似：对于一些component，通过属性传入，在componnet中可以通过这些props去对基础UI组件赋值，当传入属性值改变之后，对应component的相应UI组件的显示值也会发生变化。***

在引入ReactiveCocoa之前，view组件和viewModel的对应如下所示：

```
- (void)setContent:(CPArenaMatchViewModel *)content
{
    if ([content.sps count] != 3 || [content.supportRates count] != 3) {
        return;
    }
    
    [self.winButton setHeaderText:content.teamAName];
    [self.winButton setDetailText:[NSString stringWithFormat:@"胜 %@",content.sps[0]]];
    [self.drawButton setHeaderText:@"平"];
}

```
在引入ReactiveCocoa之后，具体viewModel和view组件的binding过程则根情景1种的binding类似。



* 情景3:

对于比较复杂的系统view，如tableView、colletionView、pageView等，目前ReactiveCocoa中没有提供category进行绑定。

个人觉得对tableview整体进行封装没有必要，一方面对于app来说，大多数复杂的界面都是在tableview的基础上定制完成的，定制越多，封装就越没有意义；另外这些复杂的系统view组件一般提供了很好的扩展性和自刷新机制（ReloadData），如tableview的section、header view、footer view、tableviewcell等，这些扩展性就对应了情景2种提到的第二种情况。

对于这种情景的处理方法：我们可以通过传入一个viewModel对象给自定义的view或者cell，当页面刷新的时候，可以根据监控到的传入viewModel变化刷新界面；即使是非定制的view，则可以对view的属性直接通过controller对应的viewModel直接赋值；

截取了tableView:cellForRowAtIndexPath的绑定代码：

```

    if ([moduleKey isEqualToString:kArenaHallDaily]) {
        
        static NSString *dailyIdentifier = @"dailyCell";
        CPArenaHallDailyCell *cell = [tableView dequeueReusableCellWithIdentifier:dailyIdentifier];
        if (cell == nil) {
        	...
        }
        
        //直接读取viewModel中的值进行view组件的赋值，当tableview reload的时候，重新读取一次
        [cell setContent:[self.fhViewModel.hallModel arenaHallDailyContent]];
        return cell;
        
    } 

```


* 情景4:

而某些container-controller（tabController和NavigationController除外），在两个或者多个controller之间切换，controller.view中基本没有交互，交互主要在navigationBar上。NavigationBar的交互主要是controller跳转和切换，可能需要从Model中获取一些数据进行判定。

对于这种情景，我觉得没有必要通过MVVM模式进行处理，直接按照MVC模式处理即可，Model的获取通过Model层提供的APIService获取（单例），其跳转或者切换的controller中可以直接通过APIService获取之前已经初始化的Model对象；


###tip
	
{% blockquote %}

(1) 关于view的常量定义仍然属于View层，没有必要将其放进viewModel中（viewModel还只是保存view需要跟Model打交道，对其进行加工的部分）；

(2) 在view层，对于无交互且无数据刷新需求的view组件不进行绑定； 对于有交互的view组件，如果交互不会对viewModel进行改变，也无需绑定；对于一些需要等待viewModel初始化完成才能进行数据显示的view组件，则可以考虑统一监控viewModel，当viewModel有更新的时候统一刷新各个view组件；

(3) 一个controller拥有一个viewModel， 比较复杂的view组件也可以拥有一个viewModel，但其viewModel的初始化在controller的viewModel中完成；

{% endblockquote %}


##关于controller的简化和跳转

引入ReactiveCocoa有两个目的：

第一个:是在controller中一些交互比较频繁、状态逻辑较复杂的情况，可以通过ReactiveCocoa的Signal机制监听view组件的状态变化简化状态逻辑的判定。 比如说ReactiveCocoa的官方经典案例：用户名密码登录框； 

第二个目的就是本文主要阐述的方向，通过引入ReactiveCocoa简化controller， 将controller中的交互逻辑和业务逻辑分开，形成有效的MVVM开发模式。在将MVVM模式应用的具体的业务组件工程中可能会引出下面一些问题：

```
1. controller之间的跳转到底应该放在view层还是viewModel层？
2. 什么代码可以从controller中分离，如何分离？

```

###Controller间的跳转	
	
在传统MVC模式下的controller跳转，需要在当前controller中import另外一个controller头文件，然后传入参数初始化controller，再调用navigation-push 或者 viewController-present， 相当于controller跳转是放到了view层。传统的MVC模式给我们带来的痛苦是无法对controller层进行单元测试，并且controlle中业务逻辑和交互逻辑混乱不堪，所以我们才希望通过MVVC模式来简化controller中的代码；

当我们将业务逻辑拆分到viewModel中，那么controller的跳转是继续留在view层还是将其拆分到viewModel层呢？我们来看一下分别放在两个层的优缺点：

1. 如果继续放在view层：MVVM模式号称的去UI的测试也就无法去验证跳转逻辑，无法做完整性测试；并且每个controller的viewModel是孤立的，只能对其做单元测试；

2. 如果直接拆分放到viewModel层，它就背离了我们之前的分层标准，viewModel层绝对不能依赖view层的任何对象；如果我们通过viewModel层提供的service去做跳转，在MVC模式和MVVM模式混合开发的工程中，则稍显复杂



所以本文觉得还是把controller的跳转继续放在view层；相当于通过view层的navigator去完成跳转，其实这跟React Native的跳转也是不谋而合的（在react Native中，每一个controller都被当作是navigator下的一个scene，每个scene跳转的时候都会保留对全局navigator的引用，直接在view层的代码中调用navigator.push完成）。另外controller的跳转对于整个app来说，也是属于view交互的定性。




###交互逻辑和业务逻辑如何分离？

前文提到MVVM模式的主要工作就是将controller中的业务逻辑拆分出来，那在一个controller中如何定义业务逻辑，具体哪些代码需要拆分到viewModel中呢？前文也提到view层的主要工作是完成view的创建、布局、动画，以及交互和跳转处理。

view组件创建过程中，一些属性值需要在创建时通过参数传入，如果这些属性值是通过常量（如标题、按钮文字、alert提示文字、按钮的图片名称等），则不需要放到viewModel中；如果属性值是需要从Model中获取的数据（有些时候需要加工处理）进行初始化，则需要将其通过viewModel提供，一开始可以先传入一个空的viewModel，并建立view属性和viewModel的binding，当viewModel初始化完成或者更新的时候，通过signal更新view组件。另外在创建、布局view的时候可能需要一些model属性的判定，这些判断逻辑直接通过引用viewModel的数值即可。view层除开viewModel的绑定之外，只能读取viewModel中的值去初始化界面。


而交互逻辑和业务逻辑的主要交叉点出现在controller的生命周期和view组件的action操作中：

1. 比如说contorller的生命周期，当controller初始化并push之后，会先跳转到viewDidload中初始化subview，然后在viewWillAppear中完成布局，并向viewModel或者model层请求数据。 或者在在生命周期中需要监听某些事件通知、执行某些定时器等，这些逻辑代码就应该放到viewModel中。

2. 另外对于view组件的action操作，当touch事件发生之后，既要完成view的显示或者隐藏、或重新布局、或者更新数据，又需要向viewModel层发起某些操作，这些向viewModel发起的操作逻辑也必须放到viewModel中；

>
当我们把这些业务逻辑拆分出去之后，***如何在某些交互或者某个生命周期状态发生时自动调用对应的业务逻辑呢？业务逻辑在viewModel中又是如何组织的呢？***

业务逻辑在viewModel中通过RACCommand进行管理，当调起某个command的执行之后，首先完成业务处理，并通过signal（return value）通知调用者发起回调。

```
    //执行添加leaveTimer
    @weakify(self);
    self.addLeaveTimerCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
        @strongify(self);
        [self addLeaveTimer];
        return [RACSignal return:@(YES)];
    }];
    
```

如果业务处理是异步的，则需要在viewModel中定义全局Signal（RACSubject），当异步业务处理执行完成或者错误的时候，通过RACSubject通知调用者。

```
//初始化更新Signal，供view层监听
self.refreshResultSignal = [[RACSubject subject] setNameWithFormat:@"CPArenaFottHallVM-refreshResultSignal"];
    
    
/**
 * 重新刷新hallModel（hallModel其实应该是一个比较通用的ViewModel）
 */
-(void)refreshContent
{
    void (^ completion)(void) = ^{
        [self.refreshResultSignal sendCompleted];
    };
    
    //通过service重新load HallModel数据
    if(1){
        [self.refreshResultSignal sendNext:@"hello"];
        completion();
    }
}

    
```

生命周期状态的监听我们通过rac_signalforSelector去完成，某些view组件的action，则直接通过view组件的category，或者监听view组件的delegate回调来完成监听。也可以对这些信号进行组合、过滤、map、返回参数的封装，这些都是ReactiveCocoa提供的方法；

监听controller生命周期状态如下：

```
    //当view层disappear时，通知viewModel层开始计时
    [[self rac_signalForSelector:@selector(viewWillDisappear:)] subscribeNext:^(id x) {
        @strongify(self);
        [self.fhViewModel.addLeaveTimerCommand execute:nil];
    }];
    
    
    //监控view层的显示Signal
    [[self rac_signalForSelector:@selector(viewWillAppear:)] subscribeNext:^(id x) {
        @strongify(self);
        [self.fhViewModel.refreshCheckCommand execute:nil];
    }];


```
	
监听view组件的delegate回调如下：

```
	//监听当前HeadView的Accessory的click动作
    RACSignal *clickAccessorySignal = [self rac_signalForSelector:@selector(didClickAccessoryViewInHeaderView:) fromProtocol:@protocol(CPArenaHeaderViewDelegate)];
    [[clickAccessorySignal map:^id(RACTuple *tuple) {
        return tuple.first;
    }] subscribeNext:^(CPArenaHeaderView *headerView) {
    	//to do something
    }


```



###tip:
>
1. 将某些delegate方法通过signal监听之后，其实是可以删去这些delegate方法，但是编译器报警告，所以保留原有的delegate方法，只是方法实现为空；真正的方法实现放到Signal监听之中；



##viewModel层和Model层之间的交互

在Model层,ReactiveCocoa提供如下基础对象signal改造的支持：NSArray、NSData、NSDictionary、NSEnumerator、NSFileHandle、NSIndexSet、NSInvocation
NSNotificationCenter、NSObject、NSOrderedSet、NSSet、NSString、NSURLConnection、NSUserDefaults. 如果工程中的Model层也是通过响应式编程实现，这些category可能对Model层通知viewModel层有很大的作用，目前还没有对model层响应式编程仔细研究（这不是必要的），这里说一下Model层在MVVM模式的主要工作模式。


###APIService的发起路径和过程

在MVC模式下，我们通常直接在controller中创建Model层提供的APIService去获取数据。而在MVVM模式中，相应APIService的请求会转移到ViewModel层，而ViewModel的初始化是根据Model层的访问完成的，其具体访问路径如下：

（1）当点击某个按钮或者发起初始化signal，点击动作或者创建会向viewModel发出一个signal执行RACCommand（RACCommand，执行 一次产生一个signal， 这个command在viewModel层定义和创建），当前view层会定监听RACCommand的执行，并根据返回的event实时更新view；

（2）viewModel接受这个signal：
	
* 如果signal是创建ViewModel，则判断当前viewModel是否初始化，如果没有，则向Model层发起一个 initial operation；
	
* 如果signal改变了viewModel中的值，viewModel的变化会自动通知binding的view组件更新；此外如果viewModel的改变和Model层有关联，则向Model层发起一个update operation；

* 当前viewModel向Model发起operation之后，当前viewModel需要定制监控发起的这个signal，并将后台返回的signal信号进行处理，或更新viewModel，或向view层监控的signal发送各个operation的完成事件；


（3）Model层不仅仅是实体对象的定义，还包括APIService的实现，APIService通过向后台发送request请求，当请求返回的时候能够根据实体定义初始化实体对象，并在Model层持有（或进一步持久化，保存为coredata或者db中）；

* 当Model层接受到viewModel（MVVC）或者controller（MVC）的 initial operation时，查看Model层持有的对象，如果存在直接返回，如果不存在，则通过APIService向后台或者本地持久化发出请求，并将请求状态和结果push给ViewModel层或者controller层；


* 当Model层接收到update operation时，一般是先通过APIService发送update operation，根据operation的结果操作本地持久化对象（或者重新拉取一遍，或者直接在update operation中返回数据对原有持久对象进行更换）；


###关于Model和网络请求的生命周期
关于Model层什么时机进行初始化，Model层持有对象的生命周期，以及网络请求的生命周期？

* Model层的初始化，分为不同的场景，初始化的时机也不一样：

(1) 如果Model是全局需要的，如UserModel，则可以在appdelegate中直接初始化；

(2) 如果是MVC模式，则直接在controller中直接调用APIService初始化；

(3) 如果是MVVM模式，则通过调用viewModel中进行初始化；

* 生命周期：

(1) 如果是全局Model，则通过单例Service进行保持；

(2) 如果是跟controller保持一致的，则通过普通service进行初始化；

* 网络请求的生命周期：

(1) 如果是单例的service，网络请求的生命周期肯定小于单例，service持久化的Model对象也会一直存在；

(2) 如果是非单例service，service持久化的Model对象随着service的dealloc而销毁， 网络请求的生命周期也会被service cancel掉；仍然小于service；