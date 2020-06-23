# SDWebImage总结

## 1.实现原理
### 1.1结构图
![](.png)

### 1.2流程图

![](.png)

## 2.目录结构

- ### Downloader
	- SDWebImageDownloader
	- SDWebImageDownloaderOperation
	
- ### Cache
	- SDImageCache
	- SDImageCacheConfig
	
- ### Decoder
	- SDWebImageDecoder
	- SDWebImageCodersManager
- ### Utils
	- SDWebImageManager
	- SDWebImagePrefetcher
	- SDWebImageTransition
	
- ### Categories
	- UIView+WebCacheOperation
	- UIImageView+WebCache
	- UIImageView+HighlightedWebCache
	- UIButton+WebCache
	- MKAnnotationView+WebCache
	- NSData+ImageContentType
	- UIImage+GIF
	- UIImage+MultiFormat
	- UIImage+WebP

## 3.核心原理	
|  类名   | 预览 |
|  ----  | ----  |
| SDWebImageDownloader | 是专门用来下载图片和优化图片加载的，跟缓存没有关系 |
| SDWebImageDownloaderOperation  | 继承于 NSOperation，用来处理下载任务的 |
| SDImageCache  | 用来处理内存缓存和磁盘缓存（可选)的，其中磁盘缓存是异步进行的，因此不会阻塞主线程 |

## 4.实现细节
- 下载（SDWebImageDownloader ）
- 缓存（SDImageCache）
- 将缓存和下载的功能组合起来（SDWebImageManager）
- 封装成 UIImageView 等类的分类方法（UIImageView+WebCache 等）

### 1. 图片下载
### 1.1 SDWebImageDownloader
SDWebImageDownloader 继承于 NSObject，主要承担了异步下载图片和优化图片加载的任务。

#### 几个问题

- 如何实现异步下载，也就是多张图片同时下载？
- 如何处理同一张图片（同一个 URL）多次下载的情况？

枚举定义

```
// 下载选项
typedef NS_OPTIONS(NSUInteger, SDWebImageDownloaderOptions) {
    SDWebImageDownloaderLowPriority = 1 << 0,
    SDWebImageDownloaderProgressiveDownload = 1 << 1,
    SDWebImageDownloaderUseNSURLCache = 1 << 2,
    SDWebImageDownloaderIgnoreCachedResponse = 1 << 3,
    SDWebImageDownloaderContinueInBackground = 1 << 4,
    SDWebImageDownloaderHandleCookies = 1 << 5,
    SDWebImageDownloaderAllowInvalidSSLCertificates = 1 << 6,
    SDWebImageDownloaderHighPriority = 1 << 7,
};

// 下载任务执行顺序
typedef NS_ENUM(NSInteger, SDWebImageDownloaderExecutionOrder) {
    SDWebImageDownloaderFIFOExecutionOrder, // 先进先出
    SDWebImageDownloaderLIFOExecutionOrder  // 后进先出
};
```

#### .h 文件中的属性：

```
@property (assign, nonatomic) BOOL shouldDecompressImages;  // 下载完成后是否需要解压缩图片，默认为 YES
@property (assign, nonatomic) NSInteger maxConcurrentDownloads;
@property (readonly, nonatomic) NSUInteger currentDownloadCount;
@property (readonly, nonatomic, nonnull) NSURLSessionConfiguration *sessionConfiguration;
@property (assign, nonatomic) NSTimeInterval downloadTimeout;
@property (assign, nonatomic) SDWebImageDownloaderExecutionOrder executionOrder;
+ (nonnull instancetype)sharedDownloader;
@property (strong, nonatomic, nullable) NSURLCredential *urlCredential;
@property (strong, nonatomic) NSString *username;
@property (strong, nonatomic) NSString *password;
@property (nonatomic, copy) SDWebImageDownloaderHeadersFilterBlock headersFilter;
```

#### .m文件中的属性：

```
@property (strong, nonatomic, nonnull) NSOperationQueue *downloadQueue;
@property (weak, nonatomic, nullable) NSOperation *lastAddedOperation;
@property (assign, nonatomic, nullable) Class operationClass;
@property (strong, nonatomic, nonnull) NSMutableDictionary<NSURL *, NSOperation<SDWebImageDownloaderOperationInterface> *> *URLOperations;
@property (strong, nonatomic, nullable) SDHTTPHeadersMutableDictionary *HTTPHeaders;
@property (strong, nonatomic, nonnull) dispatch_semaphore_t operationsLock; // a lock to keep the access to `URLOperations` thread-safe
@property (strong, nonatomic, nonnull) dispatch_semaphore_t headersLock; // a lock to keep the access to `HTTPHeaders` thread-safe

// The session in which data tasks will run
@property (strong, nonatomic) NSURLSession *session;
```

