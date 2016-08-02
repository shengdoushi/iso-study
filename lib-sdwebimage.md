# SDWebImage 图片下载库

# 概述

项目地址: https://github.com/rs/SDWebImage

主要是一个从远程下载图片库的项目。提供了:

- 为 UIImageView, UIButton 提供关于图片下载的类别，便于使用
- 异步的图片下载器
- 图片缓存，可以异步从内存和磁盘中加载图片
- gif, webp 格式图片的支持
- 异步的图片解码
- URL作为图片的key进行缓存：一次下载，坏URL不重复尝试

代码文件分类如下:

1. 图片格式

- UIImage+Gif
- UIImage+Webp
- UIImage+MultiFormat
- NSData+ImageContentType

2. UIKit的类别

- UIImageView+HighlightedWebCache
- UIImageView+WebCache
- UIView+WebCacheOperation
- UIButton+WebCache
- MKAnnotationView+WebCache

3. 图片缓存

- SDImageCache

4. 图片下载

- SDWebImageDownloader
- SDWebImageDownloaderOperation 图片下载的 NSOperation

5. 图片管理器

- SDWebImageManager

6. 图片解码

- SDWebImageDecoder

7. 其他

- SDWebImageOperation
- SDWebImagePrefetcher
- SDWebImageCompat 主要是一些杂项，一些内部宏，兼容性统一等

# UIImageView, UIButton 的类别

这个主要是为使用者方便使用，而添加的类别接口。内部转而调用 SDWebImageManager.sharedManager 进行图片下载。


# SDWebImageDownloader 图片下载器

```objective-c
@interface SDWebImageDownloader

@property (strong, nonatomic) NSOperationQueue *downloadQueue;
@property (SDDispatchQueueSetterSementics, nonatomic) dispatch_queue_t barrierQueue;
@property (strong, nonatomic) NSURLSession *session;
@end
```

downloadQueue 队列中管理下载任务，默认的活动下载数量为6个任务。每个 NSOperation 为 SDWebImageDownloaderOperation 对象，下载内部使用 NSURLSession 创建的 dataTask 实现，这个session 就是 SDWebImageDownloader 中的 session。



# SDImageCache 图片缓存

SDImageCache 提供了一个默认的缓存，用户也可以自定义。每个缓存有一个命名空间，默认的是 com.hackemist.SDWebImageCache.default，自定义的是 com.hackemist.SDWebImageCache.${name}。

```objective-c
- (id)init {
    return [self initWithNamespace:@"default"];
}

- (id)initWithNamespace:(NSString *)ns {
    NSString *path = [self makeDiskCachePath:ns];
    return [self initWithNamespace:ns diskCacheDirectory:path];
}


- (id)initWithNamespace:(NSString *)ns diskCacheDirectory:(NSString *)directory {
		// ...
        NSString *fullNamespace = [@"com.hackemist.SDWebImageCache." stringByAppendingString:ns];
        _memCache = [[AutoPurgeCache alloc] init]; // NSCache, 
        _memCache.name = fullNamespace;
        if (directory != nil) {
            _diskCachePath = [directory stringByAppendingPathComponent:fullNamespace];
        } else {
            NSString *path = [self makeDiskCachePath:ns];
            _diskCachePath = path;
		}
		//...
}
```

缓存提供了下边几种操作:

- 查询图片， 或查询图片是否在缓存中
- 删除图片
- 清空所有缓存
- 添加图片
- 计算磁盘缓存的大小

缓存的本质是一个 key-value 数据库，通过key来定位某个value。每个key对应的磁盘文件名为 key 的md5值，磁盘缓存的目录为 NSCachesDirectory 文件夹下的 com.hackemist.SDWebImageCache.${namespace} 目录。默认的磁盘缓存的生命周期为1周，在程序结束或程序进入后台的时候自动清理过期的图片文件。

查找一个图片时候，会返回一个 UIImage 和 SDImageCacheType。

```objective-c
typedef void(^SDWebImageQueryCompletedBlock)(UIImage *image, SDImageCacheType cacheType);
// 这个是查询结构，注意这个不是从磁盘缓存中查询，而是从所有缓存中查询。 
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock {
    if (!doneBlock) {
        return nil;
    }

    if (!key) {
        doneBlock(nil, SDImageCacheTypeNone);
        return nil;
    }

    // 先检查内存缓存中是否存在
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        doneBlock(image, SDImageCacheTypeMemory);
        return nil;
    }

	// 磁盘缓存中查找
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

		    // 在主线程中回调 doneBlock
            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });

    return operation;
}
```

当查询的时候，如果在内存缓存中，会是同步回调，而如果在磁盘缓存中，则是异步回调(且是在主线程中回调)。因为磁盘中加载可能涉及 io 读取，图片解码等操作，这些操作都放在了后台进行。查询接口返回的是 NSOperation, 可以取消掉，当然这个取消只是需要在磁盘缓存中查找，且在磁盘查询开始前进行取消才有效。在从磁盘找到后，如果需要，就加入到内存缓存中。

内存缓存 _memCache 是使用苹果提供的 NSCache 实现的，这个很多开源项目都使用其作为内存图片缓存使用。SDWebImage 中，当系统发出内存警告的时候，会自动将 _memCache 中所有数据清理掉。NSCache 中可以为每个数据设置一个成本，SDWebImage 每个图片的成本为图片的当前宽*当前高 (image.size.height * image.size.width * image.scale * image.scale)。

