# 搞清微信端 H5 音乐的加载和自动播放

## 背景

在微信端H5的项目里，音乐播放功能我曾经一直是通过`soundjs`来实现的，主要是因为`soundjs`兼容性很好，安卓和ios都可以实现**加载**和**自动播放**功能。
但直到某次微信升级后，ios端开始必须通过用户点击才可以播放音乐，所以`soundjs`就被弃用了，改成了`<audio>`原生标签，至于自动播放功能，改成了使用微信浏览器的事件去实现，后面会详细说明。
但弃用`soundjs`也就意味着其加载功能就不能用了。

很长时间里我一直认为不可能同时做到**加载**和**自动播放**。首先ios浏览器对流媒体有限制，再加上微信浏览器的一些特殊性。从业务角度来说，**背景音乐**往往又需要同时满足这两点。
随着对前端的理解逐渐加深，慢慢这两个问题也都有了各自的解决办法。

## 解决办法

实际上，音乐的加载和自动播放就是两个不同的问题，是可以拆开来讨论的。以下就是我对这两个问题列出的一些解决方案：

### 加载
原来`createjs`不仅提供了`soundjs`，还提供了`preloadjs`配套进行更完整的加载处理。当然你也可以选择不加载，毕竟一个音乐文件几十k或者一百多k，和大量的图片资源比就显得不重要了。下面说说我常用的三种方案：

1.  不加载

    索性就不加载，因为网速快啊。`<audio preload="auto">`就完了，在需要执行播放时直接获取 dom 元素调用`play()`即可。
	
	> 大家可能会喷，说你写到现在，就写个不加载？？你在逗我？？其实在真实项目中，不加载的情况确实很多。一是因为允许你写加载逻辑的时间很少。现在H5项目周期越来越短，加载功能又往往到验收阶段才能有时间好好整理，导致加载音乐就被忽视了。二是因为现在网速确实快了不少。
	
	> `<audio>`标签的`preload`最好加上，不用纠结有用没用，有就对了。
	
	其实这种方式已经满足了绝大部分的场景，毕竟很多 H5 就一个背景音乐。只要首页不承担大量的素材加载，基于现在的网络条件，加载单独一个音频文件几乎秒出，所以延迟会有，甚至会提前，但不会太糟。
	
