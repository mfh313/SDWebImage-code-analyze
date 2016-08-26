# iOS--SDWebImage
源码阅读总结
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
