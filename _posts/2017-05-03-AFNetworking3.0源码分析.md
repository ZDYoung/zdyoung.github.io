---
layout:     post
title:      AFNetworking3.0源码解读之数据请求
date:       2017-05-03 
summary:    本文主要分析AFNetworking的数据请求部分。
categories: objective-c AFNetworking
---

标签： objective-c AFNetworking

--- 

### 前言

无论做什么开发，都避免不了和网络打交道。在iOS中，网络请求这一块，大家肯定用过AFNetworking库。那么它到底是怎么实现的呢？本文主要讲它的数据请求部分，在以后博客中，会慢慢讲解其他部分。网上其实已经有很多相关的博客了，有些写的更好，写这篇博客的原因也是为了方便记下自己在看一些开源库的代码时学习到的编程技法、编程思想和问题的解决方案。

先从AFURLSessionManager开始，可以说它是整个库的核心，后面的AFHTTPSessionManager是就继承自AFURLSessionManager类。AFURLSessionManager类主要是对NSURLSession还有相关代理协议的封装。这个类对用户暴露的接口很少，在实现文件里定义了大量的私有方法，此外还对AFURLSessionManager进行了扩展。进行入AFURLSessionManager.m文件内看可以看到这样的结构：
![AFURLSessionManager类结构图](/images/AFURLSessionManager类的结构图.png)

在实现文件里，AFURLSessionManager.m里定义了两个类，一个类是AFURLSessionManagerTaskDelegate，还有一个就是_AFURLSessionTaskSwizzling。

AFURLSessionManagerTaskDelegate这个类从名字上看就知道，它是用来实现代理的，所有的网络请求回调都是经过它。呆会通过一个实际的例子来的说明。

那么_AFURLSessionTaskSwizzling类是用来干吗的呢？在写这篇博客的时候，笔者还没有能过例子实际的使用过，不过它的源码很少，通过看原码和注释大概知道，它是用来进行方法调配，使用了系统的运行期系统来交换两个方法的实现。注释中提到NSURLSessionTask的实现是通过class cluster来实现的，也就是说我们创建的一个NSURLSessionTask对象，并不是真正的创建这个对象而是创建了一个名字叫__NSCFLocalDataTask。这个类只干了一件了，那就是对任务的`resume`和`suspend`进行了交换，目的可能就是要实现任务的暂停和恢复的功能。对这个类先说这么点点个人理解吧。

接下来，笔者就以官方给出的例子跟踪一下，它的调用过程以下是源码：
```objc
NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
AFURLSessionManager *manager = [[AFURLSessionManager alloc] initWithSessionConfiguration:configuration];

manager.responseSerializer = [AFHTTPResponseSerializer serializer];
NSURL *URL = [NSURL URLWithString:@"http://www.baidu.com"];
NSURLRequest *request = [NSURLRequest requestWithURL:URL];

NSURLSessionDataTask *dataTask = [manager dataTaskWithRequest:request completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
    } else {
        
        NSString *result = [[NSString alloc] initWithData:responseObject  encoding:NSUTF8StringEncoding];
        NSLog(@"%@",[responseObject class]);
        NSLog(@"%@",result);
    }
}];
[dataTask resume];
```
该代码核心只有两句一个是manager的初始化还有一个就是dataTask的创建。先来看看初始化：
```objc
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;//默认配置

    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;//最大并发数，此处应该当成串行队列来用了

    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];//创建session，设置代理对象

    self.responseSerializer = [AFJSONResponseSerializer serializer];//默认的解析器

    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif

    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidResume:) name:AFNSURLSessionTaskDidResumeNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidSuspend:) name:AFNSURLSessionTaskDidSuspendNotification object:nil];

    return self;
}
```
看起来好多，对我们本次的目的其实需要关心的东西很少，我都给出来了注释。

