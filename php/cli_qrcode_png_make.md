# 调用草料接口生成二维码
项目中，根据输入的用户id和商品id生成一个独立二维码

首先引入文件
> require_once(BASE_PATH . 'vendor/Http.class.php');  
> require_once(BASE_PATH . 'vendor/simple_html_dom.class.php');

```php
if (!empty($_REQUEST['tfans_id']) && !empty($_REQUEST['goods_id'])) {
    $fansid = trim($_REQUEST['tfans_id']);
    $goodsid = trim($_REQUEST['goods_id']);
}else {
    /* 提示信息 */
    $link[] = array('text' => $_LANG['go_back'], 'href'=>'users.php?act=qrcode');
    sys_msg(sprintf($_LANG['qrcode_fail']), 0, $link);
}
$isvalid = trim($_REQUEST['isvalid']);
$fdesc = trim($_REQUEST['desc']);

// 验证是否已有二维码
if (qrcode_exist($fansid, $goodsid)) {
    /* 提示信息 */
    $link[] = array('text' => $_LANG['go_back'], 'href'=>'users.php?act=qrcode_add');
    sys_msg(sprintf($_LANG['qrcode_already_exist']), 0, $link);
}

$qrdomain = $db_config['QRCODE_DOMAIN'];    // 配置路径 data/database.php
$orgurl = $qrdomain . "/index.php?m=default&c=goods&a=index&id={$goodsid}&uid={$fansid}";
// 组装好的连接由于含有特殊字符，需要转换成url编码格式
$eurl = urlencode($orgurl);

// mhid是模版编号，可以自定义
$url = "https://cli.im/api/qrcode/code?text={$eurl}&mhid=s0qRXgi7zsohMHcsKNVROKs";

// 抓取页面
$response = Http::curlGet($url);

// get html dom from string  
$response = str_get_html($response);

// 获取id=qrcode_plugins_img 的标签
$img = $response->find('#qrcode_plugins_img');

// 获取标签的src属性，也就是图片链接
$imgurl = $img[0]->attr['src'];

// 组装图片链接
$imgurl = "http:{$imgurl}&type=png";

// 抓取图片
$qrimg = Http::curlGet($imgurl);

$filename = qrcodePath($fansid, $goodsid);
$dir = ROOT_PATH . $filename;
file_put_contents($dir, $qrimg);

// insert
$sql = "INSERT INTO byecshop.by_goods_qr_code (fuid, fgoodsid, fisvalid, furl, fqrimg, fdesc) 
        VALUES ({$fansid}, {$goodsid}, {$isvalid}, \"{$url}\", \"{$filename}\", \"{$fdesc}\") ";
$db->query($sql);

```

在Http.class.php中，可以看到curlGet函数，这个CURL技术在后面简单介绍。
```
/**
 * 通过curl get数据
 * @param type $url
 * @param type $timeout
 * @param type $header
 * @return type
 */
static public function curlGet($url, $timeout = 5, $header = "") {
    $header = empty($header) ? self::defaultHeader() : $header;
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, FALSE);    // https请求 不验证证书和hosts
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, FALSE);
    curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, $timeout);
    curl_setopt($ch, CURLOPT_TIMEOUT, $timeout);
    curl_setopt($ch, CURLOPT_HTTPHEADER, array($header)); //模拟的header头
    $result = curl_exec($ch);
    curl_close($ch);
    return $result;
}
```

----------------------
在抓取的过程中，应用到了CURL技术，下面对这个进行简单介绍：
## PHP使用CURL详解

原文地址：++http://www.cnblogs.com/manongxiaobing/p/4698990.html++

CURL是一个非常强大的开源库，支持很多协议，包括HTTP、FTP、TELNET等，我们使用它来发送HTTP请求。它给我 们带来的好处是可以通过灵活的选项设置不同的HTTP协议参数，并且支持HTTPS。CURL可以根据URL前缀是“HTTP” 还是“HTTPS”自动选择是否加密发送内容。

## 使用CURL发送请求的基本流程
使用CURL的PHP扩展完成一个HTTP请求的发送一般有以下几个步骤：

1. 初始化连接句柄；
2. 设置CURL选项；
3. 执行并获取结果；
4. 释放VURL连接句柄。

下面的程序片段是使用CURL发送HTTP的典型过程
```
// 1. 初始化
 $ch = curl_init();
 // 2. 设置选项，包括URL
 curl_setopt($ch,CURLOPT_URL,"http://www.devdo.net");
 curl_setopt($ch,CURLOPT_RETURNTRANSFER,1);
 curl_setopt($ch,CURLOPT_HEADER,0);
 // 3. 执行并获取HTML文档内容
 $output = curl_exec($ch);
 if($output === FALSE ){
 echo "CURL Error:".curl_error($ch);
 }
 // 4. 释放curl句柄
 curl_close($ch);
```

上述代码中使用到了四个函数

