#直播总结

##1.概述
关于直播的技术文章不少，成体系的不多。我们将用这篇文章，更系统化地介绍当下大热的视频直播各环节的关键技术，帮助视频直播创业者们更全面、深入地了解视频直播技术，更好地技术选型。
###1.1 一个完整的直播APP原理
`直播原理` : 把主播录制的视频，推流送到服务器，服务器经过处理(鉴黄等),通过CDN分发给观众看。

`直播环节` : 推流端(采集、美颜、编码、推流)，服务端处理(转码、录制、截图、鉴黄)、播放器(拉流、解码、渲染)、互动系统(聊天室、礼物系统、赞)
###1.2 一个完整直播APP实现流程
`1.采集、2.滤镜处理、3.编码、4.推流、5.CDN分发、6.拉流、7.解码、8.播放、9.聊天互动`
![](直播app流程.png)
###1.3 一个完整直播APP架构
`1.采集端、服务端、播放端`
![](liveApp.png)
###1.4 一个完整直播APP技术点
![](直播app技术点.jpeg)

下面我们会选择一部分技术进行讲解。

`提示`：[总结不好 图看不清 请点这里](http://www.jianshu.com/p/bd42bacbe4cc)

##2. 视频采集
###2.1 基本知识介绍
`AVFundation` : 音视频数据采集需要用AVFundation框架

`AVCaptureDevice` : 硬件设备，包括麦克风、摄像头、通过该对象可以设置物理设备的一些属性。例如相机焦距，白平衡等

`AVCaptureDeviceInput` : 硬件输入对象，可以根据AVCaptureDevice创建对应的AVCaptureDeviceInput对象，用于管理硬件输入数据

`AVCaptureOutput` : 硬件输出对象，用于接收各类输出数据，通常使用对应的子类AVCaptureAudioDataOutput(声音数据输出对象), AVCaptureVideoDataOutput(视频输出对象)

`AVCaptureConnection` : 当把一个输入和输出添加到AVCaptureSession后。AVCaptureSession就会在输出、输出设备之间建立连接，而且通过AVCaptureOutput可以获得这个对象

`AVCaptureVideoPreviewLayer` : 相机拍摄预览图层，能实时查看相机效果。创建该对象需要指定对应的AVCaptureSession对象，因为AVCaptureSession包含输出数据，有视频数据才能显示。

`AVCaptureSession` : 协调输入与输出之间传递数据

###2.2 捕获音视频步骤
包含关系:
![](AVCaptureSession.png)

步骤:
>1.  创建AVCaptureDevice(video或者audio)
>2.  根据AVCaptureDevice创建AVCaptureDeviceInput。
>3.  创建AVCaptureSession
>4.  把创建的AVCaptureDeviceInput加入AVCaptureSession
>5.  添加视频预览图层AVCaptureVideoPreviewLayer
>6.  创建AVCaptureAudioDataOutput，并加入AVCaptureSession
>7.  启动会话

官方步骤(可以忽略)：
> 1. 创建AVCaptureSession对象
> 2. 获取AVCaptureDevice录像设备（摄像头），录音设备（麦克风）。只用于配置
> 3. 根据音频/视频硬件设备(AVCaptureDevice)创建音频/视频硬件输入数据对象(AVCaptureDeviceInput)，专门管理数据输入。
> 4. 创建视频输出数据管理对象（AVCaptureVideoDataOutput），并且设置样品缓存代理(setSampleBufferDelegate)就可以通过它拿到采集到的视频数据
> 5. 创建音频输出数据管理对象（AVCaptureAudioDataOutput），并且设置样品缓存代理(setSampleBufferDelegate)就可以通过它拿到采集到的音频数据
> 6. 将数据输入对象AVCaptureDeviceInput、数据输出对象AVCaptureOutput添加到媒体会话管理对象AVCaptureSession中,就会自动让音频输入与输出和视频输入与输出产生连接.
> 7. 创建视频预览图层AVCaptureVideoPreviewLayer并指定媒体会话，添加图层到显示容器layer中
> 8. 启动AVCaptureSession，只有开启，才会开始输入到输出数据流传输


其中`AVCaptureAudioDataOutput`、`AVCaptureVideoDataOutput`包含两个代理方法，可以一直监听捕获属性。

```
- (void)captureOutput:(AVCaptureOutput *)captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection
{
    if (self.videoConnection == connection)
    {
        NSLog(@"采集到视频");
    }
    else if(self.audioConnection == connection)
    {
        NSLog(@"采集到音频");
    }
}

// 丢失帧会调用这里
- (void)captureOutput:(AVCaptureOutput *)captureOutput didDropSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection NS_AVAILABLE(10_7, 6_0)
{
    NSLog(@"丢失帧");
}
```

不清楚可以看 : [官方文档](https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/04_MediaCapture.html)

###2.3 捕捉图片
`AVCaptureStillImageOutput`可以将捕获到的Video转换为图片。
> 1. 创建device
> 2. 根据device创建deviceInput
> 3. 添加deviceInput进session
> 4. 添加预览图层
> 5. 添加deviceOutput进session
> 6. 调用AVCaptureConnection生成图片
 
###2.4 捕捉视频
`AVCaptureMovieFileOutput` 可以将捕捉到的视频输出到磁盘。可以设置录制最长时限或录制到特定大小，还可以配置成保留最小可用磁盘空间。
> 1. 创建device
> 2. 根据device创建deviceInput
> 3. 添加deviceInput进session
> 4. 添加预览图层
> 5. 添加deviceOutput进session
> 6. 调用AVCaptureMovieFileOutput把视频写入文件

`AVCaptureMovieFileOutput`包含有几个代理方法。分别是`视频开始录制`,`视频暂停`,`视频恢复`, `视频将要录制完成`, `视频录制完成`。

#2.5 采集音频视频按帧输出流程解析
> 1.找到物理设备摄像头_inputCamera、麦克风_microphone，创建摄像头输入videoInput和麦克风输入audioInput；
> 
> 2.设置videoInput和audioInput为_captureSession的输入，同时设置videoOutput和audioOutput为_captureSession的输出，并且设置videoOutput和audioOutput的输出delegate；
> 
> 3._captureSession调用startRunning，开始捕获信号；
> 
> 4.音频数据到达，把数据转发给之前设置的audioEncodingTarget，并通过调用assetWriterAudioInput的appendSampleBuffer方法写入音频数据；
> 
> 5.视频数据到达，视频数据传入响应链，经过处理后通过assetWriterPixelBufferInput的appendSampleBuffer方法写入视频数据;
> 
> 6.选择保存后，文件通过ALAssertLibrary写入手机照片库。

流程图：

![](捕获流程.png)

###2.6 Demo在这里
代码 : [捕获音视频Demo](https://github.com/tiantianlan/LiveExplanation/tree/master/%E6%8D%95%E8%8E%B7%E9%9F%B3%E8%A7%86%E5%B1%8F)

##3. GPUImage
`前面好像没看懂，可以看这里吗？`

可以，GPUImage对AVFundation进行了一层封装，就算你不会前面的也没关系。
###3.1 基本概念
`GPU` 手机或者电脑用于处理图像渲染的硬件

`OpenGL ES` 一套图形与硬件接口，用于把处理好的图片显示到屏幕上。

`GPUImage` 是一个基于OpenGL ES 2.0图像和视频处理的开源iOS框架，提供各种各样的图像处理滤镜，并且支持照相机和摄像机的实时滤镜，内置120多种滤镜效果，并且能够自定义图像滤镜。

`GPUImage` 是一个基于OpenGL ES 2.0图像和视频处理的开源iOS框架，提供各种各样的图像处理滤镜，并且支持照相机和摄像机的实时滤镜，内置120多种滤镜效果，并且能够自定义图像滤镜。

`滤镜处理的原理` 就是把静态图片或者视频的每一帧进行图形变换再显示出来。它的本质就是像素点的坐标和颜色变化

###3.1 利用GPUImage处理直播过程中美颜流程
`采集视频 => 获取每一帧图片 => 滤镜处理 => GPUImageView展示`
![](美颜流程.png)
###3.2 处理画面原理
GPUImage采用链式方式来处理画面,通过addTarget:方法为链条添加每个环节的对象，处理完一个target,就会把上一个环节处理好的图像数据传递下一个target去处理，称为GPUImage处理链。
一般的target可以分为两类：
`中间环节`的target，一般是指各种filter，是GPUImageFilter或者是子类

`最终环节`的target，GPUImageView 用于显示到屏幕上或者GPUImageMovieWriter写成视频文件。

主要分为三个环节：
`source(视频，图片源) => filter(滤镜) => final target(处理后的视频、图片)`

###3.3 美颜原理
`磨皮(GPUImageBilateralFilter)`：本质就是让像素点模糊，可以使用高斯模糊，但是可能导致边缘会不清晰，用双边滤波(Bilateral Filter) ，有针对性的模糊像素点，能保证边缘不被模糊。

`美白(GPUImageBrightnessFilter)`：本质就是提高亮度。

###3.4 GPUImage源对象
GPUImage的数据源只能是4类：

`GPUImageVideoCamera` ios摄像头的实时美颜。GPUImageVideoCamera是GPUImageOutput的子类，提供来自摄像头的图像数据作为源数据，一般是响应链的源头。

`GPUImageStillCamera` 相机拍照

`GPUImagePicture` 处理静止图像

`GPUImageMovie` 电影 

###3.5 用法
>1. 创建过滤器
>2. 创建源对象
>3. 把过滤器添加到源对象
>4. 生成target

静态图片处理：

```
UIImage *inputImage = [UIImage imageNamed:@"105"];
    
// 创建过滤器
GPUImageBrightnessFilter *filter = [[GPUImageBrightnessFilter alloc] init];
filter.brightness = 0.5;
[filter forceProcessingAtSize:inputImage.size];
[filter useNextFrameForImageCapture]; // 告诉系统从后来捕获过滤器
    
// 处理静止的图像
GPUImagePicture *stillPic = [[GPUImagePicture alloc] initWithImage:inputImage];
[stillPic addTarget:filter]; //添加过滤器
[stillPic processImage]; // 执行渲染
    
UIImage *newImage = [filter imageFromCurrentFramebuffer];
    
UIImageView *imageView = [[UIImageView alloc] initWithImage:newImage];
[imageView sizeToFit];
[self.view addSubview:imageView];
 imageView.center = CGPointMake(CGRectGetWidth(self.view.frame)/2, CGRectGetHeight(self.view.frame)/2);
```

实时美颜处理:

```
// 创建视频源
GPUImageVideoCamera *videoCamera = [[GPUImageVideoCamera alloc] initWithSessionPreset:AVCaptureSessionPresetHigh cameraPosition:AVCaptureDevicePositionBack];
// 设置方向
videoCamera.outputImageOrientation = UIInterfaceOrientationPortrait;
    
// 创建预览View
GPUImageView *videoPreview = [[GPUImageView alloc] initWithFrame:self.view.bounds];
[self.view insertSubview:videoPreview atIndex:0];
    
 // 添加预览图层到源
GPUImageBeautifyFilter *fiter = [[GPUImageBeautifyFilter alloc] init];
[_videoCamera addTarget:fiter];
[fiter addTarget:self.videoPreview];
// 开始采集视频
[videoCamera startCameraCapture];

```

到这里，仅仅是屏幕显示的内容有滤镜效果，而作为直播应用，还需要输出带有美颜效果的视频流。
我们需要使用`GPUImageMovieWriter`类，才能处理视频流。

核心思路：

通过GPUImageVideoCamera采集视频和音频信息，音频信息直接发送给`GPUImageMovieWriter`，视频信息传入响应链作为源头，渲染后的视频再写入`GPUImageMovieWriter`，同时通过`GPUImageView`显示在屏幕上。只需要`addTarget`就可以添加`GPUImageMovieWriter`;

![](GPUImageMovieWriter.png)


###3.6 实例代码在这里
[美颜的使用](https://github.com/tiantianlan/LiveExplanation/tree/master/GPUImage)

##4. 音视频编码，解码
这一章太难了，以后再写。
### VideoToolBox
### AudioToolBox

##5. 流媒体服务器
国内外有很多好用的流媒体服务区。这里为了方便搭建我们采用`nginx+RTMP`搭建流媒体服务器。
###5.1 MAC环境搭建
[MACOS上搭建nginx+rtmp环境](https://github.com/tiantianlan/LiveExplanation/blob/master/MACOS%E4%B8%8A%E6%90%AD%E5%BB%BAnginx%2Brtmp%E7%8E%AF%E5%A2%83.md)
###5.2 Centos环境搭建
[Centos下搭建的nginx + RTMP环境](https://github.com/tiantianlan/LiveExplanation/blob/master/Centos%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA.md)
###5.3 服务端常用技术
`CDN` 直播数据上传到服务器后，观看直播的人比较多，服务器是承受不了的，会将数据分发到CDN，观众直接去CDN获取数据。减少服务器的负载。

`负载均衡` 由多台服务器组成一个服务器集群，每次请求的时候，会根据服务器负载选择处理请求的服务器。
##6. 推流
###6.1 推流协议的选择
[HLS和RTMP](https://github.com/tiantianlan/LiveExplanation/blob/master/%E7%9B%B4%E6%92%AD%E5%8D%8F%E8%AE%AE%E7%9A%84%E9%80%89%E6%8B%A9.md)
###6.2 推流原理
在iOS设备上进行各推流的话，是通过AVCaptureSession这么一个捕捉会话，指定两个AVCaptureDevice 也就是iOS的摄像头和麦克风，获取个原始视频和音频，然后需要进行个H.264的视频编码和AAC的音频编码，再将编码后的数据整合成一个音视频包，通过rmtp推送到nginx服务器

###6.3 librtmp
这个参考资料很少。不过大部分都采用的这个。
因为涉及太多C/C++这里不讨论。

参考:[https://my.oschina.net/jerikc/blog/501948](https://my.oschina.net/jerikc/blog/501948)


##7. 音视频播放
###7.1 播放框架的选择
iOS的播放框架主要有以下三种：
>1. AVPlayer 可以播放本地、远程视频，可以自定义UI进行控制
>2. AVPlayerViewController 自带播放控制UI，不能自定义UI
>3. MPMoviePlayerController，MPMoviePlayerViewController (iOS9后弃用)

如果只是简单的播放视频，选择`AVPlayerViewController`，如果想自定义播放器，选择`AVPlayer`。

###7.2 AVPlayer
AVPlayer是一个用来播放基于时间的流媒体控制对象。支持播放从本地、分布下载或通过HTTP Live Streaming协议得到的流媒体。
AVPlayer只管理一个单独资源的播放，不过框架还提供了AVPlayer的一个子类AVQueuePlayer，可以用来管理一个资源队列。当你需要在一个序列中播放多个条目或者为音频、视频资源设置播放循环时可以使用该子类。

AVPlayer视频播放使用步骤：
>1. 创建视频资源地址URL，可以是网络URL
>2. 通过URL创建视频内容对象`AVPlayerItem`,一个视频对应一个`AVPlayerItem`
>3. 创建`AVPlayer`视频播放对象，需要一个`AVPlayerItem`进行初始化
>4. 创建`AVPlayerLayer`播放图层对象，添加到现实视图上去
>5. 添加KVO监听。 监听到AVPlayerItemStatusReadyToPlay的时候调用play方法

[AVPlayer的使用](https://github.com/tiantianlan/LiveExplanation/tree/master/AVPlayer)
###7.3 AVPlayerViewController
`AVPlayerViewController`属于`AV Kit`,它是`UIViewController的子类`，用于展示并控制AVPlayer实例的播放。

`AVPlayerViewController `类使用步骤
>1. 创建URL
>2. 创建AVPlayerViewController，并根据URL设置player属性
>3. 调用play方法

[AVPlayerViewController的使用](https://github.com/tiantianlan/LiveExplanation/tree/master/AVKit) 

Xcode8模拟器可能有问题，打开播放不了。
###8.开源框架
前面所讲都有第三方框架支持。采集、美颜、推流有`LFLiveKit`, 拉流播放有`IJKMediaFramework`。

[`LFLiveKit`](https://github.com/LaiFengiOS/LFLiveKit) : LFLiveKit是iOS版开源RTMP流SDK。他支持后台录制、美颜功能、支持h264、AAC硬编码，动态改变速率，RTMP传输等

[`IJKMediaFramework`](https://github.com/Bilibili/ijkplayer) : ijkplayer是B站开源的一款视频直播框架，它是基于ffmpeg。 如果从github下载是需要编译。参考地址:[http://www.jianshu.com/p/1f06b27b3ac0](http://www.jianshu.com/p/1f06b27b3ac0)

个人实验只需要配置Nginx+RTMP服务
这里我采用这两个第三方框架写了一个直播，包含在线观看直播，和直播推流,支持在线美颜，前后摄像头切换等你需要:

>1. 搭建`Nginx+RTMP环境`:[MAC](https://github.com/tiantianlan/LiveExplanation/blob/master/MACOS%E4%B8%8A%E6%90%AD%E5%BB%BAnginx%2Brtmp%E7%8E%AF%E5%A2%83.md) 或者  [Centos](https://github.com/tiantianlan/LiveExplanation/blob/master/Centos%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA.md)
>2. 下载项目真机运行。[https://github.com/tiantianlan/miaoboDemo](https://github.com/tiantianlan/miaoboDemo)
登陆界面
![](miaobo1.png)

主页
![](miaobo2.png)

直播页面
![](miaobo3.png)






参考文章：
>1. [知乎 如何搭建一个完整的直播系统](https://www.zhihu.com/question/42162310)

>2. [袁峥技术博客](http://www.jianshu.com/users/b09c3959ab3b/latest_articles)

>3. [GPUImage 基于GPU加速的图形和视频处理库](http://www.ios122.com/2015/08/gpuimage/)
>
>4. [iOS中为直播APP集成美颜功能](http://news.h5.com.cn/ios/49696.html)
>
>5. [iOS视频直播~推流、拉流原理](http://www.jianshu.com/p/e516a5b31480)
>
>6. [关于视频直播技术，你想要知道的都在这里了](http://www.jianshu.com/p/ddb640ac4fec)
>
>7. [macOS上搭建Nginx+rtmp服务器](https://www.xsd.me/mac_nginx_rtmp_server.html)
