# -http://blog.csdn.net/iosfengguibin/article/details/50404905 构造虚线的方式

objective-c ASCII NSString转换--分享   
// NSString to ASCII
NSString *string = @"A";
int asciiCode = [string characterAtIndex:0]; //65

//ASCII to NSString
int asciiCode = 65;
NSString *string =[NSString stringWithFormat:@"%c",asciiCode]; //A

Xcode的product=》edit scheme=》diagnostics=》Memory management下有一些选项可以帮助我们找到app中的内存问题：
Enable Scribble

申请内存后在申请的内存上填0xAA，内存释放后在释放的内存上填0x55；
在也就是说如果内存未被初始化就被访问，或者释放后被访问，就会引发异常，这样就可以使问题尽快暴漏出来。
比如下面的代码：
[objc]
-(BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
NSMutableArray* array=nil;
{
NSAutoreleasePool* pool=[[NSAutoreleasePool alloc] init];
array=[[[NSMutableArray alloc] init] autorelease];
[array addObject:@"aa"];
[pool drain];
}
[array addObject:@"bb"];
return YES;
}[/objc]

和我最初想的不同，执行后并没有在[array addObject:@”bb”];这行crash，看到的完全是系统的函数栈：
crash
看到这个栈，你能看出是哪里出的问题吗？
开启Enable Scribble之后试一下：
Enable Scribble
直接就找到了问题所在！但错误地址是0x5555也就是释放的内存上填的0x55，不是真实地地址。
Enable Guard Edges

申请大片内存的时候在前后page上加保护
由于要申请大片内存的时候才会加保护，上面的代码和未勾选这个选项一样。
Enable Guard Malloc

使用libgmalloc捕获常见的内存问题，比如越界、释放之后继续使用。
由于libgmalloc在真机上不存在，因此这个功能只能在模拟器上使用，用模拟器试一下：
Enable Guard Malloc
这一次，不但找到了出错的代码位置，而且错误地址就是array的地址。
Enable Zombie Objects

用僵尸对象代替原有对象，一旦对已经释放的对象发送消息，僵尸对象就会产生异常，并打印日志，用于查找内存问题。
Enable Zombie Objects
由于对象释放后被僵尸对象代替，访问已释放对象的时候，问题会被代码捕获，程序主动产生了一个断点，而且打印了日志：
[text]

crash[4863:487533] *** -[__NSArrayM addObject:]: message sent to deallocated instance 0x14658500

[/text]

值得注意的是：在大型程序里面，如果对象都不释放，将会消耗大量的内存
 

注意：

本文讨论的问题是在在非arc情况下。
现在定位到的crash位置是内存已经被释放后再次被访问的位置，如果array对象是个类的成员对象被释放后还有别的地方使用，找到这个位置并不能说就找到了问题所在，那就可以借助malloc stack来定位更详细的信息。

ipa包 重签名：http://www.cocoachina.com/bbs/3g/read.php?tid=181236

ui优化帧
http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/

手机尺寸
http://iosres.com/

http://c.biancheng.net/cpp/html/138.html


http://m.fx114.net/qa-68-396685.aspx


http://www.jianshu.com/p/2c592daeb3b9

http://endust.github.io/2016/03/12/iOS%E8%A7%86%E9%A2%91%E5%BD%95%E5%88%B6%E5%8E%8B%E7%BC%A9/


http://www.jianshu.com/p/4acdf1d25319

iOS平台基于ffmpeg的视频直播技术揭秘：：https://segmentfault.com/a/1190000005001879

现在非常流行直播,相信很多人都跟我一样十分好奇这个技术是如何实现的,正好最近在做一个ffmpeg的项目,发现这个工具很容易就可以做直播,下面来给大家分享下技术要点:

首先你得编译出ffmpeg运行所需的静态库,这个百度一下有很多内容,这里我就不多说了,建议可以用Github上的一个开源脚本来编译,简单粗暴有效率。

地址:链接描述

下载后直接用终端运行build-ffmpeg.sh脚本就行了,大概半个小时就全部编译好了…反正我觉得速度还行吧(PS:当初编译Android源码那叫一个慢啊…),若是报错就再来一遍,直到提示成功。