第二处是dataTask的创建，为了说明，先把程序跑起来看看函数的调用栈。任务一启动，首先进入到了[AFURLSessionManager dataTaskWithRequest:completionHandler:]函数中：
```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    __block NSURLSessionDataTask *dataTask = nil;
    dispatch_sync(url_session_manager_creation_queue(), ^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask completionHandler:completionHandler];

    return dataTask;
}
```
函数体内主要是在一个GCD队列里创建dataTask，此处用到的是GCD同步API。然后调用了第一个私有方法[AFURLSessionManager addDelegateForDataTask:completionHandler:]来设置dataTask的代理和回调的block:
```objc
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    
    [self setDelegate:delegate forTask:dataTask];
}
```
在这个函数内真正的设置了网络的代理，就是刚才提到的AFURLSessionManagerTaskDelegate类，这里有一个小细节需要注意一下，就是这一行`delegate.manager = self`，在该函数内定义了一个delegate并且持有了manager,而在AFURLSessionManagerTaskDelegate定义中声明了一个AFURLSessionManager对象manager。这就是说delegate持有了manager，manager持有了delegate，循环引用的问题就出来了，这里的解决方法是通过将AFURLSessionManagerTaskDelegate中的manager属性的内存管理语义声明为weak。我们看一下，该类的声明就知道了。
```objc
@interface AFURLSessionManagerTaskDelegate : NSObject <NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate>
@property (nonatomic, weak) AFURLSessionManager *manager;//内存管理语义为weak
@property (nonatomic, strong) NSMutableData *mutableData;
@property (nonatomic, strong) NSProgress *progress;
@property (nonatomic, copy) NSURL *downloadFileURL;
@property (nonatomic, copy) AFURLSessionDownloadTaskDidFinishDownloadingBlock downloadTaskDidFinishDownloading;
@property (nonatomic, copy) AFURLSessionTaskCompletionHandler completionHandler;
@end
```

接着上面的说，接下来函数的调用栈就进入了[AFURLSessionManager setDelegate:dataTask:]。
```objc
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self.lock unlock];
}
```
这个函数是用来干吗的呢？还加了锁。从代码上看，就是保存task的代理。

至此，所有的准备工作都做完了只等我们去启动这个任务了。没错，就是通过这一句`[dataTask resume]`。写到这，先来总结一下，这些准备工作的函数调用过程，直接甩图了：
![调用过程](/images/调用过程.png)

接下来就是代理了，代理的过程是这样的，AFURLSessionManager自己实现了代理协议，但是在代理协议方法中，根据dataTask的taskIdentifier找到相应的delegate对象，把具体代理该做的事，交给这个delegate。任务启动后第一个调用的是AFURLSessionManager这个函数：
```objc
//这是AFURLSessionManager类的方法
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    [delegate URLSession:session dataTask:dataTask didReceiveData:data];

    if (self.dataTaskDidReceiveData) {
        self.dataTaskDidReceiveData(session, dataTask, data);
    }
}
```
可以看出它把delegate取出来之后就转到AFURLSessionManagerTaskDelegate类中去了，调用了AFURLSessionManagerTaskDelegate同名方法。
```objc
//这是AFURLSessionManagerTaskDelegate类的方法
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    [self.mutableData appendData:data];
}
```
上面的这个函数才是真正干事的，AFURLSessionManager类中的同名方法只是打了一个酱油。当请求结束的时候，同样首先会调用AFURLSessionManager中的方法：
```objc
//这是AFURLSessionManager类的方法
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];

    // delegate may be nil when completing a task in the background
    if (delegate) {
        [delegate URLSession:session task:task didCompleteWithError:error];

        [self removeDelegateForTask:task];
    }

    if (self.taskDidComplete) {
        self.taskDidComplete(session, task, error);
    }

}
```
真正干事的还是AFURLSessionManagerTaskDelegate类中的同名方法。这里给出这个方法的部分代码：
```objc
//在没有错误的时候就执行这一段代码
dispatch_async(url_session_manager_processing_queue(), ^{
    NSError *serializationError = nil;
    responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];

    if (self.downloadFileURL) {
        responseObject = self.downloadFileURL;
    }

    if (responseObject) {
        userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
    }

    if (serializationError) {
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
    }
    //这是真正干事的
    dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
    	//执行completionHandler，把取到的数据传到外面去，这里的responseObject就是我们请求到的数据
        if (self.completionHandler) {
            self.completionHandler(task.response, responseObject, serializationError);
        }

        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
        });
    });
});

```
至此整个代理过程就结束了。其中有一个地方值得注意一下，那就是每用manager创建一个task时，manager都会以task的taskIdentifier属性为key，以delegate为value存入字典中。这样的做的目的应该是一个manager可以管理多个task。


---EOF---

