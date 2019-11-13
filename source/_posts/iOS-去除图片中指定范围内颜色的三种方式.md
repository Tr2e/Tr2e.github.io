---
title: iOS-去除图片中指定范围内颜色的三种方式
date: 2017-12-20 15:19:18
tags: 'iOS'
categories: 'iOS'
---

**实际项目场景：去除图片的纯白色背景图，获得一张透明底图片用于拼图功能**

介绍两种途径的三种处理方式(不知道为啥想起了孔乙己)，具体性能鶸并未对比，如果有大佬能告知，不胜感激。

* Core Image
* Core Graphics/Quarz 2D

<!--more-->

## Core Image

> **Core Image**是一个很强大的框架。它可以让你简单地应用各种滤镜来处理图像，比如修改鲜艳程度,色泽,或者曝光。 它利用GPU（或者CPU）来非常快速、甚至实时地处理图像数据和视频的帧。并且隐藏了底层图形处理的所有细节，通过提供的API就能简单的使用了，无须关心OpenGL或者OpenGL ES是如何充分利用GPU的能力的，也不需要你知道GCD在其中发挥了怎样的作用，Core Image处理了全部的细节。 

![Chroma Key Filter](https://upload-images.jianshu.io/upload_images/1742463-9a7f44ff8c2bb971.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

在苹果官方文档[Core Image Programming Guide](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_intro/ci_intro.html#//apple_ref/doc/uid/TP30001185-CH1-TPXREF101)中，提到了[Chroma Key Filter Recipe](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_filer_recipes/ci_filter_recipes.html#//apple_ref/doc/uid/TP30001185-CH4-SW2)对于处理背景的范例

其中使用了[HSV](https://baike.baidu.com/item/HSV/547122?fr=aladdin)颜色模型，因为HSV模型，对于颜色范围的表示，相比RGB更加友好。

**大致过程处理过程：**

1. 创建一个映射希望移除颜色值范围的立方体贴图cubeMap，将目标颜色的`Alpha`置为`0.0f`
2. 使用`CIColorCube`滤镜和cubeMap对源图像进行颜色处理
3. 获取到经过`CIColorCube`处理的`Core Image`对象`CIImage`,转换为`Core Graphics`中的`CGImageRef`对象，通过`imageWithCGImage:`获取结果图片

**注意**：第三步中，不可以直接使用`imageWithCIImage:`,因为得到的并不是一个标准的`UIImage`，如果直接拿来用，会出现不显示的情况。


```
- (UIImage *)removeColorWithMinHueAngle:(float)minHueAngle maxHueAngle:(float)maxHueAngle image:(UIImage *)originalImage{
    CIImage *image = [CIImage imageWithCGImage:originalImage.CGImage];
    CIContext *context = [CIContext contextWithOptions:nil];// kCIContextUseSoftwareRenderer : CPURender
    /** 注意
     *  UIImage 通过CIimage初始化，得到的并不是一个通过类似CGImage的标准UIImage
     *  所以如果不用context进行渲染处理，是没办法正常显示的
     */
    CIImage *renderBgImage = [self outputImageWithOriginalCIImage:image minHueAngle:minHueAngle maxHueAngle:maxHueAngle];
    CGImageRef renderImg = [context createCGImage:renderBgImage fromRect:image.extent];
    UIImage *renderImage = [UIImage imageWithCGImage:renderImg];
    return renderImage;
}

struct CubeMap {
    int length;
    float dimension;
    float *data;
};

- (CIImage *)outputImageWithOriginalCIImage:(CIImage *)originalImage minHueAngle:(float)minHueAngle maxHueAngle:(float)maxHueAngle{
    
    struct CubeMap map = createCubeMap(minHueAngle, maxHueAngle);
    const unsigned int size = 64;
    // Create memory with the cube data
    NSData *data = [NSData dataWithBytesNoCopy:map.data
                                        length:map.length
                                  freeWhenDone:YES];
    CIFilter *colorCube = [CIFilter filterWithName:@"CIColorCube"];
    [colorCube setValue:@(size) forKey:@"inputCubeDimension"];
    // Set data for cube
    [colorCube setValue:data forKey:@"inputCubeData"];
    
    [colorCube setValue:originalImage forKey:kCIInputImageKey];
    CIImage *result = [colorCube valueForKey:kCIOutputImageKey];
    
    return result;
}

struct CubeMap createCubeMap(float minHueAngle, float maxHueAngle) {
    const unsigned int size = 64;
    struct CubeMap map;
    map.length = size * size * size * sizeof (float) * 4;
    map.dimension = size;
    float *cubeData = (float *)malloc (map.length);
    float rgb[3], hsv[3], *c = cubeData;
    
    for (int z = 0; z < size; z++){
        rgb[2] = ((double)z)/(size-1); // Blue value
        for (int y = 0; y < size; y++){
            rgb[1] = ((double)y)/(size-1); // Green value
            for (int x = 0; x < size; x ++){
                rgb[0] = ((double)x)/(size-1); // Red value
                rgbToHSV(rgb,hsv);
                // Use the hue value to determine which to make transparent
                // The minimum and maximum hue angle depends on
                // the color you want to remove
                float alpha = (hsv[0] > minHueAngle && hsv[0] < maxHueAngle) ? 0.0f: 1.0f;
                // Calculate premultiplied alpha values for the cube
                c[0] = rgb[0] * alpha;
                c[1] = rgb[1] * alpha;
                c[2] = rgb[2] * alpha;
                c[3] = alpha;
                c += 4; // advance our pointer into memory for the next color value
            }
        }
    }
    map.data = cubeData;
    return map;
}
```
`rgbToHSV`在官方文档中并没有提及，笔者在下文中提到的大佬的博客中找到了相关转换处理。**感谢**
```
void rgbToHSV(float *rgb, float *hsv) {
    float min, max, delta;
    float r = rgb[0], g = rgb[1], b = rgb[2];
    float *h = hsv, *s = hsv + 1, *v = hsv + 2;
    
    min = fmin(fmin(r, g), b );
    max = fmax(fmax(r, g), b );
    *v = max;
    delta = max - min;
    if( max != 0 )
        *s = delta / max;
    else {
        *s = 0;
        *h = -1;
        return;
    }
    if( r == max )
        *h = ( g - b ) / delta;
    else if( g == max )
        *h = 2 + ( b - r ) / delta;
    else
        *h = 4 + ( r - g ) / delta;
    *h *= 60;
    if( *h < 0 )
        *h += 360;
}
```

接下来我们试一下，去除绿色背景的效果如何
![](https://upload-images.jianshu.io/upload_images/1742463-102b639c4903fc34.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/533/format/webp)

我们可以通过使用[HSV工具](http://www.color-blindness.com/color-name-hue/),确定绿色`HUE`值的大概范围为**50-170**

调用一下方法试一下
```
[[SPImageChromaFilterManager sharedManager] removeColorWithMinHueAngle:50 maxHueAngle:170 image:[UIImage imageWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"nb" ofType:@"jpeg"]]]
```
效果

![](https://upload-images.jianshu.io/upload_images/1742463-6f5f22936b3f476d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/584/format/webp)

效果还可以的样子。

如果认真观察HSV模型的同学也许会发现，我们通过指定色调角度（Hue）的方式，对于指定灰白黑显得无能为力。我们不得不去用饱和度（Saturation）和明度（Value）去共同判断，感兴趣的同学可以在代码中*判断Alpha*`float alpha = (hsv[0] > minHueAngle && hsv[0] < maxHueAngle) ? 0.0f: 1.0f;`那里试一下效果。（至于代码中为啥RGB和HSV这么转换，请百度他们的转换，因为鶸笔者也不懂。哎，鶸不聊生）

对于**Core Image**感兴趣的同学，请移步大佬的系列文章
> [iOS8 Core Image In Swift：自动改善图像以及内置滤镜的使用](http://blog.csdn.net/zhangao0086/article/details/39012231)
[iOS8 Core Image In Swift：更复杂的滤镜](http://blog.csdn.net/zhangao0086/article/details/39120331)
[iOS8 Core Image In Swift：人脸检测以及马赛克](http://blog.csdn.net/zhangao0086/article/details/39253707)
[iOS8 Core Image In Swift：视频实时滤镜](http://blog.csdn.net/zhangao0086/article/details/39433519)

## Core Graphics/Quarz 2D
上文中提到的基于`OpenGl`的`Core Image`显然功能十分强大，作为视图另一基石的[Core Graphics](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html)同样强大。对他的探究，让鶸笔者更多的了解到图片的相关知识。所以在此处总结，供日后查阅。

如果对探究不感兴趣的同学，请直接跳到文章最后 **Masking an Image with Color** 部分

### Bitmap

![侵删](https://upload-images.jianshu.io/upload_images/1742463-234e9fe782d87822.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500/format/webp)
在[Quarz 2D官方文档中，对于BitMap有如下描述](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/dq_images/dq_images.html#//apple_ref/doc/uid/TP30001066-CH212-SW3)：
>A bitmap image (or sampled image) is an array of pixels (or samples). Each pixel represents a single point in the image. JPEG, TIFF, and PNG graphics files are examples of bitmap images. 

> `32-bit and 16-bit pixel formats for CMYK and RGB color spaces in Quartz 2D`
![32-bit and 16-bit pixel formats for CMYK and RGB color spaces in Quartz 2D](https://upload-images.jianshu.io/upload_images/1742463-b3ec1da17d86370c.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/480/format/webp)

回到我们的需求，对于去除图片中的指定颜色，如果我们能够读取到每个像素上的**RGBA**信息，分别判断他们的值，如果符合目标范围，我们将他的**Alpha**值改为**0**，然后输出成新的图片，那么我们就实现了类似上文中**cubeMap**的处理方式。

强大的`Quarz 2D`为我们提供了实现这种操作的能力，下面请看代码示例：
```
- (UIImage *)removeColorWithMaxR:(float)maxR minR:(float)minR maxG:(float)maxG minG:(float)minG maxB:(float)maxB minB:(float)minB image:(UIImage *)image{
    // 分配内存
    const int imageWidth = image.size.width;
    const int imageHeight = image.size.height;
    size_t bytesPerRow = imageWidth * 4;
    uint32_t* rgbImageBuf = (uint32_t*)malloc(bytesPerRow * imageHeight);
    
    // 创建context
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();// 色彩范围的容器
    CGContextRef context = CGBitmapContextCreate(rgbImageBuf, imageWidth, imageHeight, 8, bytesPerRow, colorSpace,kCGBitmapByteOrder32Little | kCGImageAlphaNoneSkipLast);
    CGContextDrawImage(context, CGRectMake(0, 0, imageWidth, imageHeight), image.CGImage);
    
    
    // 遍历像素
    int pixelNum = imageWidth * imageHeight;
    uint32_t* pCurPtr = rgbImageBuf;
    for (int i = 0; i < pixelNum; i++, pCurPtr++)
    {
        uint8_t* ptr = (uint8_t*)pCurPtr;
        if (ptr[3] >= minR && ptr[3] <= maxR &&
            ptr[2] >= minG && ptr[2] <= maxG &&
            ptr[1] >= minB && ptr[1] <= maxB) {
            ptr[0] = 0;
        }else{
            printf("\n---->ptr0:%d ptr1:%d ptr2:%d ptr3:%d<----\n",ptr[0],ptr[1],ptr[2],ptr[3]);
        }
    }
    // 将内存转成image
    CGDataProviderRef dataProvider =CGDataProviderCreateWithData(NULL, rgbImageBuf, bytesPerRow * imageHeight, nil);
    CGImageRef imageRef = CGImageCreate(imageWidth, imageHeight,8, 32, bytesPerRow, colorSpace,kCGImageAlphaLast |kCGBitmapByteOrder32Little, dataProvider,NULL,true,kCGRenderingIntentDefault);
    CGDataProviderRelease(dataProvider);
    UIImage* resultUIImage = [UIImage imageWithCGImage:imageRef];
    
    // 释放
    CGImageRelease(imageRef);
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    return resultUIImage;
}
```
还记得我们在**Core Image**中提到的HSV模式的弊端吗？那么**Quarz 2D**则是直接利用**RGBA**的信息进行处理，很好的规避了对黑白色不友好的问题，我们只需要设置一下RGB的范围即可（因为黑白色在RGB颜色模式中，很好确定），我们可以大致封装一下。如下
```
- (UIImage *)removeWhiteColorWithImage:(UIImage *)image{
    return [self removeColorWithMaxR:255 minR:250 maxG:255 minG:240 maxB:255 minB:240 image:image];
}
```
```
- (UIImage *)removeBlackColorWithImage:(UIImage *)image{
    return [self removeColorWithMaxR:15 minR:0 maxG:15 minG:0 maxB:15 minB:0 image:image];
}
```
看一下我们对于白色背景的处理效果对比

![](https://upload-images.jianshu.io/upload_images/1742463-d2ae24ffc0bbd821.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

看起来似乎还不错，但是对于纱质的衣服，就显得很不友好。看一下笔者做的几组图片的测试

![](https://upload-images.jianshu.io/upload_images/1742463-5ef64c988f44a14c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

很显然，如果不是白色背景，“衣衫褴褛”的效果非常明显。这个问题，在笔者尝试的三种方法中，无一幸免，如果哪位大佬知道好的处理方法，而且能告诉鶸，将不胜感激。（先放俩膝盖在这儿）

除了上述问题外，这种对比每个像素的方法，读取出来的数值会同作图时出现误差。但是这种误差肉眼基本不可见。

![](https://upload-images.jianshu.io/upload_images/1742463-2d5b440542eb2e1e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)
如下图中，我们作图时，设置的**RGB**值分别为100/240/220 但是通过CG上述处理时，读取出来的值则为92/241/220。对比图中的“新的”“当前”，基本看不出色差。这点小问题各位知道就好，对实际去色效果影响并不大
![](https://upload-images.jianshu.io/upload_images/1742463-b885efb39fed068b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

### Masking an Image with Color
笔者尝试过理解并使用上一种方法后，在重读文档时发现了这个方法，简直就像是发现了**Father Apple**的恩赐。直接上代码
```
- (UIImage *)removeColorWithMaxR:(float)maxR minR:(float)minR maxG:(float)maxG minG:(float)minG maxB:(float)maxB minB:(float)minB image:(UIImage *)image{

    const CGFloat myMaskingColors[6] = {minR, maxR,  minG, maxG, minB, maxB};
    CGImageRef ref = CGImageCreateWithMaskingColors(image.CGImage, myMaskingColors);
    return [UIImage imageWithCGImage:ref];
    
}
```
[官方文档点这儿](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/dq_images/dq_images.html#//apple_ref/doc/uid/TP30001066-CH212-CJBHCADE)

## 总结
HSV颜色模式相对于RGB模式而言，更利于我们抠除图片中的彩色，而RGB则正好相反。笔者因为项目中，只需要去除白色背景，所以最终采用了最后一种方式。



