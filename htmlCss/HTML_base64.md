# HTML_base64数据处理

大家可能注意到了，网页上有些图片的src或css背景图片的url后面跟了一大串字符，比如：
> data:image/png;base64,　iVBORw0KGgoAAAANSUhEUgAAAAEAAAAkCAYAAABIdFAMAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAHhJREFUeNo8zjsOxCAMBFB/　KEAUFFR0Cbng3nQPw68ArZdAlOZppPFIBhH5EAB8b+Tlt9MYQ6i1BuqFaq1CKSVcxZ2Acs6406KUgpt5/　LCKuVgz5BDCSb13ZO99ZOdcZGvt4mJjzMVKqcha68iIePB86GAiOv8CDADlIUQBs7MD3wAAAABJRU5ErkJggg%3D%3D。

那么这是什么呢？  
这是Data URI scheme。

Data URI scheme是在RFC2397中定义的，目的是将一些小的数据，直接嵌入到网页中，从而不用再从外部文件载入。比如上面那串字符，其实是一张小图片，将这些字符复制黏贴到火狐的地址栏中并转到，就能看到它了，一张1X36的白灰png图片。

在上面的Data URI中，data表示取得数据的协定名称，image/png 是数据类型名称，base64 是数据的编码方法，逗号后面就是这个image/png文件base64编码后的数据。

目前，Data URI scheme支持的类型有：
- data:,文本数据
- data:text/plain,文本数据
- data:text/html,HTML代码
- data:text/html;base64,base64编码的HTML代码
- data:text/css,CSS代码
- data:text/css;base64,base64编码的CSS代码
- data:text/javascript,Javascript代码
- data:text/javascript;base64,base64编码的Javascript代码
- data:image/gif;base64,base64编码的gif图片数据
- data:image/png;base64,base64编码的png图片数据
- data:image/jpeg;base64,base64编码的jpeg图片数据
- data:image/x-icon;base64,base64编码的icon图片数据


base64简单地说，它把一些 8-bit 数据翻译成标准 ASCII 字符，网上有很多免费的base64 编码和解码的工具，在PHP中可以用函数base64_encode() 进行编码，如echo base64_encode(file_get_contents(‘wg.png’));

目前，IE8、Firfox、Chrome、Opera浏览器都支持这种小文件嵌入。

举个图片的例子：

网页中一张图片可以这样显示：

> <img src=“http://www.letuknowit.com/images/wg.png”/>

也可以这样显示：
> <img src=“data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAAkCAYAAABIdFAMAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAHhJREFUeNo8zjsOxCAMBFB/KEAUFFR0Cbng3nQPw68ArZdAlOZppPFIBhH5EAB8b+Tlt9MYQ6i1BuqFaq1CKSVcxZ2Acs6406KUgpt5/LCKuVgz5BDCSb13ZO99ZOdcZGvt4mJjzMVKqcha68iIePB86GAiOv8CDADlIUQBs7MD3wAAAABJRU5ErkJggg%3D%3D”/>

我们把图像文件的内容直接写在了HTML 文件中，这样做的好处是，节省了一个HTTP 请求。坏处呢，就是浏览器不会缓存这种图像。大家可以根据实际情况进行自由取舍。
　　
-----------------------
参考文档：
# base64 编码
开发者对Base64编码肯定很熟悉，是否对它有很清晰的认识就不一定了。实际上Base64已经简单到不能再简单了，如果对它的理解还是模棱两可实在不应该。大概介绍一下Base64的相关内容，花几分钟时间就可以彻底理解它。文章下边贴了一个Base64的编解码器，方便阅读文章的同时来实验。

## 一. Base64编码由来

　　为什么会有Base64编码呢？因为有些网络传送渠道并不支持所有的字节，例如传统的邮件只支持可见字符的传送，像ASCII码的控制字符就不能通过邮件传送。这样用途就受到了很大的限制，比如图片二进制流的每个字节不可能全部是可见字符，所以就传送不了。最好的方法就是在不改变传统协议的情况下，做一种扩展方案来支持二进制文件的传送。把不可打印的字符也能用可打印字符来表示，问题就解决了。Base64编码应运而生，Base64就是一种基于64个可打印字符来表示二进制数据的表示方法。

## 二. Base64编码原理

　　看一下Base64的索引表，字符选用了"A-Z、a-z、0-9、+、/" 64个可打印字符。数值代表字符的索引，这个是标准Base64协议规定的，不能更改。64个字符用6个bit位就可以全部表示，一个字节有8个bit位，剩下两个bit就浪费掉了，这样就不得不牺牲一部分空间了。这里需要弄明白的就是一个Base64字符是8个bit，但是有效部分只有右边的6个bit，左边两个永远是0。 

![image](http://images.cnitblog.com/blog/238451/201408/291137095326660.png)  

那么怎么用6个有效bit来表示传统字符的8个bit呢？8和6的最小公倍数是24，也就是说3个传统字节可以由4个Base64字符来表示，保证有效位数是一样的，这样就多了1/3的字节数来弥补Base64只有6个有效bit的不足。你也可以说用两个Base64字符也能表示一个传统字符，但是采用最小公倍数的方案其实是最减少浪费的。结合下边的图比较容易理解。Man是三个字符，一共24个有效bit，只好用4个Base64字符来凑齐24个有效位。红框表示的是对应的Base64，6个有效位转化成相应的索引值再对应Base64字符表，查出"Man"对应的Base64字符是"TWFU"。说到这里有个原则不知道你发现了没有，要转换成Base64的最小单位就是三个字节，对一个字符串来说每次都是三个字节三个字节的转换，对应的是Base64的四个字节。这个搞清楚了其实就差不多了。

![image](http://images.cnitblog.com/blog/238451/201408/291153483606946.png)
![image](http://images.cnitblog.com/blog/238451/201408/291429131888725.png)

但是转换到最后你发现不够三个字节了怎么办呢？愿望终于实现了，我们可以用两个Base64来表示一个字符或用三个Base64表示两个字符，像下图的A对应的第二个Base64的二进制位只有两个，把后边的四个补0就是了。所以A对应的Base64字符就是QQ。上边已经说过了，原则是Base64字符的最小单位是四个字符一组，那这才两个字符，后边补两个"="吧。其实不用"="也不耽误解码，之所以用"="，可能是考虑到多段编码后的Base64字符串拼起来也不会引起混淆。由此可见Base64字符串只可能最后出现一个或两个"="，中间是不可能出现"="的。下图中字符"BC"的编码过程也是一样的。

![image](http://images.cnitblog.com/blog/238451/201408/291217167983928.png)

## 三. 总结　　

　　说起Base64编码可能有些奇怪，因为大多数的编码都是由字符转化成二进制的过程，而从二进制转成字符的过程称为解码。而Base64的概念就恰好反了，由二进制转到字符称为编码，由字符到二进制称为解码。

　　Base64编码主要用在传输、存储、表示二进制等领域，还可以用来加密，但是这种加密比较简单，只是一眼看上去不知道什么内容罢了，当然也可以对Base64的字符序列进行定制来进行加密。

　　Base64编码是从二进制到字符的过程，像一些中文字符用不同的编码转为二进制时，产生的二进制是不一样的，所以最终产生的Base64字符也不一样。例如"上网"对应utf-8格式的Base64编码是"5LiK572R"，对应GB2312格式的Base64编码是"yc/N+A=="。