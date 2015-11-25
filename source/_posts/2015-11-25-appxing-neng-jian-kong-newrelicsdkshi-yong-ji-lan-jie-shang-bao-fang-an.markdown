---
layout: post
title: "App性能监控-newRelicSDK使用及拦截上报方案"
date: 2015-11-25 09:38:34 +0800
comments: true
categories: App性能监控
---

随着应用的发展越来越大，产品的开发和测试逐渐开始关注app的在线运行性能。最近一直在关注这个方向，说一下自己的心得和体会。

所谓的性能监控，并不是做出一堆花哨的图表让人赏心悦目。其实目的只有两个，一个是能够实时发现线上的问题，通过报警机制通知到相关负责人；另外一个就是能够看到历史数据，供测试和开发进行性能优化。那么监控的关键是确定监控指标，在监控指标的指导下确定需要客户端打点收集的日志。

### APP性能监控指标

最近调研了一下这个方向，行业内把这个方向称为APM（application performance management）。国内外都有不少公司在做这方面的产品，一些大公司为了防止应用监控数据的外泄，也会自行开发性能监控管理平台。国内做得比较好的如基调、博睿、云测、听云等，国外有New Relic、Blueware、OneAPM等。 各个平台的技术方案大同小异（这里主要说的是IOS平台），提供一个集成到应用的SDK（static framework 或者 .a 文件）, 然后收集日志上报到其后台进行数据处理和纬度分析。这些平台的商业策略一般都是先免费提供试用，再进行租用收费的方式。

APP的性能指标主要分为两大类：InterActions（UI交互） 和 Network （网络连接）。在这两个大分类下，会有如下细分指标：

InterActions：(按时间-版本：＋device类型，＋os类型)

* APP启动次数、APP启动平均时间；
* Controller display次数、平均display时间；
* Controller display过程中各个method（包括UI thread、worker thread中的method）的平均执行时间；


Network方面：（按时间-版本：＋域名，＋地区，＋运营商， ＋连接网络类型）

* 平均相应时间
* rpm（一分钟请求次数）
* 总耗时
* 传输数据大小
* 再具体到某个API接口：

>
 (1) 平均http响应时间 （区分web Application（webview） 和 network）；
 (2) rpm；
 (3) 平均数据传输大小（区分send和receive）；
 
 在两大指标上会增加如下方面的维度： 接入网络、运营商、设备、系统版本、地区、app版本、统计时间；
  
### NewRelic SDK使用说明

我主要关注了国内的听云平台和国外的NewRelic平台。国内听云的统计平台注册只能看到非常基础的统计，一点细分分析维度，就提示你升级套餐，当然你可以申请14天免费试用，然后就是市场给你打电话过来。 其SDK中的framework也只提供一个头文件，想要对统计数据的定制基本上没有。于是果断切换到国外的newRelic平台，newRlic平台需要vpn访问才能快点，一注册所有的功能都能够试用，当然试用期也只有14天。 

关注一下SDK的使用，SDK提供了众多可配置的头文件，让你能够更加清楚的了解sdk提供的功能；

（1）最基础的使用，当然就是通过Apptoken注册使用：(注意一下，newRelic集成在debug时会检查当前应用中hook的使用，需要将newrelic的注册使用放到所有hook之后，一般放到appdidfinishLaunch靠后的位置)

```
#import <NewRelicAgent/NewRelic.h>

[NewRelicAgent startWithApplicationToken:@"token"];

```

如果是最基础的用法，我们可以通过charles拦截其数据上报，可以发现request和response的数据都是通过ssl加密、https传输，看不到明文数据。如果需要看到明文数据可以使用非加密接口```+ (void)startWithApplicationToken:(NSString*)appToken withoutSecurity:(BOOL)disableSSL```

(2) SDK集成了各种指标收集，包括Interaction、SwiftInteraction、Crash、NSURLSession、HTTPResponse等。如下代码所示：

```
typedef NS_OPTIONS(unsigned long long, NRMAFeatureFlags){
    NRFeatureFlag_InteractionTracing                    = 1 << 1,
    NRFeatureFlag_SwiftInteractionTracing               = 1 << 2, //disabled by default 
    NRFeatureFlag_CrashReporting                        = 1 << 3,
    NRFeatureFlag_NSURLSessionInstrumentation           = 1 << 4,
    NRFeatureFlag_HttpResponseBodyCapture               = 1 << 5,
    NRFeatureFlag_ExperimentalNetworkingInstrumentation = 1 << 13, //disabled by default
    NRFeatureFlag_AllFeatures                           = ~0ULL //in 32-bit land the alignment is 4bytes
};
```

你可以按照自己的需要指定需要收集的指标数据，使用```+ (void) enableFeatures:(NRMAFeatureFlags)featureFlags; 或 + (void) disableFeatures:(NRMAFeatureFlags)featureFlags;```.


(3) 关于Interaction的主要方案就是hook controller中的各个生命周期方法，NewRelicSDK中主要拦截了如下方法：

