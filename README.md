# 搞清微信端 H5 音乐的加载和自动播放

## 背景

我们的项目以微信端 H5 为主。在音乐播放的问题上，最开始的时候使用到了`soundjs`，但自从有一版微信升级后，重新产生了** ios 端必须通过用户点击才可以自动播放音乐**的问题后，就取消了`soundjs`，改为`<audio>`原生标签，效果也不差。

长期以来，基于 ios 浏览器对流媒体的限制，比如预加载和自动播放问题，再加上微信浏览器的一些特殊性，我一直认为没有插件可以同时做到预加载和自动播放，甚至在业务场景下，这两个问题有时是一个问题，比如：背景音乐。很多情况下背景音乐在 Loading 页面就开始播放了，相当于加载完毕立即播放。

但实际上，音乐的加载和自动播放就是两个不同的问题，是可以拆开来进行讨论的。以下就是我对这两个问题列出的一些解决方案：

## 问题一 加载

当初一直不确定 `soundjs` 是否真的能做加载音乐，可能是经验问题吧，加上 ios 又不能自动播放，所以最后把 `soundjs` 这个方案直接取消了。但回过头来再看的时候，发现不仅有`soundjs`,`createjs`还提供了`preloadjs`配套进行更完整的加载处理，所以下面的几个方案会把`preloadjs`也一起考虑进去

1.  不加载

    > 即不做任何加载处理，`<audio preload="auto">`就完了，至于`preload`到底有没有作用，我觉得不用纠结，反正加上去一定是对的，在需要执行播放时直接获取 dom 元素调用`play()`即可。其实这种方式已经满足了绝大部分的场景，毕竟很多 H5 就一个背景音乐，实际有音效的不多，但是`soundjs`,`preloadjs`这些库文件是实实在在的开销，没必要为了一个音频文件搭上那么多库。而且，就像刚才提到的，背景音乐在很多情况下，是属于一加载完就需要播放的类型，剩下的大部分场景，是经过 Loading 进入首页后才进行播放，在这种情况下，只要首页不承担额外的加载任务，基于现在的网络条件，加载单独一个音频文件几乎是秒出，所以导致看似播放背景音乐没有什么影响和延迟。

2.  使用`soundjs`

    > `soundjs` 提供了`registerSound`来处理音频加载

    ```js
    createjs.Sound.alternateExtensions = ['mp3'];
    createjs.Sound.on('fileload', this.loadHandler, this);
    createjs.Sound.registerSound('bgmusic.mp3', 'sound');
    ```

3.  使用`soundjs + preloadjs`

    > `preloadjs`原本是一整个加载库，不仅可以加载音频，还可以加载图片，json 等各式文件，用来处理整体进度相当有用，所以这里由`preloadjs`完全负责加载，而`soundjs`只负责播放。

    ```js
    var queue = new createjs.LoadQueue();
    queue.installPlugin(createjs.Sound);
    queue.on('complete', handleComplete, this);

    function handleComplete() {
        soundLoadComplete = true;
    }
    ```

## 问题二 自动播放

安卓和 ios 对于自动播放的支持是不一样的。安卓手机几乎全部支持自动播放，但是 ios 就不行，不需经过用户手动点击屏幕时候才能播放音乐。而当初就是因为 ios 不能实现自动播放这个功能，才将`soundjs`抛弃。结果后来发现，在微信浏览器下，有两种方案可以越过 ios 的这个障碍，这两个方案分别是：

1. wxJSBridge

    > 任何在微信浏览器中运行的 H5 页面，都会监听到一个`WeixinJSBridgeReady`事件，据我自己的推测，可能这个事件是微信浏览器中对网页暴露出的第一个事件，所以给到了非常高的权限，甚至可以播放音乐！所以有了下面的自动播放写法：

    ```js
    document.addEventListener(
        'WeixinJSBridgeReady',
        function() {
            playBackgroundMusic();
        },
        false
    );
    ```

