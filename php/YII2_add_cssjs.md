# YII2中引入JS/CSS文件
在开发中，需要在YII2的框架中添加第三方的资源文件。研究了已有的项目，发现添加的流程大致如下。

以下流程为是分析贝板日志项目中引入mustache文件为例来说明的。

### 1. 找到asset文件夹
找到当前模块中的asset文件夹，里面默认会有个AppAsset.php文件，在同目录下新建一个NewAsset.php文件，示例代码如下：
```php
// 路径：backend/assets/CsMonitorAsset.php

namespace backend\assets;

use yii\web\AssetBundle;

class CsMonitorAsset extends AssetBundle
{
    public $depends = [
        'backend\assets\AppAsset',
        'yii\bootstrap\BootstrapPluginAsset',
        'vendor\frontlib\mustache\MustacheAsset',
        'vendor\frontlib\php\PhpAsset' // 这里的路径会寻到‘PhpAsset.php’文件
    ];

    public $jsOptions = [
        'position' => \yii\web\View::POS_HEAD,   // 这是设置所有js放置的位置
    ];
}
```

### 2. 在vendor里面添加资源文件
可以看到第一步的depends选项中的路径，现在去vendor里面自定义添加文件吧，路径需要和depends里面的符合。
看看PhpAsset.php文件中的代码，前面都是在指定路径，在这里才开始加载文件的
```php
//  路径：vendor/frontlib/php/PhpAsset.php

namespace vendor\frontlib\php;

use yii\web\AssetBundle;

class PhpAsset extends AssetBundle
{
    public $sourcePath = '@vendor/frontlib/php';

    public $js = [      // 这里的不能少，否则不会加载php.js文件
        'php.js',
    ];
    public $css = [
        'common.css',
    ];
}

```
再看一个加载文件的例子：
```php
class MustacheAsset extends AssetBundle
{
    public $sourcePath = '@vendor/frontlib/mustache';
    public $js = [
        'jquery.mustache.js',
        'mustache.min.js',
    ];
}
```

这里面的 public $js ; public $css 不能少，到了这里才开始加载文件，路径是相对当前目录的。

### 3. 前端页面显示
在前端页面是调用 register方法来加载这些资源的，代码如下：
```php
use backend\assets\CsMonitorAsset;
use yii\helpers\Html;
CsMonitorAsset::register($this);
```

加载出来后，在前端页面上查看这些资源，会发现的结果是：
```html
<script src="/assets/e2d7db14/jquery.mustache.js"></script>
<script src="/assets/e2d7db14/mustache.min.js"></script>
<script src="/assets/9c4d3c5e/php.js"></script> 
```
这里会多出个命名奇特的文件夹‘9c4d3c5e’，这个是YII2处理过的，针对每个页面加载不同的资源，以提高页面反应速度，原则上只加载需要的资源文件。

-----------------------
# 参考资料
from: http://www.yiichina.com/tutorial/411

### 1、在前台view中最简单不过的就是像之前那样一个文件一个文件的引入,于是在顶部使用use调用代码段

> use yii\helpers\Html;

然后在下面的Html中可以这样调用

> <?=Html::jsFile('@web/***/js/***.js')?>//这里***代表你的目录名或者文件名
> <?=Html::cssFile('@web/***/css/***.css')?>//***同上

这样的话就不需要动其他文件，直接引入文件就好了，需要哪个引入哪个，当然这样写的话就是每次得写很多行代码去加载，最好还是写到配置文件中，但是用配置文件来引入这个问题我暂时还没弄通，后面如果找到原因我会分享给大家

### 2、前台这样引入，那么在controller中怎么自定义样式文件呢
在控制器中加上以下代码

public $layout = 'layout';//在类中定义一个变量，名为$layout

注意的是这个layout在你的view中有个目录叫layouts，在这个目录下，我新建了一个文件名为layout.php,在其中我加上一句代码

> <?php echo $content; ?>

这样控制器就会自动去找当前视图目录下的layouts目录下的加载视图文件的php文件

以上的几行简短的代码就解决了新手不知道该如何去加载CSS,JS文件的问题，大家如果觉得写***Asset.php文件会有问题，就用我这种办法，后期等熟悉了yii2之后在改用其他的办法去加载

另外，我再补充下，在view中怎么去跳转链接到其他的视图文件

同样在顶部先引入类库

> use yii\helpers\Url;

然后再需要链接跳转的地方这样写：

> <?phpecho Url::toRoute('post/index');?>//post为你的当前控制器名，index为view模版

是不是特别简单啊！