视频直播怎么直播呢?大概流程图如下:

1.直播人设备端:从摄像头获取视频流,然后使用rtmp服务提交到服务器

2.服务器端:接收直播人提交的rtmp视频流,并为观看者提供rtmp源

3.观看者:用播放器播放rtmp源的视频.

PS:RTMP是Real Time Messaging Protocol（实时消息传输协议）的首字母缩写。该协议基于TCP，是一个协议族，包括RTMP基本协议及RTMPT/RTMPS/RTMPE等多种变种。

前期准备:

新建一个项目,将所有需要引入的ffmpeg的静态库及其他相关库引入到工程中,配置头文件搜索路径，这一步网上有很多教程就不重复叙述了。

我是用上面脚本编译的最新版,为了后期使用,需要将这些C文件添加到项目:

cmdutils_common_opts.h

cmdutils.h及cmdutils.c

config.h 在scratch目录下取个对应平台的

ffmpeg_filter.c

ffmpeg_opt.c

ffmpeg_videotoolbox.c

ffmpeg.h及ffmpeg.c
除了config.h文件外,别的文件均在ffmpeg-3.0源码目录中

注意问题:

1.编译会报错,因为ffmpeg.c文件中包含main函数,请将该函数重命名为ffmpeg_main并在ffmpeg.h中添加ffmpeg_main函数的声明.

2.ffmpeg任务完成后会结束进程,而iOS设备都是单进程多线程任务,所以需要将cmdutils.c文件中的exit_program方法中的

exit(ret);
改为结束线程,需要引入#include <pthread.h>

pthread_exit(NULL);
直播端：用ffmpeg库抓取直播人设备的摄像头信息,生成裸数据流stream,注意!!!这里是裸流,裸流意味着什么呢?就是不包含PTS(Presentation Time Stamp。PTS主要用于度量解码后的视频帧什么时候被显示出来)、DTS(Decode Time Stamp。DTS主要是标识读入内存中的bit流在什么时候开始送入解码器中进行解码)等信息的数据流,播放器拿到这种流是无法进行播放的.将这个客户端只需要将这个数据流以RTMP协议传到服务器即可。

如何获取摄像头信息:

使用libavdevice库可以打开获取摄像头的输入流,在ffmpeg中获取摄像头的输入流跟打开文件输入流很类似,示例代码:

//打开一个文件:

AVFormatContext *pFormatCtx = avformat_alloc_context();

avformat_open_input(&pFormatCtx, "test.h264",NULL,NULL);
//获取摄像头输入:

AVFormatContext *pFormatCtx = avformat_alloc_context();
//多了查找输入设备的这一步

AVInputFormat *ifmt=av_find_input_format("vfwcap");
//选取vfwcap类型的第一个输入设别作为输入流

avformat_open_input(&pFormatCtx, 0, ifmt,NULL);
如何使用RTMP上传视频流：

使用RTMP上传文件的指令是:

使用ffmpeg.c中的ffmpeg_main方法直接运行该指令即可,示例代码:

NSString *command = @"ffmpeg -re -i temp.h264 -vcodec copy -f flv rtmp://xxx/xxx/livestream";
//根据空格将指令分割为指令数组

 NSArray *argv_array=[command_str componentsSeparatedByString:(@" ")];

//将OC对象转换为对应的C对象
 int argc=(int)argv_array.count;

    char** argv=(char**)malloc(sizeof(char*)*argc);

    for(int i=0;i<argc;i++)

    {

        argv[i]=(char*)malloc(sizeof(char)*1024);

        strcpy(argv[i],[[argv_array objectAtIndex:i] UTF8String]);

    }
//传入指令数及指令数组

ffmpeg_main(argc,argv);

//线程已杀死,下方的代码不会执行

ffmpeg -re -i temp.h264 -vcodec copy -f flv rtmp://xxx/xxx/livestream
这行代码就是

-re参数是按照帧率发送,否则ffmpeg会按最高速率发送,那么视频会忽快忽慢,

-i temp.h264是需要上传的裸h264流

-vcoder copy 这段是复制一份不改变源

-f flv rtmp://xxx/xxx/livestream 是指定格式为flv发送到这个url