- curl_init() 和 curl_close() 分别是初始化CURL连接和关闭CURL连接，都比较简单。
- curl_exec() 执行CURL请求，如果没有错误发生，该函数的返回是对应URL返回的数据，以字符串表示满意；如果发生错误，该函数返回 FALSE。需要注意的是，判断输出是否为FALSE用的是全等号，这是为了区分返回空串和出错的情况。
- CURL函数库里最重要的函数是curl_setopt(),它可以通过设定CURL函数库定义的选项来定制HTTP请求。上述代码片段中使用了三个重要的选项：
    - CURLOPT_URL 指定请求的URL；
    - CURLOPT_RETURNTRANSFER 设置为1表示稍后执行的curl_exec函数的返回是URL的返回字符串，而不是把返回字符串定向到标准输出并返回TRUE；
    - CURLLOPT_HEADER设置为0表示不返回HTTP头部信息。

CURL的选项还有很多，可以到PHP的官方网站 （http://www.php.net/manual/en/function.curl-setopt.php） 上查看CURL支持的所有选项列表。


## 获取CURL请求的输出信息
在curl_exec()函数执行之后，可以使用curl_getinfo()函数获取CURL请求输出的相关信息，示例代码如下：
```
curl_exec($ch);
$info = curl_getinfo($sh);
echo ' 获取 '.$info['url'].'耗时'.$info['total_time'].'秒';
```

上述代码中curl_getinfo返回的是一个关联数组，包含以下数据：
- url:网络地址。
- content_type:内容编码。
- http_code:HTTP状态码。
- header_size:header的大小。
- request_size:请求的大小。
- filetime:文件创建的时间。
- ssl_verify_result:SSL验证结果。
- redirect_count:跳转计数。
- total_time:总耗时。
- namelookup_time:DNS查询耗时。
- connect_time:等待连接耗时。
- pretransfer_time:传输前准备耗时。
- size_uplpad:上传数据的大小。
- size_download:下载数据的大小。
- speed_download:下载速度。
- speed_upload:上传速度。
- download_content_length:下载内容的长度。
- upload_content_length:上传内容的长度。
- starttransfer_time:开始传输的时间表。
- redirect_time:重定向耗时。

curl_getinfo()函数还有一个可选择参数$opt,通过这个参数可以设置一些常量，对应到上术这个字段，如果设置了第二个参数，那么返回的只有指定的信息。例如设置$opt为CURLINFO_TOTAL_TIME，则curl_getinfo()函数只返回total_time,即总传输消耗的时间，在只需要关注某些传输信息时，设置$opt参数很有意义。


## 使用CURL发送GET请求
如何使用CURL来发送GET请求，发送GET请求的关键是拼装格式正确的URL。请求地址和GET数据由一个“?”分割,然后GET变量的名称和值用“=”分隔，各个GET名称和值由“&”连接。PHP为我们提供了一个函数专门用来拼装GET请求和数据部分——http_build_query,该函数接受一个关联数组，返回由该关联数据描述的GET请求字符串。使用这个函数，结合CURL发送HTTP请求的一般流程，我们封闭了一个发送GET请求的函数——doCurlGetRequest,具体代码如下：

```
**
 *@desc 封闭curl的调用接口，get的请求方式。
*/
function doCurlGetRequest($url,$data,$timeout = 5){
 if($curl == "" || $timeout <= 0){
 return false;
 }
 $url = $url.'?'.http_bulid_query($data);
 $con = curl_init((string)$url);
 curl_setopt($con, CURLOPT_HEADER, false);
 curl_setopt($con, CURLOPT_RETURNTRANSFER,true);
 curl_setopt($con, CURLOPT_TIMEOUT, (int)$timeout);
 
 return curl_exec($con);
}
```
这个函数把使用http_build_query 拼装好的带GET参数的URL传给curl_init函数，然后使用CURL发送HTTP请求。


## 使用CURL发送POST请求

可以使用CURL提供的选项CURLOPT_POSTFIELDS，设置该选项为POST字符串数据就可以把请求放在正文中。同样我们实现了一个发送POST请求的函数——doCurlPostRequest，代码如下：

```
/**
** @desc 封装 curl 的调用接口，post的请求方式
**/
function doCurlPostRequest($url,$requestString,$timeout = 5){
 if($url == '' || $requestString == '' || $timeout <=0){
 return false;
 }
 $con = curl_init((string)$url);
 curl_setopt($con, CURLOPT_HEADER, false);
 curl_setopt($con, CURLOPT_POSTFIELDS, $requestString);
 curl_setopt($con, CURLOPT_POST,true);
 curl_setopt($con, CURLOPT_RETURNTRANSFER,true);
 curl_setopt($con, CURLOPT_TIMEOUT,(int)$timeout);
 return curl_exec($con); 
}
```
上面代码中除了设置CURLOPT_POSTFIELDS外，我们还设置了CURL_POST为true,标识这个请求是一个POST请求。在POST请求中也是可以传输GET数据的，只需要在URL中拼装GET请求数据即可秀。

