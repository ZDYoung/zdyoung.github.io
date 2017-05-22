---
layout:     post
title:      AFNetworking3.0源码解读之下载请求
date:       2017-05-05 
summary:    本文主要分析AFNetworking的下载请求部分。
categories: objective-c AFNetworking 
---

标签： objective-c AFNetworking download

--- 

### 前言

上一篇简单介绍了一下AFURLSessionManager类的结构以及发起一个网络请求的函数调用过程。AFURLSessionManager类中有很多属性，还定义了很多的局部全局变量，在数据请求那大部分用不上。比如，AFURLSessionManager类定义了很多的block:
```objc
typedef void (^AFURLSessionDidBecomeInvalidBlock)(NSURLSession *session, NSError *error);
typedef NSURLSessionAuthChallengeDisposition (^AFURLSessionDidReceiveAuthenticationChallengeBlock)(NSURLSession *session, NSURLAuthenticationChallenge *challenge, NSURLCredential * __autoreleasing *credential);

typedef NSURLRequest * (^AFURLSessionTaskWillPerformHTTPRedirectionBlock)(NSURLSession *session, NSURLSessionTask *task, NSURLResponse *response, NSURLRequest *request);
typedef NSURLSessionAuthChallengeDisposition (^AFURLSessionTaskDidReceiveAuthenticationChallengeBlock)(NSURLSession *session, NSURLSessionTask *task, NSURLAuthenticationChallenge *challenge, NSURLCredential * __autoreleasing *credential);
typedef void (^AFURLSessionDidFinishEventsForBackgroundURLSessionBlock)(NSURLSession *session);

typedef NSInputStream * (^AFURLSessionTaskNeedNewBodyStreamBlock)(NSURLSession *session, NSURLSessionTask *task);
typedef void (^AFURLSessionTaskDidSendBodyDataBlock)(NSURLSession *session, NSURLSessionTask *task, int64_t bytesSent, int64_t totalBytesSent, int64_t totalBytesExpectedToSend);
typedef void (^AFURLSessionTaskDidCompleteBlock)(NSURLSession *session, NSURLSessionTask *task, NSError *error);

typedef NSURLSessionResponseDisposition (^AFURLSessionDataTaskDidReceiveResponseBlock)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSURLResponse *response);
typedef void (^AFURLSessionDataTaskDidBecomeDownloadTaskBlock)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSURLSessionDownloadTask *downloadTask);
typedef void (^AFURLSessionDataTaskDidReceiveDataBlock)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSData *data);
typedef NSCachedURLResponse * (^AFURLSessionDataTaskWillCacheResponseBlock)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSCachedURLResponse *proposedResponse);

typedef NSURL * (^AFURLSessionDownloadTaskDidFinishDownloadingBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location);
typedef void (^AFURLSessionDownloadTaskDidWriteDataBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, int64_t bytesWritten, int64_t totalBytesWritten, int64_t totalBytesExpectedToWrite);
typedef void (^AFURLSessionDownloadTaskDidResumeBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, int64_t fileOffset, int64_t expectedTotalBytes);

typedef void (^AFURLSessionTaskCompletionHandler)(NSURLResponse *response, id responseObject, NSError *error);
```
这里列出了该类所有的block，有没有很吓人，刚开始看的时候，我也看的头疼，后来调试的时候发现在本文接下来要讲的内容中，只用到了两个分别是AFURLSessionTaskCompletionHandler和AFURLSessionDownloadTaskDidFinishDownloadingBlock，这两个block都是delegate在完成代理后调用的。说明AFURLSessionManager中的block属性打酱油了，那我就很好奇了，既然是打酱油的那还要它干吗？其实不是，我们先来仔细观察一下它们的名字——objective-c的命名都是见名知意的，这一点我感觉很不错——以本文的下载为例，AFURLSessionDownloadTaskDidFinishDownloadingBlock意：下载任务完成时的block。那我们怎么去使用这个呢？呆会就讲。

## 正文

