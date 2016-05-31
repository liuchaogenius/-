# -http://blog.csdn.net/iosfengguibin/article/details/50404905 构造虚线的方式

objective-c ASCII NSString转换--分享   
// NSString to ASCII
NSString *string = @"A";
int asciiCode = [string characterAtIndex:0]; //65

//ASCII to NSString
int asciiCode = 65;
NSString *string =[NSString stringWithFormat:@"%c",asciiCode]; //A
