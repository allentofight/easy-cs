## 前言

一直以来用的都是 MarkEditor 写作，它有一个比较重要的功能：能自动将拷贝到编辑器中的截图同步到图床，这样如果要将文章导出发到其他平台，由于本地的图片在导出后自动转成了链接，所以无需担心图片在其他平台的识别问题。

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0ee51aea68?w=705&h=499&f=jpeg&s=74537)



但是最近发现文章同步到掘金或 CSDN 这些平台时，这些图片链接居然无法转存到他们的平台，应该是 markeditor 的图床用了防盗链技术导致图片无法转存。

**画外音：在掘金，CSDN 这些第三方平台发文时，一般会将文中非本平台的图片链接转存为本平台的，防止第三方图片链接失效导致文章中的图片在平台展示有问题。**

那么该怎么解决呢，有两种方式

1. 一种是找到那些粘贴图片后可以自动上传图床并且生成的图片链接没有防盗链的平台，如 mdnice.com， 不过我试了一下 mdnice.com,貌似有 bug，Chrome 和 Safari 上粘贴图片后自动上传图片不起作用，360浏览器倒是可以。
2. 另一种是在 MarkEditor 里设置其他图床，比如七牛云等，这样可以配置七牛云的图片不采用防盗链技术，但是要配置七牛云这样的图床，一来要收费，二来要去注册帐号，申请域名备案等等，有点麻烦。

考虑之后我决定自己整一个自动上传到图床的工具，无它，自己实现比较 Cool，怎么做呢，一般本地图片要转成最终的图床链接有以下两步

1. 剪切或者复制图片
2. 将图片上传到云端，上传成功后会返回云端的图片链接


我希望这个工具能达到如下流程图所示的效果:

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0ee92f2469?w=444&h=222&f=jpeg&s=25290)



复制图片/截图后，按下快捷键，自动完成之后上传图片到图床----->获取图片地址----->转成 markdown 中的图片地址(即** \!\[\](云端图片url) **这种形式)并将其 copy 到剪切板，这样我在 markdown 编辑器粘贴即可获取云端图片链接。

## 技术选型

使用一个快捷键就能完成后面的所有操作，第一时间我想到了 Alfred 的  workflow，Alfred 堪称是 Mac 的第一神器，它是一个用键盘通过热键、关键字、**自定义插件**来加快操作效率的工具，它不但是搜索工具，还是快速启动工具，甚至能够操作许多系统功能，扩充性极强，其中自定义插件是其核心功能，主要通过 workflow 实现, 什么是 workflow 呢，我们知道一个大的任务可以分解成一个个的小任务（work），这些小任务通过输入输出组合起来就能完成这个大任务，这样一个个 work 组合起来就形成了一个工作流（workflow）

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0eebaba2a9?w=761&h=197&f=jpeg&s=22662)

其中每一个 work 可以由 php, python 等多个编程语言编写，通过 workflow 可以串起各个 work 的输入输出，这样只要触发一下快捷键，workflow 就能自动执行，最终会得到一个结果，比如我之前就写了一个时间戳日期互相转换的 workflow，如下：

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0eece208f0?w=650&h=284&f=gif&s=208734)

在 workflow 中输入 ts（快捷键），后面跟着你要展示的时间戳/日期，即可将其转成日期/时间戳，非常方便。我们在日常中可以将一些重复的工作来用 workflow 实现，这样只要输入一个快捷键即可自动触发，能省下我们很多时间，不亦乐乎！

## 一键上传图片 workflow 实现思路

上节可知 workflow 确实强大，所以用它来实现我们的自动上传图片到图床的功能再合适不过了。

首先我选择了蛋壳（https://imgkr.com/）这个免费又稳定的图床，现在问题的关键是得看下上传图片到蛋壳拿到云端的图片逻辑该怎么写。我们可以打开 charles（或其他抓包工具）然后上传一张图片，成功后可以在 charles 看到上传图片的请求

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0eeba9978c?w=568&h=262&f=jpeg&s=27872)

然后我们看看这个上传图片的请求到底是咋样的，按以下步骤，点击  Copy as cURL，可以看看这个 curl 请求长啥样

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0eed808923?w=1896&h=828&f=jpeg&s=131064)

拷贝出来后的 curl 请求长这样

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0f6d431740?w=692&h=337&f=jpeg&s=87629)

从图中可以看到， curl 请求的请求部分除了图片的二进制数据是动态变化，其他都是固定的，图片的二进制数据无疑是从剪切板中来的，于是问题转化为了如何从剪切板中获取图片数据。

如何从剪切板中获取图片数据呢，这里介绍一个工具: pngpaste, 它可以将图片从剪切板中导出到指定路径，先用 brew 安装一下这个工具

```shell
brew install pngpaste
```

安装之后我们就可以用以下命令将剪切板中的图片导到指定路径了

```
pngpaste 图片路径
```

于是问题转化成如何获取指定路径图片的二进制数据，shell 做不到，不过 php 可以做到，所以我们最终用 php 重写了上文中的 curl 请求，也就是说我们最终选择用 php 来完成最终的 workflow, 最终的 php 实现的思路如下：

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0f6d5e32f5?w=587&h=394&f=jpeg&s=55733)

