# 视频编解码GPU加速方案设计

对于视频编解码，主要有两个加速方案：
- 硬件加速
- 非硬件加速

所谓的硬件加速指的是通过显卡，FPGA等硬件设备来实现视频文件的解码和编码功能；非硬件加速指的是通过CPU进行视频编解码．
这两个方案具备的优缺点如下：
硬件加速：
优点：编解码速度快
缺点：画面质量一般
非硬件加速：
优点：画面质量好
缺点：编解码速度较低


本篇文章，我们将从商业化的角度讨论，如果设计一套编解码器插件，来实现GPU全流程转换．

首先，对于PC用户而言，主流的三大GPU厂商的INTEL/NVIDIA/AMD都有自己的硬件加速方案，对于Mac/IOS用户而言，Apple也有提供对应的硬件加速方案，对于Android用户而言，同样有对应的硬件加速方案．

接下来，我们将从真正的商业应用产品开发的角度上，设计每个操作系统平台的硬件加速方案．

## PC端用户：
### INTEL硬件加速方案
首先我们先分析INTEL厂商的硬件加速方案：
现如今，INTEL厂商提供了以下两套硬件加速方案：
- Media SDK(又称为QSV)
- OneVPL

Media SDK(又称为QSV)：提供一套用于视频编解码以及处理（VPP）的API：libmfx，支持Linux/Windows.

