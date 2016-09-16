---
layout: post
title: "AFNetworking源码阅读(一)"
date: 2016-09-16 21:03:58 +0800
comments: true
categories: 
---

AFNetworking 3.1.0 主要是对NSURLSession做了封装.

	- 核心类是AFURLSessionManager.
	- 其派生子类AFHTTPSessionManager基于HTTP协议的请求方式进一步封装了GET, HEAD, POST, PATCH, DELETE, PUT等方法.
	- AFURLRequestSerialization提供了请求序列化
	- AFURLResponseSerialization把收到的二进制数据序列化成JSON, XML,HTTP, PropertyList, Image等便于解析的格式。
	- AFNetworkReachabilityManager监测网络状态
	- AFSecurityPolicy提供安全验证机制

###AFURLSessionManager
这个模块主要包括两个类：```AFURLSessionManager```和```AFURLSessionManagerTaskDelegate```
AFURLSessionManager用NSURLSessionConfiguration来初始化，并维护

	- 一个NSOperationQueue,
	- 一个responseSerializer（默认JSON）
	- securityPolicy
	- reachabilityManager
	- 一个存储taskID和TaskDelegate的字典来维护task，读写锁保护(NSLock)

configuration可以设置cache policy，timeout，networkService, TSL, cookie, HTTPHeader, Credential Storage等。

AFURLSessionManager主要实现了四个协议:

```
NSURLSessionDelegate（处理session失效和验证）
NSURLSessionTaskDelegate（普通的请求任务处理）
NSURLSessionDataDelegate（数据处理）
NSURLSessionDownloadDelegate（下载任务处理）
```
每个实现的协议方法都有一个对应的block，方法调用的时候执行block，如果没有再执行默认操作。

把任务完成，数据处理，上传下载进度管理，下载完成后文件拷贝等部分独立出来，专门用一个类AFURLSessionManagerTaskDelegate来管理，所以这个类也实现了后3个协议。

这个模块中有<font color=red>两个全局的```dispatch_queue```：</font>

1. 一个是串行队列用于创建任务
	dataTask(普通任务), uploadTask, downloadTask以dispatch_sync的方式放入队列
2. 一个是并发队列用于处理任务
	NSURLSessionTaskDelegate的didComplete方法，如果没有错误，把响应序列化，完成处理等封装成一个block放入此并发队列中。

3. 还有一个dispatch_group_t，用来管理维护所有task的completionBlock。完成回调的处理在主队列里面。

另外，SessionManager还维护了一个NSOperationQueue, 这个队列的最大并发数是1，等于串行。这个队列是用来处理NSURLSession的delegate回调和session的completion回调。这个completion回调是等到session中所有的data, upload, download task都结束的时候调的。这个回调主要做了一件事，给所有的task设置一个taskDelegate。既然所有任务都结束了，此刻的delegate设置有什么用呢？并且在创建任务的时候就已经设置过taskDelegate了。

那么跑一个程序看一下。

客户端发起第一个请求Get

1. 创建请求的时候发生在main queue里，如果iOS8以下是在创建任务的串行队列里
2. ```NSURLSessionDelegate```的```didReceiveChallenge```得到调用，此时运行在NSOperationQueue。
3. NSURLSessionDataDelegate的didReceiveData得到调用，此时运行在NSOperationQueue
4. sessionManager的NSURLSessionTaskDelegate的didCompleteWithError得到调用，方法中再去调用taskDelegate的didCompleteWithError方法，此时运行在NSOperationQueue。response序列化在processing_dispatch_queue中调用，taskDelegate在此方法中被移除。

可以看出所有delegate的方法回调都是在session的operationQueue里发生，请求创建在main queue或者```create_dispatch_queue```中发生，response序列化在```processing_dispatch_queue```里发生。创建结束并没有调用session的completionHandler，在sessionManager初始化时会调用一次，此时没有task。进到getTasksWithCompletionHandler方法定义处，发现注释completionHandler是用来处理未解决的data, upload, download task，猜想应该是在session意外invalide的时候，若此时有未解决的task，在session结束之前会在这里集中处理一遍。

---
##技巧
1. ```NSStringFromSelector(_cmd)```获取当前运行方法的名字

2. 信号量

	```
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);//创建一个信号量计数==0
dispatch_semaphore_signal(semaphore);//计数+1
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);//如果semaphore计数>=1,计数-1，返回，程序继续运行；如果计数==0，一直等待	
	```

	```
	- (NSArray *)tasksForKeyPath:(NSString *)keyPath {
    	__block NSArray *tasks = nil;
    	dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    	[self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
       	 if ([keyPath isEqualToString:NSStringFromSelector(@selector(dataTasks))]) {
            tasks = dataTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(uploadTasks))]) {
            tasks = uploadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(downloadTasks))]) {
            tasks = downloadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(tasks))]) {
            tasks = [@[dataTasks, uploadTasks, downloadTasks] valueForKeyPath:@"@unionOfArrays.self"];
        }
        	dispatch_semaphore_signal(semaphore);
    	}];
   	 dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
   	 return tasks;
	}
	```
	
	这个方法利用信号量，把异步返回的task，变为同步。也就是说，当CompletionHandler执行前，信号量一直为0，当前线程阻塞直到CompletionHandler执行，信号量+1，返回task。

3. 用method_swizzling给```resume```和```suspend```方法加上发通知

4. 对某处的代码忽略警告。push存储GCC的当前诊断状态，pop恢复到存储的诊断状态，push和pop使忽略效果只在之间生效。

	```
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
```



