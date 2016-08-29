     SDWebImage是常用的图片异步下载框架，本人一直想找个时间好好阅读下代码，趁这个机会好好总结下。
下载图片常用接口：
- (void)sd_setImageWithURL:(NSURL *)url;
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder;
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options;
- (void)sd_setImageWithURL:(NSURL *)url completed:(SDWebImageCompletionBlock)completedBlock;
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder completed:(SDWebImageCompletionBlock)completedBlock;
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options completed:(SDWebImageCompletionBlock)completedBlock;
   这些接口只是声明不一样，最终调用的接口只有一个,
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock;
这里看函数名就知道，主要涉及五个参数。比较有意思的是 SDWebImageOptions的定义和两个block定义。
typedef void(^SDWebImageDownloaderProgressBlock)(NSInteger receivedSize, NSInteger expectedSize);
typedef void(^SDWebImageDownloaderCompletedBlock)(UIImage *image, NSData *data, NSError *error, BOOL finished);
block都无返回值。再看看options的定义
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
一般我们的定义都是从0开始增加的。而这里用的左移的操作定义数值，不知道这样有啥好处。后面研究下。

现在开始研究函数的实现。
[self sd_cancelCurrentImageLoad];
objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
第一句的主要作用是先取消当前图片的下载。最终调用 UIView (WebCacheOperation)分类中的 sd_cancelImageLoadOperationWithKey方法。这个类绑定了一个字典去保存下载的信息（实际是响应 SDWebImageOperation协议 ）调用cancel接口去取消下载。
第二句主要作用是imageView与URL绑定.对应  - (NSURL *)sd_imageURL接口可以获取这个下载的URL。

然后进入正题，此时开始下载图片了。
__weak __typeof(self)wself = self;
id <SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadImageWithURL:url options:options progress:progressBlock completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            [wself removeActivityIndicator];
            if (!wself) return;
            dispatch_main_sync_safe(^{
                if (!wself) return;
                if (image && (options & SDWebImageAvoidAutoSetImage) && completedBlock)
                {
                    completedBlock(image, error, cacheType, url);
                    return;
                }
                else if (image) {
                    wself.image = image;
                    [wself setNeedsLayout];
                } else {
                    if ((options & SDWebImageDelayPlaceholder)) {
                        wself.image = placeholder;
                        [wself setNeedsLayout];
                    }
                }
                if (completedBlock && finished) {
                    completedBlock(image, error, cacheType, url);
                }
            });
        }];
[self sd_setImageLoadOperation:operation forKey:@"UIImageViewImageLoad"];

看得出来，SDWebImageManager是主要核心的类，包括 SDImageCache， SDWebImageDownloader。SDWebImageManager在init方法中初始化了这两个类。
对于SDImageCache的介绍，这个是单例。主要缓存的图片在 NSCache的实例里面，此外，还创建了文件读写专用进程 ioQueue。同时设置文件保存的时间为一周 _maxCacheAge。
收到内存警告的通知后会清空数据。

关于图片文件名的处理，主要是利用MD5命名处理。
- (NSString *)cachedFileNameForKey:(NSString *)key {
    const char *str = [key UTF8String];
    if (str == NULL) {
        str = "";
    }
    unsigned char r[CC_MD5_DIGEST_LENGTH];
    CC_MD5(str, (CC_LONG)strlen(str), r);
    NSString *filename = [NSString stringWithFormat:@"%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%@",
                          r[0], r[1], r[2], r[3], r[4], r[5], r[6], r[7], r[8], r[9], r[10],
                          r[11], r[12], r[13], r[14], r[15], [[key pathExtension] isEqualToString:@""] ? @"" : [NSString stringWithFormat:@".%@", [key pathExtension]]];

    return filename;
}

其中删除过期文件的方法，
- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock。

