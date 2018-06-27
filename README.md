# 度秘BOT SDK for Node.js
这是一个帮助开发Bot的SDK，我们强烈建议您使用这个SDK开发度秘的Bot。当然，您可以完全自己来处理DuerOS的协议，自己完成session、nlu、result的处理，但是度秘的DuerOS对BOT的协议会经常进行升级，这样会给您带来一些麻烦。这个SDK会与DuerOS的协议一起升级，会最大限度减少对您开发bot的影响。

## 通过bot-sdk可以快速的开发bot
我们的目标是通过使用bot-sdk，可以迅速的开发一个bot，而不必过多去关注DuerOS对Bot的复杂协议。我们提供如下功能：

* 封装了DuerOS的request和response
* 提供了session简化接口
* 提供了nlu简化接口
    * slot 操作
    * nlu理解交互接口（ask）
* 提供了多轮对话开发接口
* 提供了事件监听接口

## 安装、使用BOT SDK进行开发 
度秘BOT SDK采用npm加载 , node.js版本确保在6.10及以上。使用执行如下命令进行安装：
```shell
npm install bot-sdk
```

为了开始使用BOT SDK，你需要先新建一个js文件，比如文件名是Bot.js。

```javascript
var BaseBot = require('bot-sdk');

class Bot extends BaseBot{
    constructor (postData) {
        super(postData);
    }
}

module.exports = Bot;
```
下一步，我们处理意图。Bot-sdk提供了一个函数来handle这些意图。比如，查询个税，创建一个handler，在构造函数中添加：

```php
this.addIntentHandler('LaunchRequest', ()=>{
    return {
        "outputSpeech" : "欢迎使用"
    };
});

this.addIntentHandler('personal_income_tax.inquiry', ()=>{
    let loc = this.getSlot('location');    
    let monthlySalary = this.getSlot('monthlysalary');

    if(!monthlySalary) {
        let card = new Bot.Card.TextCard('你工资多少呢');

        // 如果有异步操作，可以返回一个promise
        return new Promise(function(resolve, reject){
            resolve({
                card : card,
                outputSpeech : '你工资多少呢'
            });
        });
    }

    if(!loc) {
        let card = new Bot.Card.TextCard('你在哪呢');
        return {
            card: card,
            outputSpeech: '你在哪呢'
        };

    }
});
```
这里`addIntentHandler`可以用来建立(intent) => handler的映射，第一个参数是条件，如果满足则执行对应的回调函数(第二个参数)。
其中，this指向当前的Bot，`getSlot`继承自父类Bot，通过slot名字来获取对应的值。回调函数返回值是一个数组，可以包含多个字段，比如：`card`，`directives`，`outputSpeech`，`reprompt`。

## 搭建Web服务

比如，使用express作为web框架进行开发。下面是一个简单示例

```javascript
const express = require('express');

const Bot = require('./Bot');
var app = express();

// 探活请求
// DuerOS会定期发送探活请求到你的服务，确保你的服务正常运转，[详情请参考](http://TODO)
app.head('/', (req, res) => {
    res.sendStatus(204);
});

// 监听post请求，DuerOS以http POST的方式来请求你的服务，[具体协议请参考](http://TODO)
app.post('/', (req, res) => {
    req.rawBody = '';

    req.setEncoding('utf8');
    req.on('data', function(chunk) { 
        req.rawBody += chunk;
    });

    req.on('end', function() {
        var b = new Bot(JSON.parse(req.rawBody));
        // 开启签名认证
        // 为了避免你的服务被非法请求，建议你验证请求是否来自于DuerOS
        b.initCertificate(req.headers, req.rawBody).enableVerifyRequestSign();

        /**
         * 如果需要监控统计功能
         * 
         * bot-sdk 集成了监控sdk，参考：https://www.npmjs.com/package/bot-monitor-sdk
         * 打开此功能，对服务的性能有一定的耗时增加。另外，需要在DBP平台上面上传public key，这里使用私钥签名
         * 文档参考：https://dueros.baidu.com/didp/doc/dueros-bot-platform/dbp-deploy/authentication_markdown
         */
        b.setPrivateKey(__dirname + '/rsa_private_key.pem').then(function(key){
            // 0: debug  1: online
            b.botMonitor.setEnvironmentInfo(key, 0);

            b.run().then(function(result){
                res.send(result);
            });
        }, function(err){
            console.error('error'); 
        });

        
        // 不需要监控
        // b.run() 返回一个Promise的实例
        //b.run().then(function(result){
        //    res.send(result);
        //});
    });
}).listen(8014);
```


## API 文档