2.  使用`soundjs`

	[soundjs官网](https://createjs.com/soundjs)
	
	这是我目前用的最多的加载方案。
	
    `soundjs` 提供了`registerSound`方法来处理音频加载
	

    ```js
	function loadHandler(event) {
	    createjs.Sound.play('sound');
	}
	
    createjs.Sound.alternateExtensions = ['mp3'];
    createjs.Sound.on('fileload', loadHandler, this);
    createjs.Sound.registerSound('bgmusic.mp3', 'sound');
    ```

3.  使用`soundjs + preloadjs`
	
	[preloadjs官网](https://createjs.com/preloadjs)

    > `preloadjs`原本是一个加载库，不仅可以加载音频，还可以加载图片，json 等各式文件，用来处理整体进度相当有用。所以采用由`preloadjs`负责加载，而`soundjs`负责播放的方式来执行。

    ```js
	function handleComplete() {
	    createjs.Sound.play('sound');
	}
	
    var queue = new createjs.LoadQueue();
    queue.installPlugin(createjs.Sound);
    queue.on('complete', handleComplete, this);
    ```

### 自动播放

安卓和ios对于自动播放的支持是不一样的。安卓几乎全部支持，但ios都不支持，必须经过用户手动点击屏幕时候才能播放音乐（当初我懵懂无知还以为实`soundjs`不管用将其抛弃，555~）。后来发现，在微信浏览器里是有两种方法即使是ios也可以实现自动播放：

1. WeixinJSBridgeReady

   这是我目前用的最多的自动播放方案。

   任何运行在微信浏览器中的 H5 页面，都会监听到一个`WeixinJSBridgeReady`事件，在这个事件中就可以播放音乐。
   > 纯主观推测：可能`WeixinJSBridgeReady`对系统有非常高的权限，因为是微信浏览器中对网页暴露出的第一个事件。

    ```js
    document.addEventListener(
        'WeixinJSBridgeReady',
        function() {
            // 播放背景音乐
            var bgMusic = document.getElementById('bgmusic');
            bgMusic.play();
        },
        false
    );
    ```
	
	`WeixinJSBridgeReady`方法有个缺点，**注册音乐**必须写到`WeixinJSBridgeReady`中，才能实现自动播放。示例如下：

    ```js
    var soundLoadComplete = false; // 背景音乐是否加载完毕
    var backgroundMusicInstance = null; // 背景音乐实例
    createjs.Sound.alternateExtensions = ['mp3'];
    createjs.Sound.on('fileload', loadHandler, this);
    
    function loadHandler(event) {
        soundLoadComplete = true;
    }
    
    // 可以自动播放的场景
    document.addEventListener(
        'WeixinJSBridgeReady',
        function() {
            // 关键代码。下面这句 registerSound 必须写在 WeixinJSBridgeReady 回调函数内才行，
            // createjs.Sound.play 就会无效
            createjs.Sound.registerSound('bgmusic.mp3', 'sound'); 
            var intInstance = setInterval(function() {
                if (soundLoadComplete) {
                    clearInterval(intInstance);
                    backgroundMusicInstance = createjs.Sound.play('sound'); // soundjs 执行播放命令
                }
            }, 10);
        },
        false
    );

    // 无法自动播放的场景
    // 在bridge外部就启动了加载，导致WeixinJSBridgeReady内部在执行播放命令之前，
    // 都没有Sound这个对象的实例，所以没法自动播放音乐
    createjs.Sound.registerSound('bgmusic.mp3', 'sound'); 
    document.addEventListener(
        'WeixinJSBridgeReady',
        function() {
            var intInstance = setInterval(function() {
                if (soundLoadComplete) {
                    clearInterval(intInstance);
                    backgroundMusicInstance = createjs.Sound.play('sound'); // soundjs执行播放命令
                }
            }, 10);
        },
        false
    );
    ```
    
2. jweixin

	[微信官方文档](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/JS-SDK.html)
	
	[jweixin，截止目前更新到1.6.0](http://res.wx.qq.com/open/js/jweixin-1.6.0.js)
    
	`jweixin`是微信官方提供的js-sdk，在其 `wx.ready` 事件中，也能实现自动播放音乐：

	```js
        // 微信配置信息 即使不正确也没问题，但是一定要有这一步
        wx.config({
            debug: false,
            appId: '',
            timestamp: 1,
            nonceStr: '',
            signature: '',
            jsApiList: [],
        });

        // 在ready时触发相关事件
        wx.ready(function() {
            // 播放背景音乐
            var bgMusic = document.getElementById('bgmusic');
            bgMusic.play();
        });
	```
	与`WeixinJSBridgeReady`不同，`jweixin`不需要**注册音乐**写在`wx.ready`中。
	

## 方案demo
排列组合一下上述**加载**和**自动播放**的办法，就得到了下面所有的细化方案：
- [原生 js，不加载音乐，页面启动后直接播放](https://superzdd.github.io/wechat-h5-backgound-music-survey/index.html)
- [原生 js，不加载音乐，监听 `WeixinJSBridgeReady` 事件播放音乐 **微信专用**](https://superzdd.github.io/wechat-h5-backgound-music-survey/index1.html)
- [使用 `wx.js`，不加载音乐，页面启动后在 `wx.ready` 事件里播放音乐 **微信专用**](https://superzdd.github.io/wechat-h5-backgound-music-survey/index1-1.html)
- [使用 `soundjs` 加载并播放](https://superzdd.github.io/wechat-h5-backgound-music-survey/index2.html)
- [使用 `soundjs`，监听 `WeixinJSBridgeReady` 事件播放音乐 **微信专用**](https://superzdd.github.io/wechat-h5-backgound-music-survey/index3.html)
- [使用 `soundjs`，`wx.js`，页面启动后在 `wx.ready` 事件里播放音乐 **微信专用**](https://superzdd.github.io/wechat-h5-backgound-music-survey/index3-1.html)
- [使用 `preloadjs`，`soundjs` 加载并播放](https://superzdd.github.io/wechat-h5-backgound-music-survey/index4.html)
- [使用 `preloadjs`，`soundjs`，监听 `WeixinJSBridgeReady` 事件播放音乐 **微信专用**](https://superzdd.github.io/wechat-h5-backgound-music-survey/index5.html)
- [使用 `preloadjs`，`soundjs`，`wx.js`，页面启动后在 `wx.ready` 事件里播放音乐 **微信专用**](https://superzdd.github.io/wechat-h5-backgound-music-survey/index5-1.html)

1. 手机设备
    - 安卓
    - 苹果

2. 手机浏览器
    - 微信
    - 原生浏览器
    - 其他浏览器，QQ，UC

3. 所有插件链接
    - [weixin](https://res.wx.qq.com/open/js/jweixin-1.4.0.js)
    - [soundjs](https://code.createjs.com/1.0.0/soundjs.min.js)
    - [preloadjs](https://code.createjs.com/1.0.0/preloadjs.min.js)

### 方案测试结果
测试结果可能会随着今后手机版本的升级慢慢改变
#### 加载
`soundjs`和`preloadjs`都能实现加载音频的功能。
> 推荐单独使用`soundjs`，因为vue和react框架的理念和原生js的理念不一致，同时用两个不仅导致资源增加，而且开发理念不一样的地方也更多，给后期理解代码造成一定困难。

#### 自动播放
- 微信端里，自动播放都可以实现
- 微信端外，ios都无法自动播放
