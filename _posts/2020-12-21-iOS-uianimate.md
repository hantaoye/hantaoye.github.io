---
title:  "iOS UI动画相关"
date:   2020-12-21 22:11:36 +0530
author: taoye
categories: [iOS, 2020年复现总结]
tags: [iOS]
---

### UIview动画，从，到展示，经历了哪些过程

![04403867b86ebea1fb731dc3d66cbca5](/assets/img/ui-animate/9BAE3186-11F1-42BF-AFDD-B8EA560819D9.png)

下图是 Core Animation 的单次渲染流水线，也就是一帧动画的渲染过程：
Handle Events：这个过程中会先处理点击事件，这个过程中有可能会需要改变页面的布局和界面层次。
Commit Transaction：此时 app 会通过&nbsp;CPU&nbsp;处理显示内容的前置计算，比如布局计算、图片解码等任务，接下来会进行详细的讲解。之后将计算好的图层进行打包发给 Render Server。
Render Server - Decode：打包好的图层被传输到 Render Server 之后，首先会进行解码。注意完成解码之后需要等待下一个 RunLoop 才会执行下一步 Draw Calls。
Render Server - Draw Calls：解码完成后，Core Animation 会调用下层渲染框架的方法进行绘制，进而调用到&nbsp;GPU。
GPU - Render：这一阶段主要由&nbsp;GPU&nbsp;进行渲染。
Display：显示阶段，需要等 render 结束的下一个 RunLoop 触发显示。
总而言之，每一帧动画的渲染对于 CPU 和 GPU 都有一定的消耗，尤其是对 GPU 的性能占用较大。

### iOS 中的渲染框架

![906c7afad67d67706f507b3dadb122c0](/assets/img/ui-animate/BC7C35D9-1C2F-488D-BA22-9352CD9C3C61.png)
Rendering Pass： Render Server 的具体操作
![e7fbaa942ccd09603997ea89e1e4b05b](/assets/img/ui-animate/E0607CE0-792D-42D8-8FCF-7835993FC2A0.png)

Render Server 通常是 OpenGL 或者是 Metal。以 OpenGL 为例，那么上图主要是 GPU 中执行的操作，具体主要包括：
GPU 收到 Command Buffer，包含图元 primitives 信息Tiler 
开始工作：先通过顶点着色器 Vertex Shader 对顶点进行处理，更新图元信息
平铺过程：平铺生成 tile bucket 的几何图形，这一步会将图元信息转化为像素，之后将结果写入 Parameter Buffer 中Tiler 更新完所有的图元信息，或者 Parameter Buffer 已满，则会开始下一步Renderer
工作：将像素信息进行处理得到 bitmap，之后存入 Render Buffer

Render Buffer 中存储有渲染好的 bitmap，供之后的 Display 操作使用


### 大图加载
1.苹果官方demo提供的分片比例裁剪方式;
用CGImageCreateWithImageInRect截取原图对应位置的内容,再通过CGContextDrawImage渲染到指定位置;
    - 把图片缩小
    - 利用CGBitmapContextCreate分块绘制 （ 拆分多块内存
    - CGContextRef合成图片、最后才加载
    - CGBitmapContextCreate方法生成一张比例缩放的位图。
    - CGImageCreateWithImageInRect根据计算的rect获取到图片数据。
    - CGContextDrawImage根据计算的rect把获取到的图片数据绘制到位图上。
    - CGBitmapContextCreateImage绘制完毕获取到图片显示。


``` 
缩略图
    func downsample(imageAt imageURL: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage
    {
        let sourceOpt = [kCGImageSourceShouldCache : false] as CFDictionary
        // 其他场景可以用createwithdata (data并未decode,所占内存没那么大),
        let source = CGImageSourceCreateWithURL(imageURL as CFURL, sourceOpt)!
        
        let maxDimension = max(pointSize.width, pointSize.height) * scale
        let downsampleOpt = [kCGImageSourceCreateThumbnailFromImageAlways : true,
                             kCGImageSourceShouldCacheImmediately : true ,
                             kCGImageSourceCreateThumbnailWithTransform : true,
                             kCGImageSourceThumbnailMaxPixelSize : maxDimension] as CFDictionary
        let downsampleImage = CGImageSourceCreateThumbnailAtIndex(source, 0, downsampleOpt)!
        return UIImage(cgImage: downsampleImage)
    }
```