* [Bot](doc/Bot.md)
* [Nlu(Bot.nlu)](doc/Nlu.md)
* [Request(Bot.request)](doc/Request.md)
* [Response(Bot.response)](doc/Response.md)
* [Session(Bot.session)](doc/Session.md)
* [Certificate](doc/Certificate.md)
* 展现卡片
    * [BaseCard(所有卡片基类)](doc/card/BaseCard.md)
    * [TextCard(文本卡片)](doc/card/TextCard.md)
    * [StandardCard(标准卡片)](doc/card/StandardCard.md)
    * [ImageCard(图片卡片)](doc/card/ImageCard.md)
    * [ListCard(列表卡片)](doc/card/ListCard.md)
* 指令
    * [BaseDirective(所有指令基类)](doc/directive/BaseDirective.md)
    * app启动指令
        * [LaunchApp(app启动指令)](doc/directive/AppLuncher/LaunchApp.md)
    * 音频
        * [Play(音频播放指令)](doc/directive/AudioPlayer/Play.md)
        * [Stop(音频停止指令)](doc/directive/AudioPlayer/Stop.md)
        * [PlayerInfo(播放信息类)](doc/directive/AudioPlayer/PlayerInfo.md)
        * 音频控制组件
            * [BaseButton(按钮控件基础类)](doc/directive/AudioPlayer/Control/BaseButton.md)
            * [Button(普通按钮控件)](doc/directive/AudioPlayer/Control/Button.md)
            * [FavoriteButton(喜欢按钮控件)](doc/directive/AudioPlayer/Control/FavoriteButton.md)
            * [LyricButton(歌词按钮控件)](doc/directive/AudioPlayer/Control/LyricButton.md)
            * [NextButton(下一曲按钮控件)](doc/directive/AudioPlayer/Control/NextButton.md)
            * [PlayPauseButton(暂停播放按钮控件)](doc/directive/AudioPlayer/Control/PlayPauseButton.md)
            * [PreviousButton(上一曲按钮控件)](doc/directive/AudioPlayer/Control/PreviousButton.md)
            * [RadioButton(单选按钮控件)](doc/directive/AudioPlayer/Control/RadioButton.md)
            * [RecommendButton(推荐按钮控件)](doc/directive/AudioPlayer/Control/RecommendButton.md)
            * [RefreshButton(刷新按钮控件)](doc/directive/AudioPlayer/Control/RefreshButton.md)
            * [RepeatButton(单曲循环按钮控件)](doc/directive/AudioPlayer/Control/RepeatButton.md)
            * [ShowFavoriteListButton(展现收藏歌曲列表按钮控件)](doc/directive/AudioPlayer/Control/ShowFavoriteListButton.md)
            * [ShowPlayListButton(展现歌曲列表按钮控件)](doc/directive/AudioPlayer/Control/ShowPlayListButton.md)
            * [ThumbsUpDownButton(封面按钮控件)](doc/directive/AudioPlayer/Control/ThumbsUpDownButton.md)
    * 视频
        * [Play(视频播放指令)](doc/directive/VideoPlayer/Play.md)
        * [Stop(视频停止指令)](doc/directive/VideoPlayer/Stop.md)
    * 展现
        * 模版
           * [BaseTemplate(基础模版类)](doc/directive/Dispay/Template/BaseTemplate.md)
           * [BodyTemplate1(文本展现模板)](doc/directive/Dispay/Template/BodyTemplate1.md)
           * [BodyTemplate2(上图下文模版)](doc/directive/Dispay/Template/BaseTemplate2.md)
           * [BodyTemplate3(左图右文模版)](doc/directive/Dispay/Template/BodyTemplate3.md)
           * [BodyTemplate4(右图左文模版)](doc/directive/Dispay/Template/BodyTemplate4.md)
           * [BodyTemplate5(图片模板)](doc/directive/Dispay/Template/BodyTemplate5.md)
           * [ListTemplate(列表模版基础类)](doc/directive/Dispay/Template/ListTemplate.md)
           * [ListTemplate1(横向列表模板)](doc/directive/Dispay/Template/ListTemplate1.md)
           * [ListTemplate2(纵向列表模板)](doc/directive/Dispay/Template/ListTemplate2.md)
           * [ListTemplateItem(模版列表项)](doc/directive/Dispay/Template/ListTemplateItem.md)
           * [TextImageTemplate(图文模版)](doc/directive/Dispay/Template/TextImageTemplate.md)
           * 用户提示指令
               * [Hint(用户提示指令)](doc/directive/Dispay/Hint.md)
           * 模版渲染
               * [RenderTemplate(模版渲染)](doc/directive/Dispay/RenderTemplate.md)

    * 支付
        * [Charge(支付指令)](doc/directive/Pay/Charge.md)