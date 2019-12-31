---
title: 投机倒把之邀请撸ccvt币
date: 2018-12-16 15:29:10
tags: 投机倒把
categories: 投机倒把
---

#### 写在开头：投机倒把的一般都凉了。一如我薅的这个被叫做ccvt的还未上交易所的币，号称是聊天就能挖矿，刺激得一笔，本人的投机倒把最终以被ban了邀请码不得了之。话不多说进入正题。
<!-- more -->

![](https://user-gold-cdn.xitu.io/2018/12/26/167eac6d85fb4d35?w=300&h=300&f=gif&s=46254)

起因是在本人在某个除了技术其他都聊的技术群里的得知一位老哥所在的公司开发了个叫ccvt的币种，在群里宣传，可以通过邀请新用户获得50个ccvt币，然后还提及这个会在明年2月份上交易所并开通提现功能，我一想这个不就可以套现，而且这个邀请注册估计就那么回事，可能可以投机倒把，果断注册看了一把，发现可以通过利用接码平台来实现虚拟号码注册从而达到邀请的目的。

欢迎到本人[博客](https://blog.lindebiao.top/2018/12/26/%E6%8A%95%E6%9C%BA%E5%80%92%E6%8A%8A%E4%B9%8B%E9%82%80%E8%AF%B7%E6%92%B8%E6%9F%90%E5%B8%81/)上查看。
- 本文用到的工具：
```
Postman 懂得都懂
PHP7 （如果需要发送邮件报警需要 PHPMailer）
laravel5.6
chrome浏览器（废话）
易码（接码平台）
百度云--文字识别之数字识别（免费的 每天500次上限）
smtp邮箱（非必须）
服务器一台（也可以是自己的破机）
```
本文编程语言使用PHP，其他语言大同小异，至于为啥用PHP--PHP是世界上最好的语言！百度云需要申请文字识别的应用,因为免费的识别率不是很高，不过够用了。易码平台是一个接码平台--相当于一个虚拟手机号可以接收验证码。

### Let's Code:

#### 1. 接口分析
   
![注册页面](https://user-gold-cdn.xitu.io/2018/12/26/167ead4cd051dac6?w=1092&h=1310&f=png&s=129647)
通过图片可以看到注册页面包含：
```
手机号 图片验证码 短信验证码 2次密码 邀请码
```
随便注册一下，用chrome控制台看下：

![](https://user-gold-cdn.xitu.io/2018/12/26/167ead7e4d4ea64e?w=1764&h=1092&f=png&s=321025)
可以获取主要请求参数：
```
callback jq的回调
country_code  手机号所属的国家编号 中国的是 86
cellphone 手机号
sms_code 短信验证码
pass_word 密码
pass_word_hash 密码的hash
invit_code 邀请码
_ 时间戳
```
其他无关紧要的就不写了,发现参数里没有涉及到图片验证码，所以应该用于短信发送的时候验证了，测试一把发现果然如此。

![图片验证码错误](https://user-gold-cdn.xitu.io/2018/12/26/167eae197b69161a?w=2734&h=996&f=png&s=348281)
看下接口：

![](https://user-gold-cdn.xitu.io/2018/12/26/167eae2441a44788?w=1354&h=736&f=png&s=154080)
可以获取发送验证码的请求参数：
```
callback
cellphone
country_code
bind_type 默认1
cfm_code 图片验证码
```
那么再获取到有效的验证码的图片就能开始开发正式投机倒把。
验证码图片接口：

![](https://user-gold-cdn.xitu.io/2018/12/26/167eae55788891b8?w=1426&h=528&f=png&s=41885)
```
https://ccvt.io/api/inc/code.php
```
生成的PHPSession Cookie:

![](https://user-gold-cdn.xitu.io/2018/12/26/167eae8710530b5a?w=1780&h=670&f=png&s=125552)
#### 2. 瞎几把分析

根据目前的几个接口可以有以下流程：
 
 - 1.请求接口 获取验证码图片，并下载下来（这个时候会生成一个关键的PHPSession 需要注意必要的话可以保存）
 - 2.把下载过来的图片请求到百度云的数字识别接口，获取数字化的图片验证码
 - 3.通过易码的接口获取手机号，并请求发送验证码的接口，发送验证码
 - 4.轮询易码的相关接口获取验证码
 - 5.基于第4步，请求注册接口
 
可能遇到的问题：
- 1.可能会获取不到图片
- 2.百度云可能因为qps超限或者超过当日请求量而报错
- 3.验证码识别失败，发送验证码接口请求失败
- 4.验证码获取不到
- 5.因某些原因注册失败

#### 3. Coding
##### PS:代码中的 request类是copy过来的，只是简单封装了curl请求类的方法,可以通过方法的命名猜到它的作用。
3.1 获取图片验证码
```
    /**
     * 获取图片验证码
     */
    public function getAuthCode()
    {
        //这个请求会生成唯一的 sessionId 需要注意接口异常问题
        $errorTime = 10;
        $start = 0;
        Resend:
        //todo 获取代理ip
        $ip = "";
        if ($ip) {
            $this->request->set_proxy($ip);
        }
        $this->request->set_referer($this->referer);
        $code = $this->request->get($this->code_url);
        if ($code) {
            $cookie = $this->request->get_cookies(self::CCVT_URL);
            if (!$cookie) {
                if ($start < $errorTime) {
                    $start++;
                    goto Resend;
                }
                $this->sendEmail('脚本告警', "获取cookie失败" . date("Y-m-d H:i:s"));
                exit;
            } else {
                self::$session = $cookie;
                $this->downloadCode($code);
                $this->getCode();
            }
        }
    }
    
    /**
     * 保存验证码图片
     * @param $image
     */
    public function downloadCode($image)
    {
        $file = app_path('Storage/ccvt/code/');
        if (!is_dir($file)) {
            mkdir($file, 0777, true);
        }
        $file .= 'code.png';
        file_put_contents($file, $image);
    }
    
    /**
     * 获取验证码
     */
    public function getCode()
    {
        //通过百度ocr接口拿验证码存在一定的准确率
        $file = app_path('Storage/ccvt/code/');
        $file = $file . "code.png";
        if (!file_exists($file)) {
            $this->sendEmail('脚本告警', "获取验证码图片失败!" . date("Y-m-d H:i:s"));
        } else {
            $ocr = new Ocr();
            $code = $ocr->getCode($file);
            if (is_numeric($code)) {
                self::$code = $code;
            } else {
                $this->sendEmail('脚本告警', "百度接口识别错误!返回内容:</br>" . json_encode($code, JSON_UNESCAPED_UNICODE), true);
                exit;
            }
        }
    }

```
流程是： 获取验证码->保存验证码->请求百度云

请求百度云的方法，具体实现可以查看 [百度云](https://cloud.baidu.com/doc/OCR/OCR-API.html#.E8.AF.B7.E6.B1.82.E8.AF.B4.E6.98.8E) 的接口：
```
public function getCode($path)
    {
        if (!$path) {
            return false;
        }
        $request = new Request();
        $request->set_header('Content-Type', 'application/x-www-form-urlencoded');
        //图片base64
        $image = $this->base64Image($path);
        $postUrl = self::OCR_NUMBER_URL . '?access_token=' . $this->getAccessToken();
        $param = [
            'image' => $image,
        ];
        $result = $request->post($postUrl, $param);
        $code = '';
        if ($result) {
            $result = json_decode($result);
            if (isset($result->words_result)) {
                //因为百度云返回的不是一次识别，需要循环拿出
                foreach ($result->words_result as $key => $value) {
                    $code .= $value->words;
                }
                $code = (int)$code;
            } else {
                return $result;
            }
        }
        return $code;
    }
```

3.2 获取手机号并发送验证码

易码的相关接口就不贴了，可以看下 [文档](http://www.51ym.me/User/apidocs.html):

```
     /**
     * 获取手机号码
     */
    public function getTelephone()
    {
        $getUrl = sprintf($this->Telephone_Api, $this->token, $this->item_num);
        $result = $this->request->get($getUrl);
        if ($result) {
            $results = explode('|', $result);
            if ($results[0] == 'success') {
                if (is_numeric($results[1])) {
                    self::$yima_telephone = $results[1];
                } else {
                    //发送邮件告警
                    $this->sendEmail('脚本告警', "获取yima手机号失败 code 2!接口返回内容:</br>" . $result, true);
                    exit;
                }
            } else {
                //发送邮件告警
                $this->sendEmail('脚本告警', "获取yima手机号失败 code 1!接口返回内容:</br>" . $result, true);
                exit;
            }
        }
    }
    
     /**
     * 发送验证码
     */
    public function sendSmsCode()
    {
        $getUrl = sprintf($this->send_sms, self::$yima_telephone, self::$code, time() * 1000);
        $errorTime = 10;
        $start = 0;
        Resend:
        $this->request->set_referer($this->referer);
        $result = $this->request->get($getUrl);
        if ($result) {
            $_result = str_replace("(", '', $result);
            $_result = str_replace(');', '', $_result);
            $_result = json_decode($_result) ?? '';
            if (isset($_result->errcode) && $_result->errcode != 0) {
                $this->freeTelephone();
                $this->sendEmail('脚本告警', "验证码发送失败: 内容如下:</br>" . "url:" . $getUrl . "</br>返回内容:" . $result, true);
                exit;
            } elseif (!isset($_result->errcode)) {
                if ($start < $errorTime) {
                    $start++;
                    goto Resend;
                }
                $this->freeTelephone();
                $this->sendEmail('脚本告警', "验证码发送失败3: 内容如下:</br>" . "url:" . $getUrl . "</br>返回内容:" . $result, true);
                exit;
            }
        } else {
            if ($start < $errorTime) {
                $start++;
                goto Resend;
            }
            $this->freeTelephone();
            $this->sendEmail('脚本告警', "验证码发送失败2: 返回数据为空 内容如下:</br>" . "url:" . $getUrl . "</br>返回内容:" . $result, true);
            exit;
        }
    }
    
    /**
     * 获取短信验证码
     */
    public function getSmsCode()
    {
        //需要轮询 60秒已经很长了
        $getUrl = sprintf($this->GetMessage_Api, $this->token, $this->item_num, self::$yima_telephone);
        $nowTime = time();
        $endTime = $nowTime + 60;
        do {
            if ($endTime < time()) {
                //超时
                break;
            }
            $result = $this->request->get($getUrl);
            if ($result) {
                $_result = explode('|', $result);
                if ($_result[0] == 'success') {
                    $content = $_result[1];
                    echo "短信获取成功:" . $content . PHP_EOL;
                    //释放
                    if (preg_match("/\d+/", $content, $codes)) {
                        self::$sms_code = $codes[0];
                    } else {
                        $this->sendEmail('脚本告警', "获取验证码失败! :</br>" . $content, true);
                        $this->freeTelephone();
                        break;
                    }
                    $this->freeTelephone();
                    break;
                } elseif ($_result[0] != 3001) {
                    //不是3001状态的直接不要这次的注册 发送邮件
                    $this->freeTelephone();
                    $this->sendEmail('脚本告警', "轮询手机号码异常！返回内容如下:</br>" . $result, true);
                    break;
                }
            }
            //5秒要一次 可能一次要到了 睡5秒也无妨
            sleep(5);
        } while (!self::$sms_code);
    }
    
    /**
     * 释放手机号
     */
    public function freeTelephone()
    {
        $getUrl = sprintf($this->Free_Telephone_Api, $this->token, $this->item_num, self::$yima_telephone);
        echo "手机号释放:" . $this->request->get($getUrl) . PHP_EOL;
    }
    
```
脚本跑下来这里一般是因为图片验证码错误失败，百度云这个识别率真的感人。。。。

3.3 最后一步！！注册
拼装好参数，请求就完事儿了：

```
/**
     * 5.注册
     */
    public function register()
    {
        if ($this->checkParam()) {
            $getUrl = sprintf($this->register_url, self::$yima_telephone, self::$sms_code, time() * 100);
            $result = $this->request->get($getUrl);
            if ($result) {
                //因为有callback所以需要解析
                $_result = str_replace("(", '', $result);
                $_result = str_replace(');', '', $_result);
                $_result = json_decode($_result) ?? '';
                if (isset($_result->errcode) && $_result->errcode == 0) {
                    //注册成功
                    $this->sendEmail('注册成功通知', "使用的手机号为:</br>" . self::$yima_telephone . "注册时间:</br>" . date("Y-m-d H:i:s"), true);
                } else {
                    echo "注册失败!" . date('Y-m-d H:i:s') . PHP_EOL;
                    echo "使用手机号:" . self::$yima_telephone . "使用验证码:" . self::$sms_code . PHP_EOL;
                    echo "返回数据结构:" . json_encode($_result, JSON_UNESCAPED_UNICODE) . PHP_EOL;
                }
            }
        }
    }
    
    /**
     * 校验必要的参数
     * @return bool
     */
    private function checkParam()
    {
        if (!self::$yima_telephone) {
            return false;
        }
        if (!self::$sms_code) {
            return false;
        }
        return true;
    }
    
```
到此主要代码就鲁完了，还可以加上代理啊，邮件报警系统呀，余额预警呀，之类的，
注册成功后，进个人中心看到到账的ccvt币心里还是很美滋滋的。

![邀请记录](https://user-gold-cdn.xitu.io/2018/12/26/167eb26d61dae126?w=2336&h=842&f=png&s=151413)
然后将代码放到laravel的任务调度(定时任务)去，发布到服务器上就可以全自动投机倒把了～～～～

可惜好景不长，在代码运行了3天后发现注册成功邮件都不发了，上服务器一看日志，发现都是注册失败的提示，本人果断手动执行一下发现还是注册失败，然后去掉了邀请码，发现就注册成功了，emmmmm,我觉得还是因为没加代理，被他们的技术人员把邀请码拉入黑名单了。投机倒把就这么没了，本人也写下这边文章记录下自己当时的过程～最后，投机倒把还是少干，xD。

PS：这个网站好几个接口存在sql注入的可能，之前测数据的时候发现接口会报sql错误，可能有sql注入的点，我没深入研究，后来再访问就没有报错了，应该是修复了（滑稽）。

最后附上该项目的 [gayhub](https://github.com/LinsVert/Miscellaneous) 欢迎各位 start 或 follow



