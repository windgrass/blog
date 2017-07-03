# select标签change事件触发下载
本项目是基于YII2开发的。
在开发中，有个地方需要用select下拉列表来实现下载选中的资源。

这里简单列出实现过程。

## 动态输出下载列表

```
<select id="dld-detail">
    <?php foreach ($details as $det) { ?>
        <option value="<?=$det['fid'] ?>">
            第<?=$det['fsn'] ?>节-<?=$det['fgame'].$det['fgamecoursesn'] ?>
        </option>
    <?php } ?>
</select>
```

## 绑定change事件

```
$('#dld-detail').change(function(){
    var did = $(this).val();
    var url = 'download.html?did='+did;
    console.log(did);
    $('#ifm-detail').attr('src',url);   
//    window.open(url);
});
```
这里有两种思路：
1. 打开新页面下载文件，用```window.open(url)```即可。

2. 用iframe，需要在页面上添加一个隐藏的iframe
```
<iframe id="ifm-detail" src="" style="display: none;"></iframe>
```
这两种方法都是带参访问了后台接口。

## 后台接口

```
public function actionDownload()
    {
        $did = Yii::$app->request->get('did');
        $detail = Model::find()->where(['fid'=>$did])->one();
        // 文件路径
        $file_dir = $detail->fdocpath;
        if (!file_exists($file_dir)) {
            echo '文件不存在';
            return;
        }
        // 下载时显示的文件名
        $file_name = time() . zip';
//        $FileName=iconv("utf-8","gb2312",$file_name);//进行文件名格式转换,以防中文乱码
        // 生成下载
        $file = fopen ( $file_dir, "r" );
        Header ( "Content-type: application/octet-stream" );
        Header ( "Accept-Ranges: bytes" );
        Header ( "Accept-Length: " . filesize ( $file_dir) );
        Header ( "Content-Disposition: attachment; filename=" . $file_name );
        //读取文件内容并直接输出到浏览器
        echo fread ( $file, filesize ( $file_dir) );
        fclose ( $file );
        exit ();
    }
```


------------------------
参考信息：

# js带参访问后台下载接口
1. JS实现新窗口打开链接
```
1.超链接<a href="http://www.jb51.net" title="脚本之家">Welcome</a>
等效于js代码
window.location.href="http://www.jb51.net";     //在同当前窗口中打开窗口
 
2.超链接<a href="http://www.jb51.net" title="脚本之家" target="_blank">Welcome</a>
等效于js代码
window.open("http://www.jb51.net");                 //在另外新建窗口中打开窗口
```

2. iframe标签的src属性跳转
在用页面跳转的方式下载，会留下一个空白页面。  
iframe等于是打开一个独立的窗口，可以用隐藏的iframe实现下载。
```
HTML：
<iframe id="ifm-detail" src="" style="display: none;"></iframe>

JS：
$('#ifm-detail').attr('src',url);
```


# PHP实现文件下载

简单的文件下载只需要使用HTML的连接标记```<a>```，并将属性href的URL值指定为下载的文件即可。所示：

```
<a href=”http://www.*****.net/download/book.rar”>下载文件</a>
```

如果通过上面的代码实现文件下载，只能处理一些浏览器不能默认识别的MIME类型文件，例如当访问book.rar文件时，浏览器并没有直接打开，而是弹出一个下载提示框，提示用户“下载”还是“打开”等处理方式。但如果需要下载后缀名为.html的网页文件、图片文件及PHP程序脚本文件等，使用这种连接形式，则会将文件内容直接输出到浏览器中，并不会提示用户下载。

为了提高文件的安全性，不希望在```<a>```标签中给出文件的链接，则必须向浏览器发送必要的头信息，以通知浏览器将要进行下载文件的处理。PHP使用```header()```函数发送网页的头部信息给浏览器，该函数接收一个头信息的字符串作为参数。文件下载需要发送的头信息包括以下三部分，通过调用三次```header()```函数完成。以下载图片test.gif为例，需要发送的头信息的所示：

