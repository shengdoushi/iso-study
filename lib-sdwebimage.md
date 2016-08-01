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

- SDWebImageCompat
- SDWebImageDecoder

7. 其他
- SDWebImageOperation
- SDWebImagePrefetcher

# UIImageView, UIButton 的类别

这个主要是为使用者方便使用，而添加的类别接口。内部转而调用 SDWebImageManager.sharedManager 进行图片下载。
