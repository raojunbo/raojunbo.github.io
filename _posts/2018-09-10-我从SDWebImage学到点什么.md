---
layout: post
title: 我从SDWebImage学到点什么
date: 2018-09-10 23:34:15.000000000 +09:00
tags: iOS技术
---

本篇文章主要是总结，我从读SDWebImage源码里学到了什么?
本来想写个长篇大论一句句解释的。但需要的时间太长。因为是开源项目，不是基本的需要深入理解的东西。都是平时使用的知识点的一个汇集。感觉还是没必要细说了。

## Category

UIImageView的分类的API设计时，其他的方法都是基于这个**全参数函数**来设计。也就是真正的核心函数就是这一个。其他的对外扩展的函数都是调用这个函数来实现的。注意此处的API设计都采用添加了添加前缀。（这就是一种很好的设计方法）

```
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock {
```

在对系统的某个框架里的内容做扩展时，就可以采取这种方法。用分类的方法，调用自己的核心逻辑类。

## SDWebImageManager
SDWebImageManager是一个门面类。利用SDWebImageCache与SDWebImageDowloader实现图片的下载与缓存的逻辑。

- 难点

这里的难点在取图片与下载图片时需要满足能取消操作。比如正在下载图片，用户取消了这个图片的下载。这时我需要知道当前所处的阶段（时正在从缓存区呢，还是正在下载呢）。
SDWebImageManager对这种情况采用了operation的操作。将取图，下载搞成两种操作。因为操作是可以取消的。（哈哈，这就是用operation的好处）

## SDWebImageCache
SDWebImageCache有内存缓存与磁盘缓存，幷专门针对SDWebImage给定一个命名空间（我觉得给命名空间是一个很好的设计）。

- 内存缓存
NSCache

- 磁盘缓存
直接是文件存储

- 清除缓存策略
每种缓存都有过期时间，条目数目限制，总占用空间大小。当进入后台或者收到清除缓存通知时。首先排序最后修改日期，然后清除过期文件。在查看是否超过了总占用空间，占用了就冲排序好的最后面删除以半。如此反复。

- 缓存里的多线程
其实只有缓存到磁盘时会对目录做操作。

综上所述 缓存其实就是内存缓存如何做，磁盘缓存如何做，还要缓存策略如何设计。

## SDWebImageDownloader

整个SDWebImageDownloader就是下面这个函数的实现。

```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageDownloaderOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageDownloaderCompletedBlock)completedBlock;
```

难点

- 取消
下载需要支持取消，具有这种可以取消的特性的操作，就用operation来是实现是最完美不过的了。从而在整个dowloader里管理这些operation。

- 同一url的一次下载
为了防止同一url下载多次。在downloader里保存了urlCallBacks，通过下载玩后，在urlCallBacks里遍历取出相对应的url的callBacks来执行回调。如果不这样处理。虽然能正常执行，但同url的下载会执行多次。

- 下载operation
SDWebImageDownloaderOperation这个主要管理单个下载的过程。利用NSURLSession进行数据的处理。实现operation的协议，比如executing，finished等，以辅助operation的可操作性。

## SDWebImageDecoder
下载的图片通常不是位图，在加载显示前都会解码。这里SDWebImage进行了提前解码的工作。幷没有自己写算法进行解码。而是将苹果的解码提前了。或者放在了一个子线程里执行。(其实就时将图片绘制，然后取出位图，是不是很巧妙)

```
  CGContextRef context = CGBitmapContextCreate(NULL,
                                                     width,
                                                     height,
                                                     bitsPerComponent,
                                                     bytesPerRow,
                                                     colorspaceRef,
                                                     kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
        
        // Draw the image into the context and retrieve the new bitmap image without alpha
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
        CGImageRef imageRefWithoutAlpha = CGBitmapContextCreateImage(context);
        UIImage *imageWithoutAlpha = [UIImage imageWithCGImage:imageRefWithoutAlpha
                                                         scale:image.scale
                                                   orientation:image.imageOrientation];
```