```
header(‘Content-Type:imge/gif'); //发送指定文件MIME类型的头信息
header(‘Content-Disposition:attachment; filename=”test.gif”‘); //发送描述文件的头信息，附件和文件名
header(‘Content-Length:3390′); //发送指定文件大小的信息，单位字节
```

如果使用```header()```函数向浏览器发送了这三行头信息，图片test.gif就不会直接在浏览器中显示，而让浏览器将该文件形成下载的形式。

在函数```header()```中
- “Content-Type”指定了文件的MIME类型
- “Content_Disposition”用于文件的描述，值“attachment
- filename=”test.gif”说明这是一个附件，并且指定了下载后的文件名
- “Content_Length”则给出了被下载文件的大小

设置完头部信息以后，需要将文件的内容输出到浏览器，以便进行下载。可以使用PHP中的文件系统函数将文件内容读取出来后，直接输出给浏览器。最方便的是使用```readfile()```函数，将文件内容读取出来直接输出。
下载文件test.gif的所示：

```
<?php
$filename = "test.gif";
header('Content-Type:image/gif'); //指定下载文件类型
header('Content-Disposition: attachment; filename="'.$filename.'"'); //指定下载文件的描述
header('Content-Length:'.filesize($filename)); //指定下载文件的大小
 
//将文件内容读取出来并直接输出，以便下载
readfile($filename);
?>
```

上面如果碰到中文名字就会无法正常下载了，对于中文名字下载文件我又找到一个文件下载实例代码

```
<?php  
header("Content-type:text/html;charset=utf-8");  
// $file_name="cookie.jpg";  
$file_name="圣诞狂欢.jpg";  

//用以解决中文不能显示出来的问题  
$file_name=iconv("utf-8","gb2312",$file_name);  

$file_sub_path=$_SERVER['DOCUMENT_ROOT']."marcofly/phpstudy/down/down/";  
$file_path=$file_sub_path.$file_name;  

//首先要判断给定的文件存在与否  
if(!file_exists($file_path)){  
echo "没有该文件文件";  
return ;  
}  

$fp=fopen($file_path,"r");  
$file_size=filesize($file_path);  

//下载文件需要用到的头  
Header("Content-type: application/octet-stream");  
Header("Accept-Ranges: bytes");  
Header("Accept-Length:".$file_size);  
Header("Content-Disposition: attachment; filename=".$file_name);  
$buffer=1024;  
$file_count=0;  

//向浏览器返回数据  
while(!feof($fp) && $file_count<$file_size){  
$file_con=fread($fp,$buffer);  
$file_count+=$buffer;  
echo $file_con;  
}  
fclose($fp);  
?>
```

header("Content-type:text/html;charset=utf-8")的作用：在服务器响应浏览器的请求时，告诉浏览器以编码格式为UTF-8的编码显示该内容

关于file_exists()函数不支持中文路径的问题:
因为php函数比较早，不支持中文，所以如果被下载的文件名是中文的话，需要对其进行字符编码转换，否则file_exists()函数不能识别，可以使用iconv()函数进行编码转换

$file_sub_path() 我使用的是绝对路径，执行效率要比相对路径高
   
Header("Content-type: application/octet-stream")的作用：通过这句代码客户端浏览器就能知道服务端返回的文件形式  

Header("Accept-Ranges: bytes")的作用：告诉客户端浏览器返回的文件大小是按照字节进行计算的 

Header("Accept-Length:".$file_size)的作用：告诉浏览器返回的文件大小  

Header("Content-Disposition: attachment; filename=".$file_name)的作用:告诉浏览器返回的文件的名称

以上四个Header()是必需的 
fclose($fp)可以把缓冲区内最后剩余的数据输出到磁盘文件中，并释放文件指针和有关的缓冲区
