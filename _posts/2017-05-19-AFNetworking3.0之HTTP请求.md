---
layout:     post
title:      AFNetworking3.0之HTTP请求
date:       2017-05-19
summary:    本文主要分析AFNetworking的HTTP请求部分。
categories: objective-c HTTP
---

标签： objective-c HTTP

## 前言

前面写过了AFNetworking的数据请求和下载的部分，也对AFNetworking有了一个基本的了解，这部分讲一下它是如何实现HTTP请求的，同时也剖析一下AFNetworking整个类的构成。

## 正文

AFNetworking3.0的代码比起以前的版本，精简了很多，对于笔者来说可读性也大大的提高了。在这版本中，它的实现是对苹果给出的两套有关于网络请求的API及其代理的封装分别为:NSURLSession和NSURLConnection。其中NSURLSession是在iOS 7 或 Mac OS X 10.9以后才出现的API旨在用来替代NSURLConnection。为了兼容以前的代码，库里面还是把NSURLConnection加进来了。之前写关于AFNetworking的博客都是基于NSURLSession实现的，笔者也不打算讲AFNetworking基于NSURLConnection实现的部分，必竟现在都不太用了。

提到HTTP，就叉开一下，复习一下关于HTTP请求的东西。

### HTTP请求
当浏览器向Web服务器发出请求时，它向服务器传递了一个数据块，也就是请求信息，它由3个部分组成：
1.  请求方法URI协议/版本
2.  请求头(Request Header)
3.  请求正文

请求格式：<br>
\<request-line\><br>
\<headers\><br>
\<blank line\><br>
\[\<request-body\>\]

一个典型的例子：

GET / HTTP/1.1<br>
Host: youku.com<br>
Upgrade-Insecure-Requests: 1<br>
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36<br>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8<br>
Accept-Encoding: gzip, deflate, sdch<br>
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6<br>

这是刚刚抓的请求包头。
第一行GET / HTTP/1.1分别为请求方法，URL（这里是根路径），协议版本1.1。从第二行开始都是请求头，请求头包含许多有关的客户端环境和请求正文的有用信息。例如，请求头可以声明浏览器所用的语言，请求正文的长度等。最后是是请求正文，这里没有。请求头和请求正文之间是一个空行，这个行非常重要，它表示请求头已经结束，接下来的是请求正文。请求正文中可以包含客户提交的查询字符。

HTTP请求方法用的最多的为GET方法与POST方法。
1.  <div><font color="#FF6100">GET方法</font>：默认的HTTP请求方法，我们日常用GET方法来提交表单数据，然而用GET方法提交的表单数据只经过了简单的编码，同时它将作为URL的一部分向Web服务器发送，因此，如果使用GET方法来提交表单数据就存在着安全隐患上。例如Http://127.0.0.1/login.jsp?Name=zhangshi&Age=30&Submit=%cc%E+%BD%BB从上面的URL请求中，很容易就可以辩认出表单提交的内容。（？之后的内容）另外由于GET方法提交的数据是作为URL请求的一部分所以提交的数据量不能太大。</div>
2.  <div><font color="#FF6100">POST方法</font>：POST方法是GET方法的一个替代方法，它主要是向Web服务器提交表单数据，尤其是大批量的数据。POST方法克服了GET方法的一些缺点。通过POST方法提交表单数据时，数据不是作为URL请求的一部分而是作为标准数据传送给Web服务器，这就克服了GET方法中的信息无法保密和数据量太小的缺点。因此，出于安全的考虑以及对用户隐私的尊重，通常表单提交时采用POST方法。</div>

HTTP响应也由三个部分组成，分别是：
1.  状态行
2.  消息报头
3.  响应正文。


响应格式：<br>
＜status-line＞<br>
＜headers＞<br>
＜blank line＞<br>
\[＜response-body＞\]

一个典型的例子：<br>
HTTP/1.1 200 OK<br>
Date: Thu, 18 May 2017 12:38:15 GMT<br>
Content-Type: text/html; charset=utf-8<br>
Content-Encoding: gzip<br>

第二行为协议状态版本代码描述，这里应答码为200，接下来都是响应头。<br>
HTTP应答码也称为状态码，它反映了Web服务器处理HTTP请求状态。HTTP应答码由3位数字构成，其中首位数字定义了应答码的类型：
1.  <div><font color="#FF6100">1XX－信息类（Information）</font>：表示收到Web浏览器请求，正在进一步的处理中。</div>
2.  <div><font color="#FF6100">2XX－成功类（Successful）</font>:表示用户请求被正确接收，理解和处理例如：200 OK。</div>
3.  <div><font color="#FF6100">3XX-重定向类（Redirection）</font>:表示请求没有成功，客户必须采取进一步的动作。</div> 
4.  <div><font color="#FF6100">4XX-客户端错误（Client Error）</font>:表示客户端提交的请求有错误。例如：404 NOTFound，意味着请求中所引用的文档不存在。</div> 
5.  <div><font color="#FF6100">5XX-服务器错误（Server Error）</font>:表示服务器不能完成对请求的处理：如 500。</div> 