和上一篇一样还是以官方git上的例子看起，源码如下：
``` objc
NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
AFURLSessionManager *manager = [[AFURLSessionManager alloc] initWithSessionConfiguration:configuration];

NSURL *URL = [NSURL URLWithString:@"http://img04.tooopen.com/images/20130701/tooopen_10055061.jpg"];
NSURLRequest *request = [NSURLRequest requestWithURL:URL];

NSURLSessionDownloadTask *downloadTask = [manager downloadTaskWithRequest:request progress:nil destination:^NSURL *(NSURL *targetPath, NSURLResponse *response) {
    NSURL *documentsDirectoryURL = [[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:NO error:nil];

    return [documentsDirectoryURL URLByAppendingPathComponent:[response suggestedFilename]];
} completionHandler:^(NSURLResponse *response, NSURL *filePath, NSError *error) {
    NSLog(@"File downloaded to: %@", filePath);
    self.imgView.image = [UIImage imageWithData:[NSData dataWithContentsOfURL:filePath]];

}];
//以下三行是我添加的，下面三行设置了下载完成后的回调
typedef NSURL * (^DownloadTaskDidFinishDownloadingBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location);

id block = ^(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location){
    //doing something
    NSLog(@"%@",location);
/*
 * 将上面的打印语句注释掉，改成下面的，再运行一次看看会有什么结果？
    NSURL *documentsDirectoryURL = [[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:NO error:nil]; 
        
    return [documentsDirectoryURL URLByAppendingPathComponent:[downloadTask.response suggestedFilename]];
    */
};
[manager setDownloadTaskDidFinishDownloadingBlock:block];


[downloadTask resume];

```
这里我在网上随便找了一张图片，然后还设置了manager的一个AFURLSessionDownloadTaskDidFinishDownloadingBlock，就是刚才说那些打酱油中的block中的一个。在这里只是简单的打印了一下location，并没有返回一个路径，注意哦，这个block要返回一个路径的哦，这个路径是用来存放我们下载的图片的。

这里通过manager实例方法来创建downloadTask:
```objc
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(NSProgress * __autoreleasing *)progress
                                          destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    __block NSURLSessionDownloadTask *downloadTask = nil;
    dispatch_sync(url_session_manager_creation_queue(), ^{
        downloadTask = [self.session downloadTaskWithRequest:request];
    });
    //这个是用来设置下载代理的
    [self addDelegateForDownloadTask:downloadTask progress:progress destination:destination completionHandler:completionHandler];

    return downloadTask;
}
```
然后又进入到[NSURLSessionManager addDelegateForDownloadTask:progress:destination:completionHandler]这个函数设置了具体的代理对象，以及下载完成后设置了图片存储路径的block，上一篇的数据请求也会进行这一步，但下载比数据请求多了一些东西，让我们来看看多了什么：
```objc
- (void)addDelegateForDownloadTask:(NSURLSessionDownloadTask *)downloadTask
                          progress:(NSProgress * __autoreleasing *)progress
                       destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                 completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;
//downloadTaskDidFinishDownloading -> NSURL * (^AFURLSessionDownloadTaskDidFinishDownloadingBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location);
    if (destination) {
        delegate.downloadTaskDidFinishDownloading = ^NSURL * (NSURLSession * __unused session, NSURLSessionDownloadTask *task, NSURL *location) {
            return destination(location, task.response);
        };
    }

    if (progress) {
        *progress = delegate.progress;
    }

    downloadTask.taskDescription = self.taskDescriptionForSessionTasks;

    [self setDelegate:delegate forTask:downloadTask];
}
```
和数据请求相比，这里主要多了两个地方一个是设置了下载完成后的回调，一个是设置了下载的进度。这里设置的是delegate相应的回调。函数的最后一句就是保存delegate，这个和数据请求部分一样，不解释了。最后还会调用这个函数：
```objc
- (void)setDownloadTaskDidFinishDownloadingBlock:(NSURL * (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location))block {
    self.downloadTaskDidFinishDownloading = block;
}
```
因为我们之前设置了manager下载完成后的回调就是这句：
```objc
[manager setDownloadTaskDidFinishDownloadingBlock:block];
```

现在可以启动downloadTask了。