如有兴趣可以看下如下代码实现，注释写得很详尽了，相信大家应该能看懂

```php
// 将剪切板中的图片拷贝到指定的路径
$command = '/usr/local/bin/pngpaste /tmp/test.jpeg';
$output = shell_exec($command);

require 'workflows.php';
$wf = new Workflows();

// 加载图片二进制数据
$data = file_get_contents('/tmp/test.jpeg');


// 以下为上传图片逻辑
$ch = curl_init('http://imgkr.com/api/v2/files/upload');
$boundary = 'WebKitFormBoundaryCQV3KALJwjBXA5ue';

$files = ['171f3cdebc5586.jpeg' => $data];
$delimiter = '-------------' . $boundary;
$data = '';
foreach ($files as $name => $content) {
    $data .= "--" . $delimiter . "\r\n"
        . 'Content-Disposition: form-data; name="file"; filename="' . $name . '"' . "\r\nContent-Type: image/jpeg" . "\r\n\r\n"
        . $content . "\r\n";
}
$data .= "--" . $delimiter . "--\r\n";
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_HTTPHEADER => [
        'Content-Type: multipart/form-data; boundary=' . $delimiter,
        'Content-Length: ' . strlen($data),
        'Host: imgkr.com',
        'User-Agent: Mozilla/5.0 (Macintosh, Intel Mac OS X 10_15_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.89 Safari/537.36',
        'X-Requested-With: XMLHttpRequest',
        'Accept: */*',
        'Origin: https://imgkr.com',
        'Sec-Fetch-Site: same-origin',
        'Sec-Fetch-Mode: cors',
        'Sec-Fetch-Dest: empty',
        'Referer: https://imgkr.com/',
        'Accept-Language: zh,en-US,q=0.9,en,q=0.8,zh-CN,q=0.7',
        'Cookie: _ga=GA1.2.377288389.1594181932, _gid=GA1.2.851545805.1594809662, _gat=1',
    ],
    CURLOPT_POSTFIELDS => $data
]);

curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_ENCODING, 'gzip, deflate');
$test = json_decode(curl_exec($ch));

// 以下三行为上传图片成功后，将其转化为 markdown 中的图片格式并将其写入剪切板中，这样最终在 markdown 中粘贴后即为对应的 markdown 图片链接
$result = '\'![](' . $test->data . ')\'';
$copy = "echo $result | pbcopy";
shell_exec($copy);

$query = '';
$result = '拷贝到剪切板成功!';
$wf->result($query, $result, $result, $query, 'icon.png');
echo $wf->toxml();
```

**画外音：workflows.php 是用 php 写 workflow 的一个辅助插件，能在 Alfred 的下拉框中展示我们需要的结果，大家如有兴趣可以去 https://github.com/joetannenbaum/alfred-workflow 进一步了解一下**


##  配置 workflow 

脚本既然写好，那配置 workflow 自然不在话下,新建的 workflow 如下:

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0f6d6e9596?w=663&h=759&f=jpeg&s=104142)

以上 workflow 表示当按下「shift+cmd+s」时（即图片中的 Hotkey），会自动执行对应的脚本（Script Filter）将剪切板中的图片上传到图床（执行图片中的脚本 Script Filter），并最终将云端图片转成 markdown 的图片url 并拷贝到剪切板。这样我们只要在编辑器执行一下粘贴命令即可得到我们想要的云端图片 url，效果如下图所示，workflow 成功执行后会在 Alfred 的下拉框中展示「拷贝到剪切板成功」这个信息。

![](https://user-gold-cdn.xitu.io/2020/7/20/17369d0f6e8965ed?w=1960&h=546&f=gif&s=589109)

从此以后，如果我想截图并且获取此图片的链接即可一键搞定！再也不要机械的手动上传图片了！是不是很 Cool!

## 总结

工具化，自动化是工程师非常重要的思维方式，我们应该把重复低效的工作工具化，自动化，把有限的时候投入在更值得做的事情上去，就像现在的自动化测试等也是为了用工具化，自动化的思维帮助研发测试人员从重复低效的工作中解脱出来。workflow 无疑给我们提供了一个很好的手段，日常工作中，我们可以借助 workflow 来提升我们的工作效率！

## 彩蛋

文中的 workflow 我已经整好放在百度网盘了，大家在后台回复「666」即可获取，获取后双击即可使用，需要注意的是要使用 workflow 必须使用付费版的 Alfred，不过 Alfred 的强大功能绝对值得你的投入，我已经买了终身升级版了，使用 workflow 能省下你的很多时间，相信我，买它！绝对物超所值！！！如果链接失效请发我微信「geekoftaste」，我发你。


最后欢迎大家关注公号，加我私人微信「geekoftaste」，一起交流，共同进步！

![](https://user-gold-cdn.xitu.io/2020/7/6/1732461bf651fd4e?w=430&h=430&f=jpeg&s=41396)
