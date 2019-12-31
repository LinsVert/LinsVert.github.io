---
title: GO初体验之爬取情侣头像
date: 2019-03-13 13:18:03
tags: Learning
categories: Learning
---


######  最近有想转golang的想法，所以抽了点时间学习了一波golang的基本语法，感觉上这个静态解析语言用起来的确没有php那么舒服，但感觉上用起来还是很爽的。

######  刚好女票要换个情侣头像,那就用Go爬一波情侣头像吧～

<!-- more -->

##### Let's Code

![image](http://upload-images.jianshu.io/upload_images/7393873-cae7d6070c367825.gif?imageMogr2/auto-orient/strip)

##### 本次使用的工具

```

golang go1.11.4

编辑器 vscode1.31.1

浏览器 Chrome72.0.3626.119

```

### 1.找到目标网站 分析一波

![目标网站模样](http://upload-images.jianshu.io/upload_images/7393873-19c6f2a309f25d91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "[目标网站]")

F12看一波结构:

发现每个 `li` 对应跳转一个目标网站，估计是图片的集合

![页面结构](http://upload-images.jianshu.io/upload_images/7393873-4c10a31f6c754f3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "[页面结构]")

翻到底部，看到有分页标记（感觉是模版生成的。。。）,看一波链接，这个链接遵循只是最后资源地址不一样，其他是一样的，所以可以使用正则匹配一波，可以利用分页爬取比较大量的起始链接。

![分页链接](http://upload-images.jianshu.io/upload_images/7393873-e7118e8ffe007be5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "[分页链接]")

1.分页正则:

` http:\/\/www.wxcha.com\/touxiang\/qinglv\/hot_\d+.html `

2.详情页正则:

` http://www.wxcha.com/touxiang/\d+.html `

接下来去详情页看看：

![详情页](http://upload-images.jianshu.io/upload_images/7393873-3eb08f62c1109644.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "[详情页]")

F12定位下图片元素,不难得到图片资源地址的正则:

3.图片地址正则:

` http://img.wxcha.com/file/\d+/\d+/\w+.jpg `

好了，目前看来这个网站的图片结构还是很明朗的，并没有用JS加载，应该是模版同步渲染的，接下来就是研究下怎么设计这个爬虫。

### 2.设计爬虫

页面有3层，所以可以有以下思路:

1.先用第一页，按一定深度爬取分页，拿到一定数量基础页面链接

2.利用这些基础链接，获取详情页的链接

3.循环获取图片地址，并下载

思路理顺了，接下来就是如何设计这个爬虫了。

***

为了熟悉Golang，所以没有用其他的爬虫框架，考虑到复用性需要写一个爬虫的方法用于一些基本的数据爬取,所以先定义一个结构体（可以简单理解为PHP的类）:

```

type Spider struct {

Types string

Url string

Regex string

Deep int64

Client *http.Client

Req *http.Request

ResponseBody []byte

Headers map[string]string

}

```

Types是Request Method，像`Get`或`Post`

Url 是目标链接

Regex 是正则，用于Url页面内容的解析（不知道可以用Xpath不，没研究过，Todo下）

Deep 是爬虫深度，用于获取分页内容或者其他有深度的请求页面,定义深度防止无限抓取

Client 是`net/http`包的自定义client方法（用它主要是能自定义header，一些网站有反爬机制，需要定义header）

Req 是http.NewRequest主要用于一些设置一些自定义请求

ResponseBody 顾名思义是请求返回的body体，因为返回内容是`[]byte`所以是定义为`[]byte`

Headers 自定义的header头，提供外部定义方法，方便爬取数据

目前想到的能通用的只有这些了,然后在需要函数具体实现这些：

1.需要一个SetHeader方法设置Headers

```

func (sp *Spider) setHeader(header map[string]string) {}

```

入参用map key-value形式比较适合～

2.需要一个SetParam方法设置一些基本sp的一些参数

```

func (sp *Spider) SetParam(param map[string]string) {}

```

比如修改Url,Deep,Regex,Types这类需要重复赋值的参数

3.需要一个getContent方法用于获取请求的内容（核心方法）

```

func (sp *Spider) getContent() {}

```

4.需要一个Find方法用于正则匹配getContent获取的内容（可以考虑多种方式匹配内容，Todo）

```

func (sp *Spider) Find() []string {}

```

5.需要一个Download方法用于保存返回内容（图片啊，网页啊之类的）

```

func (sp *Spider) Download(fileNamePath string, fileTypes string, downloadUrl string, fileName string) string {}

```

考虑到要保存的可能不止只有图片，所以就定义下入参，方便灵活下载内容

所以基本的爬虫方法已经定义好，就差内部具体实现了，具体实现就看代码吧～这里就不贴出来了,这里只记录下要点。

- 获取到数据后，需要调用 `resp.Body.Close()` 关闭body

- 使用包 `io/ioutil` 来获取返回内容 `ioutil.ReadAll(resp.Body)`

- 因为 `SetParam` 方法定义了入参为 `map[string]string` 所以 Deep参数不能使用int，需要在 `SetParam` 使用 `strconv.ParseInt()` 来string to int

- 正则使用的是 `regexp` 包，定义好 `MustCompile` 后 执行 `regexp` 的 `FindAllString` 方法

- 下载内容没有定义fileName时，fileName使用的是Unix时间戳,但如果用 go func 可能会遇到文件名称一样的问题，可以加上随机字符串

- 写入文件调用 `io/ioutil` 的 `ioutil.WriteFile`

大多数的包可以在 [go语言标准库](https://studygolang.com/pkgdoc) 中找到，而且每个方法都有详细说明，如果调包参数未知可以上这个网站上查看，也可以使用 [go doc命令](http://wiki.jikexueyuan.com/project/go-command-tutorial/0.5.html) 来查看相关命令的文档。

### 3.设计目标网站的爬虫

前面的已经设计了3层，爬虫类也实现了，那么接下来就实现这个qinglv爬虫。

基本的，为了防止反爬拦截，事先定义下headers:

```

var Url string = "http://www.wxcha.com/touxiang/qinglv/hot_1.html"

headers := map[string]string{

"User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X) AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1",

"Referer": Url,

}

```

定义下UA和referer。

#### 3.1 按深度获取分页下的页面链接

根据第一页，获取所有符合正则条件的页面链接,核心代码如下：

```

resp := sp.Find()

result := resp

if len(resp) > 0 {

//有数据才继续

i := sp.Deep

var flag int64 = 0

maxFor := len(resp)

forLen := 0

for flag < i {

flag++

for len(resp) > 0 && forLen <= maxFor {

target := resp[0]

if readySearch[target] == 0 {

readySearch[target] = 1

sp.Url = target

resp = resp[1 : len(resp)]

_resp := sp.Find()

resp = append(resp, _resp...)

result = append(result, _resp...)

} else {

resp = resp[1 : len(resp)]

}

forLen++

}

forLen = 0

}

}

```

获取到第一条页面的所有内容后，再for循环获取resp下所有页面链接下的分页链接，并去重，深度需要自己实现，这里比较简单暴力，根据第一次获取的结果的长度作为深度标志，循环获取的结果长度超过这个长度就进行下一深度循环，虽然会获取重复数据，但本人也没有更好的解决方法，就简单的实现下了。

去重方法：

```

    //去重 想下是否可以想个办法简化这个去重

var finalResultMap map[string]int

finalResultMap = make(map[string]int)

for _, v := range result {

finalResultMap[v] = 1

}

var finalResultSlice []string

for k, _ := range finalResultMap {

finalResultSlice = append(finalResultSlice, k)

}

```

去重方法度娘谷歌没有好的解决方案，就临时实现下了。

#### 3.2 根据前面获取的链接slice获取详情页的链接

获取3.1的链接后，获取详情页的链接，方法与3.1雷同，只是变更了正则。

#### 3.3 最后一步根据详情页获取图片地址并下载

在3.2的基础上，获取到图片链接后，调用 `sp.Download()` 下载图片

```

func DownloadImg(){

fmt.Printf("开始下载图片...\n")

var downLoadFile string = "/Users/lins/Desktop/spider/"

var param map[string]string

param = make(map[string]string)

param["Regex"] = `http://img.wxcha.com/file/\d+/\d+/\w+.jpg`

param["Types"] = "get"

sp := &lsp.Spider{}

headers := map[string]string{

"User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X) AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1",

"Referer": Url,

}

lsp.SetHeaders(headers)

sp.SetParam(param)

var result []string

detailUrls := GetDetailUrls()

for _,v := range detailUrls {

sp.Url = v

resp := sp.Find()

result = append(result, resp...)

}

//去重 想下是否可以想个办法简化这个去重

var finalResultMap map[string]int

finalResultMap = make(map[string]int)

for _, v := range result {

finalResultMap[v] = 1

}

var finalResultSlice []string

for k, _ := range finalResultMap {

downLoadPath := sp.Download(downLoadFile, "jpg", k, "")

if downLoadPath == "" {

continue

}

fmt.Printf("result %s\n", downLoadPath)

finalResultSlice = append(finalResultSlice, downLoadPath)

}

}

```

### 4.调用DownloadImg下载

以上都写好后就在main 中调用DownloadImg 设置deep = 1, 命令行 cd 到 mian.go `go run main.go` 启动下载。经过十多分钟的疯狂输出终于下好了，1深度就下了5000+,刺激

![下载图片](http://upload-images.jianshu.io/upload_images/7393873-048e5618453a68a7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "[下载图片]")

![下载内容简介](http://upload-images.jianshu.io/upload_images/7393873-ab57e623880649d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "[下载数据]")

至此，go初体验之爬取情侣头像就到这里结束啦，虽然有个bug在于找不到情侣头像的对应（希望别被女票打死）,但是这个过程还是对Golang这个语言有了很大程度的理解，收获满满，

总结下这次爬取的Coding之旅：

- 一定要注意类型！！！！PHP弱类型的通病让我突然"强迫症"式地用强类型就遇到好多次的类型异常；

- 还有就是 map定义没赋值 记得用之前make一下哦～

- go的标准包还是很nice的，要用的时候多翻翻文档哦

- go官网的tour很棒，可以一边看文档一边coding(慕课网的golang 基础课也有)

- go还有很多牛逼的特性，是PHP这类脚本型语言所不能及的，计划以后coding一波

本项目代码地址: https://github.com/LinsVert/Golang

以上。

![溜了](http://upload-images.jianshu.io/upload_images/7393873-43bc9f1291d569af.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 本人博客 ： [博客](blog.lindebiao.top)

#### 本人GayHub : [Github](https://github.com/LinsVert) 欢迎Start Follow!