这里看到输入是裸流或者是文件,但是我们从摄像头获取到的是直接内存流,这怎么解决呢?

当然是有办法的啦

1.将这串参数中temp.h264参数变为null

2.初始化自定义的AVIOContext，指定自定义的回调函数。示例代码如下：

//AVIOContext中的缓存

unsigned char *aviobuffer=(unsigned char*)av_malloc(32768);

AVIOContext *avio=avio_alloc_context(aviobuffer, 32768,0,NULL,read_buffer,NULL,NULL);

pFormatCtx->pb=avio;

if(avformat_open_input(&pFormatCtx,NULL,NULL,NULL)!=0){

     printf("Couldn't open inputstream.（无法打开输入流）\n");

     return -1;

}
自己写回调函数,从输入源中取数据。示例代码如下:
//Callback
int read_buffer(void opaque, uint8_t buf, int buf_size){
//休眠,否则会一次性全部发送完
   if(pkt.stream_index==videoindex){

           AVRational time_base=ifmt_ctx->streams[videoindex]->time_base;

           AVRational time_base_q={1,AV_TIME_BASE};

           int64_t pts_time = av_rescale_q(pkt.dts, time_base, time_base_q);

           int64_t now_time = av_gettime() - start_time;

           if (pts_time > now_time)

               av_usleep(pts_time - now_time);

       }
     //fp_open替换为摄像头输入流

        if(!feof(fp_open)){

            inttrue_size=fread(buf,1,buf_size,fp_open);

            return true_size;

        }else{

            return -1;

        }

   }
服务端:原谅我一个移动开发不懂服务器端,大概应该是获取直播端上传的视频流再进行广播.所以就略过吧.

播放端:播放端实际上就是一个播放器,可以有很多解决方案,这里提供一种最简单的,因为很多直播软件播放端和客户端都是同一个软件,所以这里直接使用项目中已经有的ffmpeg进行播放简单粗暴又省事.

在Github上有个基于ffmpeg的第三方播放器kxmovie,直接用这个就好.

地址: 链接描述

当你把kxmovie的播放器部分添加到之前做好的上传部分,你会发现报错了......

查找的结果是kxmovie所使用的avpicture_deinterlace方法不存在,我第一个想法就是想办法屏蔽到这个方法,让程序能正常使用,结果......当然不能正常播放视频了,一百度才发现这个方法居然是去交错,虽然我视频只是不够丰富,但是也知道这个方法肯定是不能少的.

没事,只有改源码了.从ffmpeg官方源码库中可以找到这个方法.

地址: 链接描述

发现这个方法在之前的实现中是在avcodec.h中声明是AVPicture的方法,然后在avpicture.c中再调用libavcodec/imgconvert.c这个文件中,也就是说这个方法本身就是属于imgconvert.c的,avpicture.c只是间接调用,查找ffmpeg3.0的imgconvert.c文件,居然没这个方法,但是官方代码库中是有这个方法的,难道是已经移除了?移除不移除关我毛事,我只想能用,所以简单点直接改avpicture.c

首先添加这几个宏定义

#define deinterlace_line_inplace deinterlace_line_inplace_c

#define deinterlace_line         deinterlace_line_c

#define ff_cropTbl ((uint8_t *)NULL)

 然后从网页上复制这几个方法到avpicture.c文件中

static void deinterlace_line_c

static void deinterlace_line_inplace_c

static void deinterlace_bottom_field

static void deinterlace_bottom_field_inplace

int avpicture_deinterlace

再在avcodec.h头文件中, avpicture_alloc方法下面添加声明:
attribute_deprecated

int avpicture_deinterlace(AVPicture *dst, const AVPicture *src,

 enum AVPixelFormat pix_fmt, int width, int height);
保存后再用终端执行build-ffmpeg.sh脚本编译一次就行了…再次导入项目中kxmovie就不会报错了,播放视频的代码如下:

KxMovieViewController *vc = [KxMovieViewController movieViewControllerWithContentPath:path parameters:nil];

[self presentViewController:vc animated:YES completion:nil];
注:其中path可以是以http/rtmp/trsp开始的url