2. 利用CATiledLayer层级的API,自动进行绘制;CATiledLayer是苹果提供的API,实现细节都在内部处理,所以显得会流畅许多,观察发现它利用多个线程进行绘制,
    - CATiledLayer存在于Core Animation框架中.
    - 继承于CALayer
    - 是一个异步加载提供内存切片的图层。
    - 创建一个自定义view,将图层设置为 CATiledLayer
    - 传入源图片、scale ( scale用于设置实际显示frame
    - 设置 CATiledLayer 相关参数
    - 利用drawRect: 截取图片、绘制渲染


当你用 UIImage 或 CGImageSource 的那几个方法创建图片时，图片数据并不会立刻解码。图片设置到 UIImageView 或者 CALayer.contents 中去，并且 CALayer 被提交到 GPU 前，CGImage 中的数据才会得到解码。这一步是发生在主线程的，并且不可避免。如果想要绕开这个机制，常见的做法是在后台线程先把图片绘制到 CGBitmapContext 中，然后从 Bitmap 直接创建图片。目前常见的网络图片库都自带这个功能。



### 图片解压缩卡顿问题
解压缩会带来一些性能问题。但是如果我们给imageView提供的是解压缩后的位图那么系统就不会再进行解压缩操作。这种方式也是SDWebImage和YYWebImage的实现方式。具体解压缩的原理就是CGBitmapContextCreate方法重新生产一张位图然后把图片绘制当这个位图上，最后拿到的图片就是解压缩之后的图片。

YY 源码、 sd源码类似
```
CGImageRef YYCGImageCreateDecodedCopy(CGImageRef imageRef, BOOL decodeForDisplay) {
    if (!imageRef) return NULL;
    size_t width = CGImageGetWidth(imageRef);
    size_t height = CGImageGetHeight(imageRef);
    if (width == 0 || height == 0) return NULL;
    
    if (decodeForDisplay) { //decode with redraw (may lose some precision)
        CGImageAlphaInfo alphaInfo = CGImageGetAlphaInfo(imageRef) & kCGBitmapAlphaInfoMask;
        BOOL hasAlpha = NO;
        if (alphaInfo == kCGImageAlphaPremultipliedLast ||
            alphaInfo == kCGImageAlphaPremultipliedFirst ||
            alphaInfo == kCGImageAlphaLast ||
            alphaInfo == kCGImageAlphaFirst) {
            hasAlpha = YES;
        }
        // BGRA8888 (premultiplied) or BGRX8888
        // same as UIGraphicsBeginImageContext() and -[UIView drawRect:]
        CGBitmapInfo bitmapInfo = kCGBitmapByteOrder32Host;
        bitmapInfo |= hasAlpha ? kCGImageAlphaPremultipliedFirst : kCGImageAlphaNoneSkipFirst;
        CGContextRef context = CGBitmapContextCreate(NULL, width, height, 8, 0, YYCGColorSpaceGetDeviceRGB(), bitmapInfo);
        if (!context) return NULL;
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef); // decode
        CGImageRef newImage = CGBitmapContextCreateImage(context);
        CFRelease(context);
        return newImage;
        
    } else {
        CGColorSpaceRef space = CGImageGetColorSpace(imageRef);
        size_t bitsPerComponent = CGImageGetBitsPerComponent(imageRef);
        size_t bitsPerPixel = CGImageGetBitsPerPixel(imageRef);
        size_t bytesPerRow = CGImageGetBytesPerRow(imageRef);
        CGBitmapInfo bitmapInfo = CGImageGetBitmapInfo(imageRef);
        if (bytesPerRow == 0 || width == 0 || height == 0) return NULL;
        
        CGDataProviderRef dataProvider = CGImageGetDataProvider(imageRef);
        if (!dataProvider) return NULL;
        CFDataRef data = CGDataProviderCopyData(dataProvider); // decode
        if (!data) return NULL;
        
        CGDataProviderRef newProvider = CGDataProviderCreateWithCFData(data);
        CFRelease(data);
        if (!newProvider) return NULL;
        
        CGImageRef newImage = CGImageCreate(width, height, bitsPerComponent, bitsPerPixel, bytesPerRow, space, bitmapInfo, newProvider, NULL, false, kCGRenderingIntentDefault);
        CFRelease(newProvider);
        return newImage;
    }
}

```

### 图片缓存
通过 imageNamed 创建 UIImage 时，系统实际上只是在 Bundle 内查找到文件名，然后把这个文件名放到 UIImage 里返回，并没有进行实际的文件读取和解码。当 UIImage 第一次显示到屏幕上时，其内部的解码方法才会被调用，同时解码结果会保存到一个全局缓存去。据我观察，在图片解码后，App 第一次退到后台和收到内存警告时，该图片的缓存才会被清空，其他情况下缓存会一直存在。

我要是用 imageWithData 能不能避免缓存呢？

不能。通过数据创建 UIImage 时，UIImage 底层是调用 ImageIO 的 CGImageSourceCreateWithData() 方法。该方法有个参数叫 ShouldCache，在 64 位的设备上，这个参数是默认开启的。这个图片也是同样在第一次显示到屏幕时才会被解码，随后解码数据被缓存到 CGImage 内部。与 imageNamed 创建的图片不同，如果这个图片被释放掉，其内部的解码数据也会被立刻释放。

怎么能避免缓存呢？

1. 手动调用 CGImageSourceCreateWithData() 来创建图片，并把 ShouldCache 和 ShouldCacheImmediately 关掉。这么做会导致每次图片显示到屏幕时，解码方法都会被调用，造成很大的 CPU 占用。
2. 把图片用 CGContextDrawImage() 绘制到画布上，然后把画布的数据取出来当作图片。这也是常见的网络图片库的做法。

我能直接取到图片解码后的数据，而不是通过画布取到吗？

1.CGImageSourceCreateWithData(data) 创建 ImageSource。
2.CGImageSourceCreateImageAtIndex(source) 创建一个未解码的 CGImage。
3.CGImageGetDataProvider(image) 获取这个图片的数据源。
4.CGDataProviderCopyData(provider) 从数据源获取直接解码的数据。
ImageIO 解码发生在最后一步，这样获得的数据是没有经过颜色类型转换的原生数据（比如灰度图像）。

### [图片](https://blog.ibireme.com/2015/11/02/ios_image_tips/)

怎样像浏览器那样边下载边显示图片？
第一种是 baseline，即逐行扫描。默认情况下，JPEG、PNG、GIF 都是这种保存方式。
第二种是 interlaced，即隔行扫描。PNG 和 GIF 在保存时可以选择这种格式。
第三种是 progressive，即渐进式。JPEG 在保存时可以选择这种方式。
在下载图片时，首先用 CGImageSourceCreateIncremental(NULL) 创建一个空的图片源，随后在获得新数据时调用
CGImageSourceUpdateData(data, false) 来更新图片源，最后在用 CGImageSourceCreateImageAtIndex() 创建图片来显示。


###  离屏渲染：

定义：如果要在显示屏上显示内容，我们至少需要一块与屏幕像素数据量一样大的frame buffer，作为像素数据存储区域，而这也是GPU存储渲染结果的地方。如果有时因为面临一些限制，无法把渲染结果直接写入frame buffer，而是先暂存在另外的内存区域，之后再写入frame buffer，那么这个过程被称之为离屏渲染。

所有CPU进行的光栅化操作（如文字渲染、图片解码），都无法直接绘制到由GPU掌管的frame buffer，只能暂时先放在另一块内存之中，说起来都属于“离屏渲染”。 （CGContext）

作为“画家”的GPU虽然可以一层一层往画布上进行输出，但是无法在某一层渲染完成之后，再回过头来擦除/改变其中的某个部分——因为在这一层之前的若干层layer像素数据，已经在渲染中被永久覆盖了。这就意味着，对于每一层layer，要么能找到一种通过单次遍历就能完成渲染的算法，要么就不得不另开一块内存，借助这个临时中转区域来完成一些更复杂的、多次的修改/剪裁操作。
[链接](https://zhuanlan.zhihu.com/p/72653360)


当视频控制器还未读取完成时，即屏幕内容刚显示一半时，GPU 将新的一帧内容提交到帧缓冲区并把两个缓冲区进行交换后，视频控制器就会把新的一帧数据的下半段显示到屏幕上，造成画面撕裂现象
为了解决这个问题，GPU 通常有一个机制叫做垂直同步（简写也是 V-Sync），当开启垂直同步后，GPU 会等待显示器的 VSync 信号发出后，才进行新的一帧渲染和缓冲区更新
在 VSync 信号到来后，系统图形服务会通过 CADisplayLink 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次 VSync 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 VSync 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。

性能消耗在哪里： 
1. 需要先将部分内容进行额外的渲染并保存到 Offscreen Buffer，然后在必须时刻进行offscreen buffer 和frame buffer切换，content切换消耗性能，
2. Offscreen buffer 本身占用空间， 且最大不超过屏幕总像素的2.5倍

### 为什么使用离屏渲染

1.一些特殊效果需要使用额外的 Offscreen Buffer 来保存渲染的中间状态，所以不得不使用离屏渲染。
2. 处于效率目的，可以将内容提前渲染保存在 Offscreen Buffer 中，达到复用的目的。

模糊效果，不采用系统提供的UIVisualEffect，而是另外实现模糊效果（CIGaussianBlur）


### 图层树

Layer也和View一样存在着一个层级树状结构,称之为图层树(Layer Tree),直接创建的或者通过UIView获得的(view.layer)用于显示的图层树,称之为模型树(Model Tree),模型树的背后还存在两份图层树的拷贝,一个是呈现树(Presentation Tree),一个是渲染树(Render Tree). 呈现树可以通过普通layer(其实就是模型树)的layer.presentationLayer获得,而模型树则可以通过modelLayer属性获得(详情文档).模型树的属性在其被修改的时候就变成了新的值,这个是可以用代码直接操控的部分;呈现树的属性值和动画运行过程中界面上看到的是一致的.而渲染树是私有的,你无法访问到,渲染树是对呈现树的数据进行渲染,为了不阻塞主线程,渲染的过程是在单独的进程或线程中进行的,所以你会发现Animation的动画并不会阻塞主线程.

![abbddf14976de78a71b70c4d3d4bd37f](/assets/img/ui-animate/2081841B-52CA-4326-BEBD-A13C727A5848.png)


CALayer的那些可用于动画的(Animatable)属性,称之为Animatable Properties. 如果一个Layer对象存在对应着的View,则称这个Layer是一个Root Layer,非Root Layer一般都是通过CALayer或其子类直接创建的.所有的非Root Layer在设置Animatable Properties的时候都存在着隐式动画,默认的duration是0.25秒.

    subLayer.position = CGPointMake(300,400);

像上面这段代码当下一个RunLoop开始的时候并不是直接将subLayer的position变成(300,400)的,而是有个移动的动画进行过渡完成的.

任何Layer的animatable属性的设置都应该属于某个CA事务(CATransaction),事务的作用是为了保证多个animatable属性的变化同时进行,不管是同一个layer还是不同的layer之间的.CATransaction也分两类,显式的和隐式的,当在某次RunLoop中设置一个animatable属性的时候,如果发现当前没有事务,则会自动创建一个CA事务,在线程的下个RunLoop开始时自动commit这个事务,如果在没有RunLoop的地方设置layer的animatable属性,则必须使用显式的事务.

### CALayer隐式动画
事务（coreAnimation监听runloop）0.25s

图层行为--CAAction
Core Animation使用动作对象为图层实现了隐式动画行为。动作对象服从CAAction协议并定义了一些运行于图层的相关行为。所有CAAnimation对象都实现了这个协议。无论何时，如果一个图层对象的属性发生变化，则这些动作对象将被分派执行。


 我们把改变属性时CALayer自动执行的动画称作行为，当CALayer的属性被修改时，它会调用-actionForKey:方法传递属性名称，我们可以找到这个方法的具体说明如下：
- (nullable id<CAAction>)actionForKey:(NSString *)event;

Core Animation以下面的顺序搜索动作对象：

如果图层有一个代理，并且代理实现了actionForLayer：forKey：方法，图层调用该方法。代理必须完成下面所述操作之一：
. 返回给定的键指定的动作对象
. 如果代理不处理动作则返回nil，而搜索操作将继续。
. 返回NSNull对象，这将引起搜索操作立即结束。

图层在图层的action字典内搜索给定的键
图层在style字典中查询一个包含键的动作字典。（换句话说，style字典包含一个actions键，它的值也是字典。图层在第二个字典中搜索给定的键。）
图层调用它的defaultActionForKey：类方法。
图层执行由Core Animation定义的隐式动作（如果有）。



每个UIView对它关联的图层都遵循了CALayerDelegate协议，并且实现了-actionForLayer:forKey方法。当不在一个动画块中修改动画属性时，UIView对所有图层行为都返回了NULL，但是在动画Block范围就返回了非空值