2. wx.js

    > 这里的`wx.js`是简称，全称叫做`jweixin-1.4.0.js`，名字比较长，所以简称`wx.js`。在 wx.js 的 ready 事件中，也能进行自动播放音乐的处理：

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
        // 微信
        makeLog('wx ready');
        var intInstance = setInterval(function() {
            if (soundLoadComplete) {
                clearInterval(intInstance);
                makeLog('play');
                backgroundMusicInstance = createjs.Sound.play('sound');
            }
        }, 10);
    });
    ```

我们组合一下加载和自动播放的这些方案，就得到了后面所有的细化方案测试项，下面就是进行细化的测试：

## 详细的测试流程

1. 手机设备

    - 安卓
    - 苹果

2. 手机浏览器

    - 微信
    - 原生浏览器
    - 其他浏览器，QQ，UC

3. 使用到的所有插件链接
    - [weixin](https://res.wx.qq.com/open/js/jweixin-1.4.0.js)
    - [soundjs](https://code.createjs.com/1.0.0/soundjs.min.js)
    - [preloadjs](https://code.createjs.com/1.0.0/preloadjs.min.js)
4. 测试流程

    - 原生 js，不加载音乐，页面启动后直接播放
    - 原生 js，不加载音乐，监听 `WeixinJSBridgeReady` 事件播放音乐 **微信专用**
    - 使用 `wx.js`，不加载音乐，页面启动后在 `wx.ready` 事件里播放音乐 **微信专用**
    - 使用 `soundjs` 加载并播放
    - 使用 `soundjs`，监听 `WeixinJSBridgeReady` 事件播放音乐 **微信专用**
    - 使用 `soundjs`，`wx.js`，页面启动后在 `wx.ready` 事件里播放音乐 **微信专用**
    - 使用 `preloadjs`，`soundjs` 加载并播放
    - 使用 `preloadjs`，`soundjs`，监听 `WeixinJSBridgeReady` 事件播放音乐 **微信专用**
    - 使用 `preloadjs`，`soundjs`，`wx.js`，页面启动后在 `wx.ready` 事件里播放音乐 **微信专用**

## 结果

1.  `soundjs`和`preloadjs`都能实现加载音频的功能，所以在比如音频多，或者对音乐有准确实际要求的项目里，可以使用这两个 js 库进行开发。

2.  自动播放在微信端里，这两种方式都可以实现。但是，除了微信端以外，的所有 ios 其他浏览器都无法自动播放。

    2.1. WeixinJSBridgeReady

    > `WeixinJSBridgeReady`方法有个最大的缺点，也就是播放音乐的引用必须传入`WeixinJSBridgeReady`方法中才能实现自动播放的功能。示例如下：

    ```js
    // 可以自动播放的场景
    document.addEventListener(
        'WeixinJSBridgeReady',
        function() {
            createjs.Sound.registerSound('bgmusic.mp3', 'sound'); // 关键代码。注意，这句 registerSound 必须写在 WeixinJSBridgeReady 回调函数内才行,否则下方 createjs.Sound.play 就会无效
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
    createjs.Sound.registerSound('bgmusic.mp3', 'sound'); // 在bridge外部就启动了加载，导致WeixinJSBridgeReady内部在执行播放命令之前都没有国Sound这个对象的实例，所以没法自动播放音乐
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

    2.1. wx.js

    > 在使用`wx.js`的方案中，不必考虑到回调函数中必须有`sound`的引用，但是要加一段`wx.config...`的代码，其实也挺繁琐的，但好在不用破坏代码的逻辑结构。示例如下：

    ```js
    // 微信配置信息 即使不正确也没问题，但是这里必须要写
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
        var intInstance = setInterval(function() {
            if (soundLoadComplete) {
                clearInterval(intInstance);
                makeLog('play');
                backgroundMusicInstance = createjs.Sound.play('sound');
            }
        }, 10);
    });
    ```
