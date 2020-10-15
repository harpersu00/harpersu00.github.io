---
title: AFNetworking 3.1.0 第一部分
date: 2016-09-06
categories: 源码解析
---

## **写在前面**

我主要根据作者polobymulberry的博客[AFNetworking源码阅读系列](http://www.cnblogs.com/polobymulberry/category/785705.html)来学习，因为我有很多知识不懂，所以有些地方可能会记录得过于繁琐。原作者的大体流程和记录不变，某些知识点会有自己的补充。在此非常感谢博客作者polobymulberry的分享。

<!-- more -->

> 第一次运行运行example时总是出现 Module ‘AFNetworking’ not found 问题，查了好多关于Module的资料，对于问题的解决却没有帮助，后来偶然间在AFNetworking的github上的issues里找到解决办法，真是特别惭愧。应该点击文件夹内的`AFNetworking.xcworkspace`，而不是其他`.xcodeproj`文件。

图片1

运行成功后，就开始我们艰难的学习旅程吧！

## **开始**

### AppDelegate

此文件主要就是实现函数didFinishLaunchingWithOptions。将windows的rootViewController设置为rootViewController为GlobaltimelineViewController的NavigationController。此处有两点需要注意一下：

第一处

```
NSURLCache *URLCache = [[NSURLCache alloc] initWithMemoryCapacity:4 * 1024 * 1024 diskCapacity:20 * 1024 * 1024 diskPath:nil];
[NSURLCache setSharedURLCache:URLCache];
```

NSURLCache 为您的应用的 URL 请求提供了内存中（对应memoryCapacity）以及磁盘上（对应diskCapacity）的综合缓存机制。所以你想使用NSURLCache带来的好处，就需要在此处设置一个sharedURLCache。

第二处

```
[[AFNetworkActivityIndicatorManager sharedManager] setEnabled:YES];
```

为了说明AFNetworkingActivityIndicator是什么，直接上图：

图2

当你有session task正在运行时，这个小菊花就会转啊转。这个是自动检测的，只需要你设置AFNetworkingActivityIndicatorManager的sharedManager中的enabled设为YES即可。

这里我简单看了下AFNetworkingActivityIndicatorManager，发现它对外接口不多，比较容易理解它的业务流程。所以我准备在第三部分就将AFNetworkingActivityIndicatorManager的源码拿下。

设置完了cache和AFNetworkingActivityIndicator，接着就是进入GlobalTimelineViewController（UITableViewController）了。这里我学到一个，就是UITableViewController可以使用initWithStyle进行初始化。（**因为我对iOS界面不太了解，所以这个initWithStyle现在并不懂**）

> polobymulberry在开篇画了一个iOS Example的代码结构图，我不太懂MVC，特地查了下。以下是我个人非常粗浅的了解：
>
> - M代表Model，V代表View，C代表Cotroller。这是一种设计模式，是想让各模块分离，视图跟数据处理以及中间的控制协调端各司其职。
> - 视图只用于展现APP的界面，用于人和程序的交互。至于你点击按钮后产生的反馈，是由Controller来传递给Model处理，比如点击按钮后，会在文本框内展现文字。则Model从数据库中读取文字并通知Controller，事件已经处理完，Controller收到通知然后决定怎么处理，比如通过outlet控制View展示文字。
> - 注意View和Model之间并不直接通信。
>
>   图3
>
> 参考自[实际案例讲解iOS设计模式——MVC模式](http://blog.csdn.net/nhwslxf123/article/details/49703773)

### GlobalTimelineViewController

主要是围绕UITableView的delegate和dataSource来说。

##### 1\) UITableViewDelegate

主要是计算heightForRowAtIndexPath这个函数比较麻烦（应该是`-(CGFloat)tableView:heightForRowAtIndexPath:`函数），这里的Cell比较简单，可以直接使用posts中存储的text值来计算高度，核心代码就下面这句：

```
CGRect rectToFit = [text boundingRectWithSize:CGSizeMake(240.0f, CGFLOAT_MAX) options:NSStringDrawingUsesLineFragmentOrigin attributes:@{NSFontAttributeName: [UIFont systemFontOfSize:12.0f]} context:nil];
```

对于boundingRectWithSize的使用又增进了一步。（**这里我也不懂**）

##### 2\) UITableViewDataSource

主要是用posts作为数据源，而posts的获取在此处尤为关键，是通过Post本身（model）的globalTimelinePostsWithBlock函数获取数据的，这里作者将网络端的请求放在了model里面。

接着调用了refreshControl控件的setRefreshingWithStateOfTask:。setRefreshingWithStateOfTask:其实是UIRefreshControl+AFNetworking的一个category中定义的。UIRefreshControl+AFNetworking的源码很简单，放在第四部分讲。

注意setRefreshingWithStateOfTask:有一个参数就是NSURLSessionTask\*。而这个NSURLSessionTask的获取是调用了Post类中的globalTimelinePostsWithBlock:函数。

在globalTimelinePostsWithBlock:函数中其实封装了一层AFHTTPSessionManager的GET函数

```
- (nullable NSURLSessionDataTask *)GET:(NSString *)URLString
                            parameters:(nullable id)parameters
                              progress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                              success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                                failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
```

具体细节后面讨论，此处我们知道是根据一个url获取到服务器端的数据即可。注意获取到的数据是JSON格式的，这里作者在Post类，即Model中定义了一个JSON—->Model函数-initWithAttributes，，也就是说模型数据转化部分也放在了model中。

另外，调用GET方法不是直接用AFHTTPSessionManager的manager，而是又定义了一个AFAppDotNetAPIClient，继承自AFHTTPSessionManager。并在其定义的单例模式中简单地封装了一些AFHTTPSessionManager的设置。

```
+ (instancetype)sharedClient {
       static AFAppDotNetAPIClient *_sharedClient = nil;
       static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 初始化HTTP Client的base url，此处为@"https://api.app.net/"
        _sharedClient = [[AFAppDotNetAPIClient alloc] initWithBaseURL:[NSURL URLWithString:AFAppDotNetAPIBaseURLString]];
           // 设置HTTP Client的安全策略为AFSSLPinningModeNone
        _sharedClient.securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeNone];
});

    return _sharedClient;
}
```

知识点：SSL Pinning

Https对比Http已经很安全，但在建立安全链接的过程中，可能遭受中间人攻击。防御这种类型攻击的最直接方式是Client使用者能正确鉴定Server发的证书【目前很多浏览器在这方面做的足够好，用户只要不在遇到警告时还继续其中的危险操作】，而对于Client的开发者而言，一种方式保持一个可信的根证书颁发机构列表，确认可信的证书，警告或阻止不是可信根证书颁发机构颁发的证书。

SSL Pinning其实就是证书绑定，一般浏览器的做法是信任可信根证书颁发机构颁发的证书，但在移动端【非浏览器的桌面应用亦如此】，应用只和少数的几个Server有交互，所以可以做得更极致点，直接就在应用内保留需要使用的具体Server的证书。对于iOS开发者而言，如果使用AFNetwoking作为网络库，那么要做到这点就很方便，直接证书作为资源打包进去就好，AFNetworking会自动加载，具体代码就不贴了，nsscreencast已经有很好的tutorial。

至于model根据网络层获取的数据赋值，除了user的头像那块比较难，因为涉及到UIImageView+AFNetworking等文件，其他部分很简单。而AFNetworking的UIImageView+AFNetworking的部分其实很类似SDWebImage的思路。

##### Add：BLock作为函数参数

`block原本的形式为:`

`返回值 (^block名称 可省) (参数 可省) ＝ ^{函数体};`  
`调用形式： block名称(参数);`

在GlobalTimelineViewController.m `- (void)reload:` 函数中遇到了第一个block

```
NSURLSessionTask *task = [Post globalTimelinePostsWithBlock:^(NSArray *posts, NSError *error) {
        if (!error) {
            self.posts = posts;
            [self.tableView reloadData];
        }
    }];
```

`^(NSArray *posts, NSError *error){}` 整个block作为参数传递给`+ (NSURLSessionDataTask *)globalTimelinePostsWithBlock:`函数

> 参数传递只需要 \^\(参数\)\{block函数体\}
>
> \^表明是block形式

Post.m `+ (NSURLSessionDataTask *)globalTimelinePostsWithBlock:`函数:

```
+ (NSURLSessionDataTask *)globalTimelinePostsWithBlock:(void (^)(NSArray *posts, NSError *error))block {
                 ......
}
```

> 即这个函数的参数为\(void \(\^\)\(NSArray _posts, NSError_ error\)\)block
>
> \(返回值 \(\^\)\(参数\)\)block名称

也就是说在block作为函数参数传递时，定义block的这个A函数\(`globalTimelinePostsWithBlock:`\)并没有block函数体，而是调用A函数\(`[Post globalTimelinePostsWithBlock:]`\)在传参数时定义block的具体执行内容。