在Media SDK的[官网](https://www.intel.com/content/www/us/en/developer/articles/tool/media-sdk.html)，我们可以看到下面这段话：
```C++
The Intel Media SDK project is no longer active.

For continued support and access to new features, Intel Media SDK users are encouraged to read the transition guide on upgrading from Intel® Media SDK to Intel® oneAPI Video Processing Library (oneVPL), and to move to oneVPL as soon as possible.

For more information, see the oneVPL website.
```

从这段Discontinuation Notice，我们可以看出Media SDK(又称为QSV)是旧版本的硬件加速方案，而OneVPL是新的硬件加速方案，但是作为一个商业化学产品的底层架构而言，由于各大厂商的硬件加速方案均采用向前兼容模式，因此我们不得不将两套方案均封装成编解码器插件，以适配旧的I卡设备，同时单独采用一个版本的Media SDK的话，还是无法覆盖到全部I卡设备的用户群体，因此根据每套Media SDK版本的适配群体，我们需要同时提供两个版本的Media SDK编解码插件（2017和2019）,同时再封装一套最新的OneVPL编解码器插件．

### NVIDIA硬件加速方案
作为GPU的AI霸主：NVIDA，Nvidia曾经一度提出VDPAU与Intel 提出的VA-API在Linux上竞争，但最近的趋势似乎是Nvidia走向了更为封闭的方式，最主要的倾向是，Nvidia似乎放缓了对VPDAU的支持，取而代之的是提供较为封闭的NVDEC与NVENC库

因此我们从商业化的角度上来看，选择[Video_Codec_SDK](https://developer.nvidia.com/video-codec-sdk)作为我们的硬件加速方案(PS:Video_Codec_SDK这是一套能运行于Windows 和 Linux 上硬件加速视频编码和解码的方案).

同样为了适配低端旧型号的N卡，我们也需要采用两套版本的Video_Codec_SDK，这边一般采用Video_Codec_SDK 11和Video_Codec_SDK 12

需要注意的是Video_Codec_SDK提供一套跟各版本DirectX/OpenGL/Valken互操作的接口(每个平台支持的互操作对象都不一样)，开发人员可以通过互操作方式将NvDec解码后的数据从CUDA中映射到Direct/OpenGL/Valken，或者将Direct/OpenGL/Valken数据通过互操作映射到CUDA后送给NvEnc编码器．
但是这种互操作的方式除非是必须进行Device转换，否则最好是做成CUDA全流程，并且最好是NV12管线的CDUA全流程(NvDec->Effect->NvEnc)，并且中间Effect链路(旋转/缩放/crop/hdr2sdr等)最好是采用ptx预加载模式，而不是直接用cu脚本，这样才能将性能拉到极致．
### AMD硬件加速方案
AMF SDK用于控制AMD媒体加速器，以进行视频编码和解码以及色彩空间转换，现在开源出来的[版本](https://github.com/GPUOpen-LibrariesAndSDKs/AMF)，并未支持Linux，只能在Windows上进行编码，支持的Codec有AVC/HEVC。需要指出的是AMF的全称是Advanced Media Framework，之前有时会被称之为VCE(Video Coding Engine)

另外，VCE实际上支持两种模式，一种模式是所谓的full fixed mode，这种模式之下，所有的编码相关执行使用的ASIC方式，而另一种模式则是hybrid mode，主要是通过GPU中的3D引擎的计算单元执行编码相关动作，而对应的接口则是AMD's Accelerated Parallel Programming SDK 以及 OpenCL。

但是很可惜的是，AMF SDK的效果真的很一般，在实际用户使用场景下，经常出现部分A卡解码绿屏或者乱码的问题，因此如果要采用这套AMF SDK的话，那就需要覆盖测试，仅开放部分显卡型号，其他型号的A卡全部禁止使用这套AMF SDK．

那么假如用户的电脑不具备I卡核显，并且只有一张A卡，且该A卡型号采用AMF SDK的话会出现解码绿屏/花屏之类问题的话，那么我们难道就只能走CPU编解码了吗？
此时我们就需要使用其他的硬件加速方案．

### 其他硬件加速方案
作为开发PC电脑系统的微软，在windows系统上也有自己的硬件加速方案，分为:
- DXVA
- D3D11VA
- D3D12VA

DXVA一共有DXVA1.0和DXVA2.0两个版本，它们之间的区别是:在DXVA 1中，软件解码器必须通过视频呈现器访问API。如果不调用视频呈现器，就无法使用DXVA 1 API。DXVA 2 中已删除此限制。使用 DXVA 2，主机解码器 (或任何应用程序) 都可以通过 IDirectXVideoDecoderService 接口直接访问API.

一般从产品商业化的角度上，我们并不会采用DXVA作为AMF SDK补充的硬件加速方案，这是因为DXVA2.0经过实际测试过，性能方面不如D3D11VA，而D3D11VA作为现在主流系统的一个AMF SDK补充的硬件加速方案是较为适合的一种选择，而D3D12VA相比D3D11VA而言，主要的新增点在于增加了对AV1编码的支持.

因此，如果从极致性能选择上来说:应该采用D3D11VA＋D3D12VA作为AMF SDK补充的硬件加速方案，并且只在支持D3D12的高版本系统上且需要AV1编码的时候才调用D3D12VA解决方案．

我们在对
Direct3D 11 Video API

## Mac/IOS端用户：
### Apple硬件加速方案
在macOS上的硬件加速接口也是跟随着Apple经历了漫长的演化，从90年代初的QuickTime 1.0所使用的基于C的API开始，一直到iOS 8 以及 OS X 10.8，Apple 才最终发布完整的Video Toolbox framework（之前的硬件加速接口并未公布，而是Apple自己内部使用），期间也出现了现在已经废弃的Video Decode Acceleration (VDA)接口。Video Toolbox是一套C API，依赖了CoreMedia,CoreVideo,以及CoreFoundation框架，同时支持编码，解码，Pixel转换等功能．

因此在Mac端别无选择，直接采用[VideoToolbox](https://developer.apple.com/documentation/videotoolbox)即可

## Linux端用户：

Linux上的硬件加速接口，经历了一个漫长的演化过程，现如今的现状是：
[VDPAU](https://http.download.nvidia.com/XFree86/vdpau/doxygen/html/index.html)与[VAAPI](https://github.com/intel/libva)共存，而这两个API其后的力量，则分别是支持VDPAU的Nvidia和支持VA-API的Intel，另一个熟悉的厂商AMD，实际上同时提供过基于VDPAU和VA-API的支持，真是为难了他。另外，对照VDPAU与VA-API可知，VDPAU仅定义了解码部分的硬件加速，缺少了编码部分的加速（解码部分也缺乏VP8/VP9的支持，且API的更新状态似乎也比较慢），此外，值得一提的是，最新的状态是，Nvidia似乎是想用NVDEC去取代提供VDPAU接口的方式去提供Linux上的硬件加速，或许不久的将来，VA-API会统一Linux上的Video硬件加速接口（这样，AMD也不必有去同时支持VDPAU与VAAPI而双线作战的窘境），这对Linux上的用户，无疑可能是一个福音。除去VDPAU和VAAPI，Linux的Video4Linux2 API的扩展部分定义了M2M接口，通过M2M的接口，可以把CODEC作为Video Filter去实现，现在某些SoC平台下，已经有了支持，这个方案多使用在嵌入式环境之中。

PS:VDPAU与NVDEC的区别:
- NVDEC作为一种私有API，只被nvidia driver支持。
- VDPAU是一种开源API，最初是由Nvidia开发的用于支持Nvidia PureVideo SIP Block，后来发展为可以被多种GPU支持。

### INTEL硬件加速方案
对于Intel Media SDK，除了可以编解码，还有可以进行视频的其他操作。2017年开始，Linux上才有开源的Intel Media SDK实现,之前Linux上的对应方案叫做Intel® Media Server Studio，现在已经不可用了。

Linux上的Intel Media SDK底层基于Libva。编译Intel Media SDK也是要安装VAAPI驱动等Intel媒体栈软件。


类比PC端用户，如果是该硬件加速方案是运行在服务器上的话，那么根据显卡型号选择一个对应版本的Media SDK或者OneVPL作为硬件加速方案即可．当然如果你相信自己实力，也可以直接用VA-API作为硬件加速方案，但是VA-API接口设计很底层，且复杂,没必要干这种吃力不讨好的事情.

### NVIDIA硬件加速方案
类比PC端用户，Linux端从产品商业化角度上来说，直接采用Video_Codec_SDK这套硬件加速方案即可，但是需要注意的是一般Linux端的编解码底层很多时候并不是直接封装成UI提供给用户使用的，而是在企业的服务器上进行运行，提供用户云化的一套转换方案，因此对于Video_Codec_SDK的选择还是需要根据当前服务器企业级显卡的型号来进行版本选择，例如：采用A10显卡的话，那么就需要查看一下当前驱动支持的Video_Codec_SDK版本，如果采用Video_Codec_SDK　12版本的话，NvEncOpenEncodeSessionEx等相关API就会返回无效版本信息，此时就必须采用Video_Codec_SDK　11版本才能正常进行转换

### AMD硬件加速方案
AMF Linux支持尚未正式发布，只能上VA-API

## Android端用户：
仅有Google在Android API 16之后推出的用于音视频编解码的一套偏底层的API:[MediaCodec](https://developer.android.com/reference/android/media/MediaCodec)可以作为Android端硬件加速方案，一般结合OpenGL ES进行来实现全流程GPU转换

# 图片编解码GPU加速方案设计
对于CPU的图片的解码和编码，站在产品商业化的角度上来讲，我们一般采用FreeImage，理由是:FreeImage相较于libpng和stb_image对图片格式支持的类型更多．

而在桌面端系统上仅有两套图片编解码GPU加速方案:
- VideoToolBox(Mac x86/ARM)
- NvJPEG(Windows/Linux)

而在移动端系统上也仅有两套图片编解码GPU加速方案:
- VideoToolBox(IOS)
- NDK ImageDecoder API(Android)

其中VideoToolBox无论是桌面端还移动端，均仅支持HEIC解码

PS:高效率图像格式（High Efficiency Image Format ，HEIF）最早被苹果公司的iPhone所使用，HEIC在各方面均优于JPEG，通过使用更现代的压缩算法，它可以将相同数量的数据大小压缩到JPEG图像文件的50%左右。随着手机Camera的不断升级，照片的细节也日益增加。通过将照片存储为HEIF格式而不非JPEG，可以让文件大小减半，几乎可以在同一部手机上存储以前2倍的照片数量。如果一些云服务也支持HEIF文件，则上传到在线服务的速度也会更快，并且使用更少的存储空间。在iPhone上，这意味着您的照片应该会以以前两倍的速度上传到iCloud照片库。HEIC唯一缺点：兼容性目前使用HEIF或HEIC照片唯一的缺点就是兼容性问题。现在的软件只要能够查看图片，那它肯定就可以读取JPEG图像，但如果你拍摄了以HEIF或HEIC扩展名结尾的图片，并不是在所有地方和软件中都可以正确识别。

因此对于图片编解码GPU加速方案设计最优为：
- 桌面端，如果是HEIC文件且为Apple的设备，走VideoToolBox，如果具备N卡，且为JPEG图片，走NvJPEG，否则走FreeImage.
- 移动端，如果是HEIC文件且为Apple的设备，走VideoToolBox，否则走FreeImage或者用ffmpeg的图片解码也基本够用．