UIViewController	

1. viewDidLoad
2. viewWillAppear:
3. viewDidAppear:
4. viewWillDisappear:
5. viewDidDisappear:
6. viewWillLayoutSubviews
7. viewDidLayoutSubviews

UIImage:

1. imageNamed:
2. imageWithContentsOfFile:
3. imageWithData:
4. imageWithData:scale:
5. initWithContentsOfFile:
6. initWithData:
7. initWithData:scale:

NSJSONSerialization:

1. JSONObjectWithData:options:error:
2. JSONObjectWithStream:options:error:
3. dataWithJSONObject:options:error:
4. writeJSONObject:toStream:options:error:

NSManagedObjectContext	

1. executeFetchRequest:error:
2. processPendingChanges


(4)在ARC模式下，你也可以使用如下宏拦截你想统计的应用中关键方法：

```
- (void)myMethod
{
  NR_TRACE_METHOD_START(0);

  // … existing code

  NR_TRACE_METHOD_STOP;
}
```

更多的使用请关注其SDK文档说明：[https://docs.newrelic.com/docs/mobile-monitoring/mobile-sdk-api/new-relic-mobile-sdk-api/work-ios-sdk-api](https://docs.newrelic.com/docs/mobile-monitoring/mobile-sdk-api/new-relic-mobile-sdk-api/work-ios-sdk-api)


### newRelic SDK的数据拦截上报

newRelic SDK已经做得非常精致，但是如果我们不想把数据上报到第三方平台管理，那应该怎么办呢，自己去研发一套，耗时费力，而且还费力不讨好。那何不做一个数据拦截，把数据转发到我们自己的后台平台呢？ 

static framework public的头文件并不包括上传class的头文件，但是当网络不好或者没有网的时候，我们可以从控制台打印上报数据失败的日志看到一些端倪，我们可以找到上传日志的方法所在的类为“NRMAHarvesterConnection”，如果是ssl上传，日志为send方法，如果是非ssl上传，日志中显示为sendData方法；

为了更好的知道类中方法的分布，我们可以用class-dump-z工具解析出SDK中所有的头文件。 class-dump-z是不能直接dump framework中的头文件，解析出来的头文件中会有一个source 为null的提示。 一开始觉得可能是因为framework做了加密处理，找了一个越狱机器在cydia中安装了openssh-how-to工具，通过sftp把framework和Clutch上传到越狱机器中，开始用Clutch进行解密，发现Clutch并不能直接解析，也不能解析debug安装的调试app。

最后终于将集成了newRelic SDK的的app 通过archive打包，导出一个ipa。解压出app目录之后，再用class-dump-z工具对app目录的二进制文件进行头文件导出，这时候我们就能够看到sdk中的所有头文件了；

我找到“NRMAHarvesterConnection”文件，如下所示，可以看到其中的sendData方法。

```
#import <XXUnknownSuperclass.h> // Unknown library

@class NRMAConnectInformation, NSString;

@interface NRMAHarvesterConnection : XXUnknownSuperclass {
	BOOL _useSSL;
	NRMAConnectInformation* _connectionInformation;
	NSString* _collectorHost;
	NSString* _applicationToken;
	NSString* _crossProcessID;
	long long _serverTimestamp;
}
@property(assign) BOOL useSSL;
@property(retain) NSString* crossProcessID;
@property(assign) long long serverTimestamp;
@property(retain) NSString* applicationToken;
@property(retain) NSString* collectorHost;
@property(retain) NRMAConnectInformation* connectionInformation;
-(void).cxx_destruct;
-(id)collectorHostURL:(id)url;
-(id)collectorHostDataURL;
-(id)collectorConnectURL;
-(id)createDataPost:(id)post;
-(id)createConnectPost:(id)post;
-(id)sendData:(id)data;
-(id)sendConnect;
-(id)send:(id)send;
-(id)createPostWithURI:(id)uri message:(id)message;
-(id)init;
@end
```

为了更好的了解方法的输入和输出参数，我使用了阿里最近开源的ali-wax的lua调试工具进行了sendData的方法的输入输出了解, 我们可以看到输入参数data的class类型，是一个“NRMAHarvestData”，我们同样可以找到其头文件，查询到里面的JSONObject方法，可以把data转化成一个json对象。 得到这个对象了我们就可以将数据转发到我们自己的平台，并伪造一个http上传成功或失败的对象给sendData方法进行内存数据删除处理即可。

```
waxClass{"NRMAHarvesterConnection"}

--如果需要上报，拦截这个方法即可
function sendData(self,data)
  print(data:class())
  jsonObject = data:JSONObject()
  self:ORIGsendData(data)
end

```

注：这里我只提供一种方案，公司产品并没有采用这种方法，因为我们自己的性能监控虽然数据粒度不够，已经足够日常查询问题使用了。更多是分享一下取巧的思路，以及纪录一下破解的过程。