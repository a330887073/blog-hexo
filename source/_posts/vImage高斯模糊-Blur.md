title: vImage高斯模糊(Blur)
date: 2016-04-12 16:57:09
tags: [iOS,UIImage,Blur]
author: Kael
---


本文主要内容:
1.用vImage来做`实时`高斯模糊
2.遇到的坑
3.爬坑
<!-- more -->

  在iOS7以后,半透明模糊效果在系统中大量使用,不仅在iPhone上,Mac上也随处可见这种效果.在iOS上实现这种效果的方法很多,不同的框架(CoreImage,GPUImage...)和各种扩展(UIVisualEffectView)提供了不同的方式方法.
  具体可以查看stackoverflow上这个问题:
  >[http://stackoverflow.com/questions/17041669/creating-a-blurring-overlay-view](http://stackoverflow.com/questions/17041669/creating-a-blurring-overlay-view)

## 用vImage来做`实时`高斯模糊
这是[http://stackoverflow.com/questions/17041669/creating-a-blurring-overlay-view](http://stackoverflow.com/questions/17041669/creating-a-blurring-overlay-view)问题的回答中代码的一点修改
```objc

//#import <Accelerate/Accelerate.h>

-(UIImage *)boxblurImageWithBlur:(CGFloat)blur oImg:(UIImage *)oImg{
    if (blur < 0.f || blur > 1.f) {
        blur = 0.5f;
    }
    int boxSize = (int)(blur * 50);
    boxSize = boxSize - (boxSize % 2) + 1;
    CGImageRef img = oImg.CGImage;
    vImage_Buffer inBuffer, outBuffer;
    vImage_Error error;
    void *pixelBuffer;
    CGDataProviderRef inProvider = CGImageGetDataProvider(img);
    CFDataRef inBitmapData = CGDataProviderCopyData(inProvider);
    inBuffer.width = CGImageGetWidth(img);
    inBuffer.height = CGImageGetHeight(img);
    inBuffer.rowBytes = CGImageGetBytesPerRow(img);
    inBuffer.data = (void*)CFDataGetBytePtr(inBitmapData);
    pixelBuffer = malloc(CGImageGetBytesPerRow(img) * CGImageGetHeight(img));
    if(pixelBuffer == NULL)
        NSLog(@"No pixelbuffer");
    outBuffer.data = pixelBuffer;
    outBuffer.width = CGImageGetWidth(img);
    outBuffer.height = CGImageGetHeight(img);
    outBuffer.rowBytes = CGImageGetBytesPerRow(img);
    error = vImageBoxConvolve_ARGB8888(&inBuffer, &outBuffer, NULL, 0, 0, boxSize, boxSize, NULL, kvImageEdgeExtend);
    if (error) {
        NSLog(@"JFDepthView: error from convolution %ld", error);
    }
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGContextRef ctx = CGBitmapContextCreate(outBuffer.data,
                                             outBuffer.width,
                                             outBuffer.height,
                                             8,
                                             outBuffer.rowBytes,
                                             colorSpace,
                                             kCGImageAlphaNoneSkipLast);
    CGImageRef imageRef = CGBitmapContextCreateImage (ctx);
    UIImage *returnImage = [UIImage imageWithCGImage:imageRef];
    
    //clean up
    CGContextRelease(ctx);
    CGColorSpaceRelease(colorSpace);
    
    free(pixelBuffer);
    CFRelease(inBitmapData);
    
    CGImageRelease(imageRef);
    
    return returnImage;
}
```
赶紧搞个本地图片,写个小Demo测试下

[demo1](https://github.com/Sdoy/vImageBlur/blob/master/vImageBlur/TestDemoViewController.m#L30)
```objc
UIImageView *view1=[[UIImageView alloc]initWithImage:_image1];
_view1.image=[self boxblurImageWithBlur:slider.value oImg:_image1];
```

效果图:
![](https://img.alicdn.com/imgextra/i4/373400920/TB2h8UmmVXXXXcJXXXXXXXXXXXX_!!373400920.gif)

恩,效果相当不错

## 然后坑来了,

把本地图上传到图床,然后用SDWebImage下载之后再进行模糊

[demo2](https://github.com/Sdoy/vImageBlur/blob/master/vImageBlur/TestDemoViewController.m#L52)
```objc
[view2 sd_setImageWithURL:[NSURL URLWithString:@"https://img.alicdn.com/imgextra/i3/373400920/TB2KIkfmVXXXXbYXXXXXXXXXXXX_!!373400920.png"] completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
        _image2=image;
    }];
...

_view2.image=[self boxblurImageWithBlur:slider.value oImg:_image2];


```

效果图:
![](https://img.alicdn.com/imgextra/i2/373400920/TB2BuD6mVXXXXcwXpXXXXXXXXXX_!!373400920.gif)

第一张是本地图,第二张是从网上下载的图
简直,不忍直视....

## 脱坑
哪有问题?

猜想:是不是下载的文件格式变了,PNG变JPG这种.
用下面的代码来测试
[-(NSString *)typeForImageData:(NSData *)data](https://github.com/Sdoy/vImageBlur/blob/master/vImageBlur/TestDemoViewController.m#L93)
```objc
 NSLog(@"image1-%@",[self typeForImageData:UIImagePNGRepresentation(_image1)]);
 NSLog(@"image2-%@",[self typeForImageData:UIImagePNGRepresentation(_image2)]);
```
输出:image1-image/png
    image2-image/png

然而格式并没有变

猜想:跟图片的Alpha有关系么.
所以来看看图片中的exif有什么不同吧
用下面的代码
[-(void)readExif:(UIImage *)image](https://github.com/Sdoy/vImageBlur/blob/master/vImageBlur/TestDemoViewController.m#L82)
```objc
[self readExif:_image1];
[self readExif:_image2];
```
从输出的结果中发现
\_image2的信息用多了一个字段HasAlpha
貌似图床把图片的Alpha通道打开了
![](https://img.alicdn.com/imgextra/i4/373400920/TB28hj7mVXXXXcOXpXXXXXXXXXX_!!373400920.png)

在高斯模糊的方法中[CGBitmapContextCreate](https://github.com/Sdoy/vImageBlur/blob/master/vImageBlur/TestDemoViewController.m#L138)函数的最后一个参数uint32_t bitmapInfo跟Alpha有点关系,从命名中就可以知道

```objc
typedef CF_ENUM(uint32_t, CGImageAlphaInfo) {
  kCGImageAlphaNone,               /* For example, RGB. */
  kCGImageAlphaPremultipliedLast,  /* For example, premultiplied RGBA */
  kCGImageAlphaPremultipliedFirst, /* For example, premultiplied ARGB */
  kCGImageAlphaLast,               /* For example, non-premultiplied RGBA */
  kCGImageAlphaFirst,              /* For example, non-premultiplied ARGB */
  kCGImageAlphaNoneSkipLast,       /* For example, RBGX. */
  kCGImageAlphaNoneSkipFirst,      /* For example, XRGB. */
  kCGImageAlphaOnly                /* No color data, alpha data only */
};

```
逐对之后一个参数一个个替换测试,当测试到kCGImageAlphaNoneSkipFirst时,
效果是这样的:
![](https://img.alicdn.com/imgextra/i3/373400920/TB2VevYmVXXXXasXFXXXXXXXXXX_!!373400920.gif)

有Alpha通道的变成正常的了,本地图片又坑爹了,
反复测试之后:

## 总结出了这样的结果:
1.如果图片没有Alpha通道,高斯模糊在创建图片的时候(CGBitmapContextCreate()方法),应该设置成kCGImageAlphaNoneSkipLast
2.如果图片有Alpha通道,高斯模糊在创建图片的时候(CGBitmapContextCreate()方法),应该设置成kCGImageAlphaNoneSkipFirst

那么我在blur时候指定bitmapInfo就好了.
(还有一个思路就是直接改UIImage的exif信息,我并没有尝试)

然后写了个UIImage的扩展类
[UIImage+Blur](https://github.com/Sdoy/vImageBlur/blob/master/vImageBlur/Blur/UIImage%2BBlur.m)


最后效果:
本地图片和远程图片都正常了
![](https://img.alicdn.com/imgextra/i3/373400920/TB2Bb.bmVXXXXcfXpXXXXXXXXXX_!!373400920.gif)

项目地址:[SampleCode](https://github.com/Sdoy/vImageBlur)