网络正常的情况以及URL没错的情况下，收到下载的数据后就会调用manager实现的代理方法：
```objc
//AFURLSessionManager->NSURLSessionDownloadDelegate->下载时调用，可能调用多次
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:downloadTask];
    [delegate URLSession:session downloadTask:downloadTask didWriteData:bytesWritten totalBytesWritten:totalBytesWritten totalBytesExpectedToWrite:totalBytesExpectedToWrite];
//downloadTaskDidWriteData - > void (^AFURLSessionDownloadTaskDidWriteDataBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, int64_t bytesWritten, int64_t totalBytesWritten, int64_t totalBytesExpectedToWrite);
    if (self.downloadTaskDidWriteData) {
        self.downloadTaskDidWriteData(session, downloadTask, bytesWritten, totalBytesWritten, totalBytesExpectedToWrite);
    }
}
```
这个函数没什么可以讲主要就是用来计算下载的进度。当下载完成的时候就会进入到这个代理方法中：
```objc
//AFURLSessionManager->NSURLSessionDownloadDelegate->下载完成是调用
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:downloadTask];
//downloadTaskDidFinishDownloading -> NSURL * (^AFURLSessionDownloadTaskDidFinishDownloadingBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location);
    //下载的时候这个block没有用
    if (self.downloadTaskDidFinishDownloading) {
        NSURL *fileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        if (fileURL) {
            delegate.downloadFileURL = fileURL;
            NSError *error = nil;
            [[NSFileManager defaultManager] moveItemAtURL:location toURL:fileURL error:&error];
            if (error) {
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:error.userInfo];
            }

            return;
        }
    }

    if (delegate) {
        [delegate URLSession:session downloadTask:downloadTask didFinishDownloadingToURL:location];
    }
}
```
这里有一个重点，单步执行的时候会发现，程序会进入到第一个if里面。还记得在开头的例子中我加了三行代码：
```objc
typedef NSURL * (^DownloadTaskDidFinishDownloadingBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location);

id block = ^(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location){
    NSLog(@"%@",location);
};
[manager setDownloadTaskDidFinishDownloadingBlock:block];
```
这三行设置了manager的downloadTaskDidFinishDownloading属性，它是下载完成后的一个回调，它要求返回一个URL用来存放下载的图片，如果在这个block里返回了一个URL那么它会进入到第一个if里去，并且执行完就返回了。图片也就保存到了返回的这个URL了，后面的代码就不会执行了，如果没有返回一个URL的话，程序就是把下载的图片保存到这里：
```objc
//这是例子开头我们设置的用来保存图片的URL,如果我们设置了managerr的downloadTaskDidFinishDownloading属性并且同时在这个block里返回了一个URL那么下面这两行代码就不会执行了，因为已经有保存图片的URL了。只有block里没有返回URL下面的代码才会执行
NSURL *documentsDirectoryURL = [[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:NO error:nil];

return [documentsDirectoryURL URLByAppendingPathComponent:[response suggestedFilename]];
```
上面的这两行代码是在这个代理方法里调用的：
```objc
//AFURLSessionManagerTaskDelegate->NSURLSessionTaskDelegate->下载完成时调用
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    NSError *fileManagerError = nil;
    self.downloadFileURL = nil;
//downloadTaskDidFinishDownloading -> NSURL * (^AFURLSessionDownloadTaskDidFinishDownloadingBlock)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location);
    if (self.downloadTaskDidFinishDownloading) {
        //下载完成时调用block
        self.downloadFileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        if (self.downloadFileURL) {
            [[NSFileManager defaultManager] moveItemAtURL:location toURL:self.downloadFileURL error:&fileManagerError];

            if (fileManagerError) {
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:fileManagerError.userInfo];
            }
        }
    }
}
```
这里需要注意的是这个downloadTaskDidFinishDownloading是delegate里的和manager里的不一样，不要被迷惑了。这个block返回URL，往下执行就是把图片保存在这个URL上了。这个block是不是不知道我们什么时候设置的了。写了这么我，我也差一点忘记了。它是在这个[AFURLSessionMananger addDelegateForDownloadTask:progress:destination:completionHandler:]函数里设置的，注意里面的这一行代码：
```objc
if (destination) {
    delegate.downloadTaskDidFinishDownloading = ^NSURL * (NSURLSession * __unused session, NSURLSessionDownloadTask *task, NSURL *location) {
        return destination(location, task.response);
    };
}
```
block的用法有没有很神奇它竟然捕获从最外面传进来的destination，并且保存到现在，而且现在要执行它来返回图片要保存的URL。

到这里基本也就快结束了，执行完，就会进入这个[AFURLSessionManagerTaskDelegate URLSession:task:didCompleteWithError:]函数里了，这一步和数据请求没有什么区别。经过这么多调用，图片也就下载完了。前面设置manager的downloadTaskDidFinishDownloading属性，这个个人认为可以用来返回指定的URL，也可以用来做一些，下载完成后的后续处理工作。

---EOF---

