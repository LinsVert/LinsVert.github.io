---
title: 我用PHP刷某站的阅读量
date: 2018-05-17 00:00:19
tags: 投机倒把
categories: 投机倒把
---

之前逛掘金帖子的时候，发现一个群的群主拿Python刷了他在某个博客的阅读量，今天我也用世界上最好的语言 : ) PHP来 刷一刷某站点的阅读量。

框架用的是 [PHPSpider](https://github.com/owner888/phpspider)一个比较轻量的爬虫框架。本爬虫主要用了其中的 ```request库```和 ```selector库```。想要了解这个框架的可以链过去看看。

<!-- more -->

###### 进入正题：

1.瞎几把分析一波：进入文章详情页，刷新几遍，发现阅读量上去了，说明刷阅读量方法可行。
![第一次](https://upload-images.jianshu.io/upload_images/7393873-44261b7a2aaf10db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
![第三次](https://upload-images.jianshu.io/upload_images/7393873-24d4fad4ea0777cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

2.先直接访问文章详情页
```
include __DIR__."/../autoloader.php";
use spider\core\requests;
$url = "https://www.jianshu.com/p/26c2eca28dc5";
$html = requests::get($url);//简单的get
```
php cli 之，然后刷新页面，发现没增加次数，说明肯定存在某种方式的验证。万金油先加```referer header```

```
include __DIR__."/../autoloader.php";
use spider\core\requests;
$url = "https://www.jianshu.com/p/26c2eca28dc5";
$referer = "https://www.jianshu.com";
requests::set_referer($referer);
$html = requests::get($url);//简单的get
```
cli然后刷新，还是没用，那就再复制这个页面的 ```cookie```
```
include __DIR__."/../autoloader.php";
use spider\core\requests;
$url = "https://www.jianshu.com/p/26c2eca28dc5";
$referer = "https://www.jianshu.com";
$cookie = "自己复制，太长就不贴了"
requests::set_referer($referer);
requests::set_cookie($referer,$cookie);
$html = requests::get($url);//简单的get
```
发现还是没用，难道是```userAgent```?加了发现还是不行，那么就可能存在JS动态获取阅读量的可能。F12一波，看下```XHR```,这个```mark_viewd.json```明显就是加阅读量的方法了,看了下返回204，not data，说明阅读量是下次刷新生效。
![增加阅读量的请求](https://upload-images.jianshu.io/upload_images/7393873-b8f629fb87ee4000.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
请求内容：
![请求体](https://upload-images.jianshu.io/upload_images/7393873-da85b3d69d924f36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

找波规律：
1.请求的url中有该文章的关键字
2.post 有个奇怪的uuid和 referer
说明请求的url ```notes/```后跟的是这个文章的资源ID；至于uuid发现其他AJAX请求没有给到这个值，极有可能是事先渲染好的，果断在Elements 中Ctrl+F一波
![文章页中的uuid](https://upload-images.jianshu.io/upload_images/7393873-82ecbc67e0b7e471.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
发现文章详情页的Script 有个JSON包含这个uuid,那么逻辑就可以理顺了，先验证是否是这个请求增加阅读量的。
```
$url = "https://www.jianshu.com/notes/5764256f0c43/mark_viewed.json";
$referer = "https://www.jianshu.com/p/5764256f0c43";
requests::set_referer($referer);//必须将referer调成当前页面
$data = [
"uuid"=>"912e01f5-6da4-4bfd-8e71-4d1b62499709",
"referer"=>"https://www.jianshu.com/"
];
requests::post($url,json_encode($data));
```
执行之后发现次数增加了，需要注意的是 ```request::referer() 请求体referer```要设置成该页面的值，不然不会加阅读量，设置成```www.jianshu.com```我没测试。

那么接下来就是撸波代码,用上多进程同时刷我自己的其他文章：
###### Code
```
<?php
/**
 * Created by PhpStorm.
 * User: lins
 * Date: 14/05/2018
 * Time: 11:26 PM
 */

//简书 刷阅读量
include __DIR__."/../autoloader.php";
use spider\core\requests;
use spider\core\selector;

$url = "https://www.jianshu.com/u/6b1fa1764b51";
$referer = "https://www.jianshu.com";
$urll = "https://www.jianshu.com/p/26c2eca28dc5";
$userAgent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.139 Safari/537.36";
$cookie = "_ga=GA1.2.1446998420.1512720610; read_mode=day; default_font=font2; locale=zh-CN; web-note-ad-fixed=1524537468387; Hm_lvt_0c0e9d9b1e7d617b3e6842e85b9fb068=1526200190,1526220098,1526282339,1526312631; sensorsdata2015jssdkcross=%7B%22distinct_id%22%3A%22160352cb5d7300-071265b68a5aa1-17396d57-1296000-160352cb5d8d37%22%2C%22%24device_id%22%3A%22160352cb5d7300-071265b68a5aa1-17396d57-1296000-160352cb5d8d37%22%2C%22props%22%3A%7B%22%24latest_traffic_source_type%22%3A%22%E8%87%AA%E7%84%B6%E6%90%9C%E7%B4%A2%E6%B5%81%E9%87%8F%22%2C%22%24latest_referrer%22%3A%22https%3A%2F%2Fwww.google.co.jp%2F%22%2C%22%24latest_referrer_host%22%3A%22www.google.co.jp%22%2C%22%24latest_search_keyword%22%3A%22%E6%9C%AA%E5%8F%96%E5%88%B0%E5%80%BC%22%2C%22%24latest_utm_source%22%3A%22desktop%22%2C%22%24latest_utm_medium%22%3A%22not-signed-in-like-button%22%2C%22%24latest_utm_campaign%22%3A%22maleskine%22%2C%22%24latest_utm_content%22%3A%22note%22%7D%2C%22first_id%22%3A%22%22%7D; _m7e_session=9cfeba4267a7645d2058d2a971670be3; signin_redirect=https%3A%2F%2Fwww.jianshu.com%2Fp%2F26c2eca28dc5; Hm_lpvt_0c0e9d9b1e7d617b3e6842e85b9fb068=1526391092";
requests::set_referer($referer);

requests::set_cookie($referer,$cookie);

requests::set_useragent($userAgent);


$html = requests::get($url);

$result = selector::select($html,'//div[@class="content"]/a/@href');//选择器拿到用户下的所有文章

$result = array_map(function ($e)use($referer){
    $word = explode('/',$e);//word关键词用来拼接json
    $word = $word[2];
    $ref = $referer.$e;//文章地址
    return ['word'=>$word,'referer'=>$ref];
},(array)$result);

//玩下fork

$num = 4;
$runTimes = 7200;
if(count($result) < $num){
    $num = count($result);
}
var_dump($num);
for ($i = 0;$i<$num;$i++){

    $pid = pcntl_fork();
    if(0 < $pid){
        //father
        //准备回收
        pcntl_signal(SIGCHLD,function() use( $pid ) {
            echo "收到子进程退出".PHP_EOL;
            pcntl_waitpid( $pid, $status, WNOHANG );
        });
//        pcntl_alarm(1);
        pcntl_signal_dispatch();

    }elseif(0 == $pid){
        $time = time() + $runTimes;
        while(time() < $time){
            $url = toSee($i,$result,$num);
            file_put_contents('debug.json',json_encode($url).PHP_EOL,FILE_APPEND);

            sleep(rand(1,5));
        }
        echo $pid."*$i run finished ".date('Y-m-d H:i:s').PHP_EOL;
        exit;
    }else {
        echo 'fork error';
    }
}
function toSee(int $i,array $result,int $num){
    $referer = "https://www.jianshu.com/u/6b1fa1764b51";
    $url = [];
    $reNums = count($result);
    //将抓到的文章资源平均分配到子进程中
    if ($reNums == $num){
        $url[] = $result[$i];
    }else {
        do{
            $url[] = $result[$i];
            $i = $i+$num;
        }while(isset($result[$i]));

    }
    foreach ($url as $key) {
        $ulas = "https://www.jianshu.com/notes/%s/mark_viewed.json";
        $res = requests::get($key['referer']);//拿到文章详情页

        $path = "/html/body/script[1]/text()";

        $ress = selector::select($res,$path);//拿到script 下的json
        $ress = json_decode($ress);

        $data = [
            "referer" => $referer,//这个值好像随意是jianshu的域名就可以
            'uuid' => $ress->note_show->uuid
        ];
        $ulas = sprintf($ulas,$key['word']);
        echo $ulas.PHP_EOL;
        requests::set_referer($key['referer']);//必须将referer调成当前页面
        requests::post($ulas,json_encode($data));
    
    }
    return $url;
}
```
本文用的抓取语法是[XPATH](http://www.w3school.com.cn/xpath/xpath_syntax.asp)有兴趣的可以了解下。
保存，然后丢到服务器，开个定时脚本跑，大功告成。

PS:这段代码请放在项目的demo文件夹里，然后```php 文件名.php```就可以了。

值得注意的是，有些*unix服务器```curl```是 nss 方式 请求ssl的 pctnl_frok的时候会报错，需要将curl 请求ssl的nss 变为 openssl,方法如下：
```
先yum update openssl

再yum update curl

curl -V 
```
如果不是下面的 openssl 
![curl -V](https://upload-images.jianshu.io/upload_images/7393873-56b30ad8a587c043.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
那就看下这个链接吧，
https://www.cnblogs.com/showker/p/4706271.html

以上。
撒花。:)
小刷怡情，大刷伤身。

2018-05-20 22:27 更新 ，因为会被jianshu大佬ban Ip 所以我另外写了爬虫抓取免费的ip代理，相关更新代码和更新版本请异步小弟的  [GayHub](https://github.com/LinsVert/Spider)


