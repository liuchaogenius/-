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