#### .h 文件中方法

```
- (nonnull instancetype)initWithSessionConfiguration:(nullable NSURLSessionConfiguration *)sessionConfiguration NS_DESIGNATED_INITIALIZER;

- (void)setValue:(nullable NSString *)value forHTTPHeaderField:(nullable NSString *)field;

- (nullable NSString *)valueForHTTPHeaderField:(nullable NSString *)field;

- (void)setOperationClass:(nullable Class)operationClass;

- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock;

- (void)cancel:(nullable SDWebImageDownloadToken *)token;

- (void)setSuspended:(BOOL)suspended;

- (void)cancelAllDownloads;

- (void)createNewSessionWithConfiguration:(nonnull NSURLSessionConfiguration *)sessionConfiguration;

- (void)invalidateSessionAndCancel:(BOOL)cancelPendingOperations;
```

#### .m 文件中方法

```
// Lifecycle
+ (void)initialize;
+ (SDWebImageDownloader *)sharedDownloader;
- init;
- (void)dealloc;

// Setter and getter
- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field;
- (NSString *)valueForHTTPHeaderField:(NSString *)field;
- (void)setMaxConcurrentDownloads:(NSInteger)maxConcurrentDownloads;
- (NSUInteger)currentDownloadCount;
- (NSInteger)maxConcurrentDownloads;
- (void)setOperationClass:(Class)operationClass;

// Download
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageDownloaderOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageDownloaderCompletedBlock)completedBlock;
- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock
          andCompletedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock
                     forURL:(NSURL *)url
             createCallback:(SDWebImageNoParamsBlock)createCallback;

// Download queue            
- (void)setSuspended:(BOOL)suspended;
```

### 具体实现

![](3.png)
首先从`initialize`开始分析，初始化`SDNetworkActivityIndicator`进行监听添加和移除下载通知，暂停通知，来显示和隐藏状态栏上的 network activity indicator。这里如果不需要的话就不用导入到项目中，这里用了runtime不会出现什么问题。

项目中的`sharedDownloader`和`init`方法分别是创建了单例方法初始化和调用了`initWithSessionConfiguration`方法，使用了默认的`defaultSessionConfiguration`配置进行了初始化方法。

![](4.png)

在`initWithSessionConfiguration`方法中进行了一些属性的初始化

```
- (nonnull instancetype)initWithSessionConfiguration:(nullable NSURLSessionConfiguration *)sessionConfiguration {
	设置下载 operation 的默认执行顺序（先进先出还是先进后出）
	初始化下载队列
	最大下载队列数
	设置 _HTTPHeaders 默认值
	userAgent
	超时时长默认15s
	队列🔐 && HTTPHeaders🔐
	...	
}
```

核心下载方法`downloadImageWithURL:options:progress:completed`


```
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    
    //这里进行了下载地址的安全校验
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return nil;
    }
    
    //由于可能出现下载多个图片所以需要进行加锁处理，否则可能会出现线程安全问题。
    LOCK(self.operationsLock);
    
    //这里需要根据URL获取对应的operation
    NSOperation<SDWebImageDownloaderOperationInterface> *operation = [self.URLOperations objectForKey:url];
    
    //进行当前任务是否完成的判断
    if (!operation || operation.isFinished) {
        operation = [self createDownloaderOperationWithUrl:url options:options];
        //这里需要进行任务完成的后调且进行任务移除操作。
        __weak typeof(self) wself = self;
        operation.completionBlock = ^{
       	// 使用了strong weak dance的操作和信号量加锁，保证了不会出现循环引用和移除的安全问题
            __strong typeof(wself) sself = wself;
            if (!sself) {
                return;
            }
            LOCK(sself.operationsLock);
            [sself.URLOperations removeObjectForKey:url];
            UNLOCK(sself.operationsLock);
        };
        //添加到了任务队列中
        [self.URLOperations setObject:operation forKey:url];
        //添加到了下载队列中
        [self.downloadQueue addOperation:operation];
    }
    UNLOCK(self.operationsLock);

   //SDWebImageDownloadToken 是作为任务的容器封装类，包含了当前任务的operation && url && token.downloadOperationCancelToken
    id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
    
    SDWebImageDownloadToken *token = [SDWebImageDownloadToken new];
    token.downloadOperation = operation;
    token.url = url;
    token.downloadOperationCancelToken = downloadOperationCancelToken;
    return token;
}
```

#### 知识点：

1. `strong weak dance` 的使用
2. `dispatch_semaphore_t `的使用保证任务的安全
3. `NSOperationQueue` 的使用