先找到图片的磁盘空间，
[NSURL fileURLWithPath:self.diskCachePath isDirectory:YES];
然后用文件管理器找到文件的集合。
NSArray *resourceKeys = @[NSURLIsDirectoryKey, NSURLContentModificationDateKey, NSURLTotalFileAllocatedSizeKey];
NSDirectoryEnumerator *fileEnumerator = [_fileManager enumeratorAtURL:diskCacheURL
                                                   includingPropertiesForKeys:resourceKeys
                                                                      options:NSDirectoryEnumerationSkipsHiddenFiles
                                                                 errorHandler:NULL];
通过判断是否过期去删除文件
NSDate *modificationDate = resourceValues[NSURLContentModificationDateKey];
if ([[modificationDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
      [urlsToDelete addObject:fileURL];
       continue;
}

进入正题，下载图片的流程是先在NSCache中寻找图片，然后再在文件中寻找，如果仍然不存在，则在donBlock中执行下载流程，
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock。
详细逻辑是
UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        doneBlock(image, SDImageCacheTypeMemory);
        return nil;
    }

    NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            return;
        }

        @autoreleasepool {
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.shouldCacheImagesInMemory) {
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });
先在内存中寻找，然后在磁盘中寻找，如果找到了就在memCache中存储。最终调用执行回调 doneBlock(diskImage, SDImageCacheTypeDisk);注意一点，这里面的diskImage可能是空指针。同时观察这里面的返回值。真实的NSOperation是在这里创建的。
此时如果图片仍然不存在，就需要在网络上下载了。在doneBlock里面可以看到详细。因为operation可以取消的特性，所以可以看见代码中随处看见是否取消的判断逻辑。

复杂的option判断：
SDWebImageDownloaderOptions downloaderOptions = 0;
            if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
            if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
            if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
            if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
            if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
            if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
            if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
            if (image && options & SDWebImageRefreshCached) {
                // force progressive off if image already cached but forced refreshing
                downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                // ignore image read from NSURLCache if image if cached but force refreshing
                downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
            }
这里会生成实际的对imageDownloader有用的操作枚举。

最终到了最后的下载逻辑了。

id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished)
这里为了避免循环引用，在block内部，使用strong指针强引用了 SDWebImageCombinedOperation，这个类刚开始很绕口，你看就了就会发现还是很有巧妙的地方的。 它将下载的取消糅合在一起。
__strong__typeof(weakOperation) strongOperation = weakOperation;
此工程中，为了避免多线程竞争问题，多次使用了 @synchronized闭包来加锁。

- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback;
此方法的作用是判断下载过程是否在进行，否则执行 createCallback来发起下载请求。
NSMutableDictionary *URLCallbacks中存储的block请求。

dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }

        // Handle single download of simultaneous download request for the same URL
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
        }
    });

这里我们梳理下这个self.URLCallbacks结构，首先是字典。调用self.URLCallbacks[url]获取的是操作的数组。下半部分的操作是刷新progressBlock和completedBlock。新建存储这两个的字典。NSMutableDictionary *callbacks = [NSMutableDictionary new];
if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
设置好值后，然后将其数组加入这个字典，然后重新赋值。如果first是YES则进入createCallback回调内。

[x] 这里不设置self.URLCallbacks[url] 为字典而是数组的原因是可以返回多个block回调。因为同时多个imageView下载时候一次请求，下载完各自返回block

[x] 好处是不会发起多次下载。

重磅代码在这里：
     __block SDWebImageDownloaderOperation *operation;
  __weak __typeof(self)wself = self;

NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];

operation = [[wself.operationClass alloc] initWithRequest:request
                                                        inSession:self.session
                                                          options:options
                                                         progress:
                                                         completed:
                                                         cancelled:
最后将继承 NSOperation的SDWebImageDownloaderOperation加入到 SDWebImageDownloader的 downloadQueue中，这样就会执行下载过程了。

if (wself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
            // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
            [wself.lastAddedOperation addDependency:operation];
            wself.lastAddedOperation = operation;
}
这里有趣的地方是利用NSOperation的优先级，可以实现后进先出或者先进先出。

对 SDWebImageDownloaderOperation的总结
