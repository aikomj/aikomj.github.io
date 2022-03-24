---
layout: post
title: 1M图片压缩优化到100kb
category: java
tags: [java]
keywords: java
excerpt: JDK原生API压缩图像实战与其他开源库介绍
lock: noneed
---

以下文章来源于沉默王二 ，作者沉默王二

## 1、图像压缩

图像压缩是数据压缩技术在数字图像上的应用，目的是减少图像数据中的冗余信息，从而用更加高效的格式存储和传输数据。图像压缩可以是有损数据压缩，也可以是无损数据压缩。

<img src="/assets/images/2022/compress-image-1.jpg" style="zoom:50%;" />

<img src="/assets/images/2022/compress-image-2.jpg" style="zoom:67%;" />

更多关于图像压缩的资料可参考以下链接:

[机器之心：https://www.jiqizhixin.com/graph/technologies/08b2b25e-21a0-48e1-9de8-f91d424adfe1](https://www.jiqizhixin.com/graph/technologies/08b2b25e-21a0-48e1-9de8-f91d424adfe1)

## 2、图像压缩实战

数字图像处理（Digital Image Processing）是通过计算机对图像进行去除噪声、增强、复原、分割、提取特征等处理的方法和技术。

![](/assets/images/2022/compress-image-3.jpg)

输入的是图像信号，然后经过 DIP 进行有效的算法处理后，输出为数字信号。

为了压缩图像，我们需要读取图像并将其转换成 BufferedImage 对象，BufferedImage 是 Image 类的一个子类，描述了一个具有可访问的图像数据缓冲区，由 ColorModel 和 Raster 的图像数据组成。

刚好我本地有一张之前用过的封面图，离 1M 只差 236 KB，可以拿来作为测试用。

![](/assets/images/2022/compress-image-5.jpg)

这其中要用到 ImageIO 类，这是一个静态类，提供了一系列方法用来读和写图像，同时还可以对图像进行简单的编码和解码。

```java
// 将图像读取到 BufferedImage 对象
File input = new File("ceshi.jpg");
BufferedImage image = ImageIO.read(input);
```

比如说通过 `ImageIO.getImageWritersByFormatName()` 可以返回一个Iterator，其中包含了通过命名格式对图像进行编码的 ImageWriter

```java
Iterator<ImageWriter> writers =  ImageIO.getImageWritersByFormatName("jpg");
ImageWriter writer = (ImageWriter) writers.next();
```

比如说通过 `ImageIO.createImageOutputStream()` 可以创建一个图像的输出流对象，有了该对象后就可以通过 `ImageWriter.setOutput()` 将其设置为输出流。

```java
File compressedImageFile = new File("bbcompress.jpg");
OutputStream os =new FileOutputStream(compressedImageFile);
ImageOutputStream ios = ImageIO.createImageOutputStream(os);
writer.setOutput(ios);
```

我们对ImageWriter 进行一些参数配置，比如说压缩模式，压缩质量等等

```java
ImageWriteParam param = writer.getDefaultWriteParam();

param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
param.setCompressionQuality(0.01f);
```

压缩模式一共有四种，MODE_EXPLICIT 是其中一种，表示 ImageWriter 可以根据后续的 set 的附加信息进行平铺和压缩，比如说接下来的 `setCompressionQuality()` 方法，方法的参数是一个 0-1 之间的数，0.0 表示尽最大程度压缩，1.0 表示保证图像质量很重要。对于有损压缩方案，压缩质量应该控制文件大小和图像质量之间的权衡（例如，通过在写入 JPEG 图像时选择量化表）。对于无损方案，压缩质量可用于控制文件大小和执行压缩所需的时间之间的权衡（例如，通过优化行过滤器并在写入 PNG 图像时设置 ZLIB 压缩级别）。

整体代码如下：

```java
public class Demo {
    public static void main(String[] args) {

        try {
            File input = new File("ceshi.jpg");
            BufferedImage image = ImageIO.read(input);


            Iterator<ImageWriter> writers = ImageIO.getImageWritersByFormatName("jpg");
            ImageWriter writer = (ImageWriter) writers.next();

            File compressedImageFile = new File("bbcompress.jpg");
            OutputStream os = new FileOutputStream(compressedImageFile);
            ImageOutputStream ios = ImageIO.createImageOutputStream(os);
            writer.setOutput(ios);


            ImageWriteParam param = writer.getDefaultWriteParam();

            param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
            param.setCompressionQuality(0.01f);

            writer.write(null, new IIOImage(image, null, null), param);

            os.close();
            ios.close();
            writer.dispose();

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

执行压缩后，可以看到图片的大小压缩到了 19 KB：

![](/assets/images/2022/compress-image-6.jpg)

可以看得出，质量因子为 0.01f 的时候图片已经有些失真了，可以适当提高质量因子比如说 0.5f，再来看一下

![](/assets/images/2022/compress-image-7.jpg)

图片质量明显提高了，但大小依然只有 64 KB，压缩效果还是值得信赖的。

## 3、其他开源库

接下来，推荐一些可以轻松集成到项目中的图像处理库吧，它们全都是免费的。

1）ImageJ，用 Java 编写的，可以编辑、分析、处理、保存和打印图像。

<img src="/assets/images/2022/java/compress-image.jpg" style="zoom:67%;" />

2 )Apache Commons Imaging，一个读取和写入各种图像格式的库，包括快速解析图像信息（如大小，颜色，空间，ICC配置文件等）和元数据

![](/assets/images/2022/java/compress-image-2.jpg)

4）OpenCV，由BSD许可证发布，可以免费学习和商业使用，提供了包括 C/C++、Python 和 Java 等主流编程语言在内的接口。OpenCV 专为计算效率而设计，强调实时应用，可以充分发挥多核处理器的优势。

![](/assets/images/2022/java/compress-image-3.jpg)

这里就以 OpenCV 为例，来演示一下图像压缩。当然了，OpenCV 用来压缩图像属于典型的大材小用。

第一步，添加 OpenCV 依赖到我们的项目当中，以 Maven 为例。

```xml
<dependency>
 <groupId>org.openpnp</groupId>
 <artifactId>opencv</artifactId>
 <version>4.5.1-2</version>
</dependency>
```

第二步，要想使用 OpenCV，需要先初始化。

```java
OpenCV.loadShared();
```

第三步，使用 OpenCV 读取图片。

```java
Mat src = Imgcodecs.imread(imagePath);
```

第四步，使用OpenCV压缩图片

```java
MatOfInt dstImage = new MatOfInt(Imgcodecs.IMWRITE_JPEG_QUALITY, 1);
Imgcodecs.imwrite("resized_image.jpg", sourceImage, dstImage);
```

MatOfInt 的构造参数是一个可变参数，第一个参数 IMWRITE_JPEG_QUALITY 表示对图片的质量进行改变，第二个是质量因子，1-100，值越大表示质量越高。

借这个机会，来对比下 OpenCV 和 JDK 原生 API 在压缩图像时所使用的时间。

这是我本机的配置情况，早年买的顶配 iMac，也是我的主力机。一开始只有 16 G 内存，后来加了一个 16 G 内存条，不过最近半年电脑突然死机重启的频率明显提高了，不知道是不是 Big Sur 这个操作系统的问题还是电脑硬件老了。

![](/assets/images/2022/java/compress-image-4.jpg)

结果如下：

```java
opencvCompress压缩完成，所花时间：1070
jdkCompress压缩完成，所花时间：322
```

压缩后的图片大小差不多，都是 19 KB，并且质量因子都是最低值。

![](/assets/images/2022/java/compress-image-5.jpg)