好了，现在叉回来。前面讲过AFURLSessionManager是整个库的核心，因为在HTTP这一部分，都是在这个类的基础上实现的。通过源码也可以看到AFURLSessionManager这个类有1000多行代码而接下来要讲的AFHTTPSessionManager类，只有300行多一点，而且AFHTTPSessionManager是继承自AFURLSessionManager的，因此HTTP请求部分的绝大多数工作都交给了AFHTTPSessionManager的父类那部分。来看看整个库的类的构成：![AFNetworking结构图](/images/AFNetworking结构图.png)

简单介绍一下这张图上的每个类吧，其实
有些类的功能，笔者也没用过，不过往后可以试试：
*   <div><font color="#FF6100">AFURLSessionManager:</font>这个类前面讲过，它对NSURLSession及NSURLSessionTaskDelegate、NSURLSessionDataDelegate、NSURLSessionDownloadDelegate和NSURLSessionDelegate进行了封装，实现了诸如数据请求，下载和上传任务。</div>
*   <div><font color="#FF6100">AFURLConnectionOperation:</font>它是NSOperation的子类，实现了NSURLConnection代理方法，这是iOS 6 或 Mac OS X 10.8以前用的API，现在基本不用了。</div>
*   <div><font color="#FF6100">AFHTTPSessionManager:</font>这是基于NSURLSession实现的用于HTTP的GET、POST等类，也是本文主要讲的东西。</div>
*   <div><font color="#FF6100">AFHTTPRequestOperation:</font>这是基于AFURLConnectionOperation子类，它和AFHTTPRequestOperationManager是一起配套使用的。这个类封装了可接受的状态码和可接受的内容类型。</div>
*   <div><font color="#FF6100">AFHTTPRequestOperationManager:</font>这个是实现HTTP各个请求方法的和AFHTTPRequestOperation配套使用的。</div>
*   <div><font color="#FF6100">AFURLRequestSerialization:</font>用来封装参数的，也可以用来设置请求头，请求的正文的格式等。比如，可以将请求的HTTP的body设置成JSON并在请求头部分将Content-Type字段的值设成application/json。</div>
*   <div><font color="#FF6100">AFURLResponseSerialization:</font>根据服务器的响应细节来解析响应的数据，此外还可以对响应的数据进行验证。例如，如果期待得到的是JSON格式的数据，那么它会检查响应码是否是2xx开头的，同时还会检查响应头的Content-Type字段是否为application/json，从而将响应的数据正解的解析成对象。</div>
*   <div><font color="#FF6100">AFNetworkReachabilityManager:</font>用来监控网络状态。</div>
*   <div><font color="#FF6100">AFSecurityPolicy:</font>用来管控网络安全的。</div>

抛开基于NSURLConnection那一部分，真正核心的就两个两类了AFHTTPSessionManager和AFURLSessionManager。

还是以官方给的例子讲吧，这里从网上找了一张图片，使用了GET方法，获取图片并且显示在一个UIImageView上：
```objc
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];

//使用了默认的序列化器
manager.responseSerializer = [AFHTTPResponseSerializer serializer];

//发起一个GET请求还是巨方便的。
[manager GET:@"http://img04.tooopen.com/images/20130701/tooopen_10055061.jpg"  parameters:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nonnull responseObject) {
    //这里知道得到的数据是一张图片，所以就直接这样生成图片了
    self.imgView.image = [UIImage imageWithData:responseObject];
    
} failure:^(NSURLSessionDataTask * _Nonnull task, NSError * _Nonnull error) {
    
    NSLog(@"%@", error);
    
}];
```
从[AFHTTPSessionManager GET:parameters:success:failure:]方法来一步一步看看，manager都什么了什么。
```objc
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(id)parameters
                      success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                      failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"GET" URLString:URLString parameters:parameters success:success failure:failure];

    [dataTask resume];

    return dataTask;
}
```

代码还是很少的，就是根据给定的URL地址调用本类的[AFHTTPSessionManager:dataTaskWithHTTPMethod:URLString:parameters:success:failure]方法生成一个dataTask然后启动它。这方法比较关键，一起来看看：
```objc
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    NSError *serializationError = nil;
    //设置request，根据HTTP的请求方法添加参数
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    __block NSURLSessionDataTask *dataTask = nil;
    //这个方法调用的是父类的方法
    dataTask = [self dataTaskWithRequest:request completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```
之所以说很关键，是因为它做了两次事，一是生成request，二是调用了父类的方法，用于设置代理，回调之类的，前面说过AFHTTPSessionManager继承自AFURLSessionManager的。这样AFHTTPSessionManager把剩下的部分全部分交给了AFURLSessionManager部分。对于AFURLSessionManager不熟悉的请求出门左拐。库中的AFURLRequestSerialization和AFHTTPResponseSerializer也挺重要的，找个时间看看。到这里整个库也差不多看了一半多吧。写的这些都是根据官方给出的用法，去一点一点看探究它的实现。刚开始的时候，笔者也想造个轮子，但是慢慢地发现造轮子考虑的东西太多，费时费力，而且又有写好开源的轮子，何不拿来用呢。世界这么大，得把时间用到别的地方去。用归用，但是也得看看源码，看看人家是怎么实现的，不是一个好的使用者，肯定不适合造轮子。

--EOF--