而磁盘缓存查找逻辑里，建立了一个 _ioQueue 来作为磁盘操作的一个分派队列（dispatch queue), 注意这个 ioQueue 是一个串行队列。即所有的磁盘缓存查找都是一个查找完成，才开始查找下一个，这样查询结果返回 NSOperation 的取消操作才有意义。

```objective-c
_ioQueue = dispatch_queue_create("com.hackemist.SDWebImageCache", DISPATCH_QUEUE_SERIAL);
```

所有的对于磁盘缓存的操作都是在 _ioQueue 上完成的，包括查询，删除，清空，添加等。

从缓存中移除一个图片实现起来就比较简单了, 从内存和磁盘都移除:

```objective-c

- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion {
    
    if (key == nil) {
        return;
    }

    if (self.shouldCacheImagesInMemory) {
        [self.memCache removeObjectForKey:key];
    }

    if (fromDisk) {
        dispatch_async(self.ioQueue, ^{
            [_fileManager removeItemAtPath:[self defaultCachePathForKey:key] error:nil];
            
            if (completion) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completion();
                });
            }
        });
    } else if (completion){
        completion();
    }
}
```

最后是添加图片：

```objective-c
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk {
    if (!image || !key) {
        return;
    }
    // 没有关闭内存缓存，就先放到内存里
    if (self.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(image);
        [self.memCache setObject:image forKey:key cost:cost];
    }

    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            NSData *data = imageData;

	        // 当原始的图片数据没有，或是强制从 UIImage 中重新计算图片数据的话。就需要重新计算图片数据，并保存到文件
            if (image && (recalculate || !data)) {
                int alphaInfo = CGImageGetAlphaInfo(image.CGImage);
                BOOL hasAlpha = !(alphaInfo == kCGImageAlphaNone ||
                                  alphaInfo == kCGImageAlphaNoneSkipFirst ||
                                  alphaInfo == kCGImageAlphaNoneSkipLast);
                BOOL imageIsPng = hasAlpha;
                if ([imageData length] >= [kPNGSignatureData length]) {
                    imageIsPng = ImageDataHasPNGPreffix(imageData);
                }

                if (imageIsPng) {
                    data = UIImagePNGRepresentation(image);
                }
                else {
                    data = UIImageJPEGRepresentation(image, (CGFloat)1.0);
                }
            }
			// 保存到某个磁盘图片文件中
            [self storeImageDataToDisk:data forKey:key];
        });
    }
}

```

## 图片解码

图片解码指的是从一个原始的图片文件数据到UIKit显示的图片数据的转换，因为这个过程耗时，所以可能引起主界面卡顿，所以 SDWebImage 将这个过程放在了后台解码。解码过程见 UIImage 的 +(UIImage*)decodedImageWithImage:(UIImage*) image。 主要流程是构建一个 CGContext 上下文，在上边将图片数据写入到里边，然后在导出成一个 UIImage 数据。

# 其他

## 主线程block执行

sdwebimage 中的场景是在主线程执行一个block，如果是主线程则立即执行，否则传递到主线程去执行, 实现起来，就是需要用 [NSThread isMainThread] 来判断下当前是不是主线程：

```objective-c
#define dispatch_main_sync_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_sync(dispatch_get_main_queue(), block);\
    }

#define dispatch_main_async_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
```


## NSCache 苹果提供的缓存

NSCache 和 NSDictionary 用法非常像。当内存过低时候，会自动清理缓存，所以这次取出来，不代表下次就能取出来。NSCache 提供了两个限制控制变量: countLimit 和 totalCostLimit。 countLimit 顾名思义，就是数量限制，最多能放多少个数据， 如果数据超出这个限制了，内部会自动将一些数据清掉，以保证不超过这个限制。 totalCostLimit 是一个成本控制，在 NSCache 中，可以为每个数据设置一个成本，当总成本超过 totalCostLimit 了，会自动内部清理下，以保证不超过这个限制。

## UIBackgroundTaskIdentifier 后台任务

app 进入后台后，会被自动挂起。苹果允许后台执行一些特殊任务，如跟踪gps，播放音乐，下载内容等。

SDWebImage 中使用的 UIBackgroundTaskIdentifier 来做后台清理磁盘任务。UIBackgroundTaskIdentifier 主要用来做一些有限的任务，任务结束后就可以回归正常后台状态。官方代码demo如下：


```objective-c
- (void)applicationDidEnterBackground:(UIApplication *)application
{
    bgTask = [application beginBackgroundTaskWithName:@"MyTask" expirationHandler:^{
        // Clean up any unfinished task business by marking where you
        // stopped or ending the task outright.
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }];
 
    // Start the long-running task and return immediately.
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
 
        // Do the work associated with the task, preferably in chunks.
 
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    });
}
```

## PNG 图片判断

一个 PNG 文件格式是:

```
| PNG文件头 |	PNG数据块 |	……	| PNG数据块 |
```

PNG 文件头是一个 8 个字节的定值：

```
89 50 4E 47 0D 0A 1A 0A
```

## JPEG 文件判断

一般读取第一个字节是否是 0xFF

## Gif 文件判断

Gif的前三个字节为固定值 "GIF", 即

```
89 50 4E
```

## 其他图片格式判断

可以参考  http://blog.csdn.net/include1224/article/details/5195470

## @autoreleasepool
