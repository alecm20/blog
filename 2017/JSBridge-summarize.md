# JSBridge总结

## JSBridge简析
前端开发中，JS一般是运行在浏览器中。由于安全原因，JS和设备本身的交互较为受限。
随着设备性能的提高，我们对Web体验的要求也越来越高。Web页面要承载的内容也越来越多，逐步发展成Web App。
而Native App中的WebView能够让Web页面展示在App中，这就进一步混淆了Native和Web的界限。
而运行在WebView中的Web页面这时候是不能够调用Native的功能的，为了更好的体验，我们想要在Web页面中调用Native的API。
这时候，JSBridge就应运而生。故名思议，Bridge，充当的角色就是连接Native和Web的桥梁，让Web页面能够和Native进行沟通。
之前的PhoneGap之类的框架其实就是运用的JSBridge，用前端代码生成一个App。现在的App混合开发，称为Hybrid App，就是用的这一技术。

## JSBridge原理
Hybrid App的前端页面运行在App的WebView中，App能够对WebView中的各种行为进行拦截操作，运用此特性，以此来完成我们的JSBridge功能。
进行拦截操作的方式有两种：**拦截URL SCHEME** **注入API**两种方式。拦截URL SCHEME是WebView在发起网络请求的时候，客户端能够拦截到发起的网络请求，从而进行相应的操作。
注入API则是在WebView内部的JS引擎中，注入JS方法，供前端调用。前端调用注入的方法的时候，客户端捕获相应的调用，执行对应的操作，达到JS和Native交互的目的。

## JSBridge实现思路
### JS调用Native
#### 拦截URL SCHEME
这种方式是客户端打开的URL进行拦截，解析出相应的内容，执行操作。
我们普通的URL格式```http://action?test=123```，在这里我们需要定制一种格式，和普通的请求作区分。
我们可以修改请求的协议部分，把http协议替换成我们的自定义协议，客户端检测到自定义协议后，转而执行相应的操作而不是打开普通的页面。
例如：```jsbridge://action?test=123```，这里我们用jsbridge开头，传递的信息放在query string里面。客户端去解析query string，执行操作。

比如我们进行分享：

```
jsbridge://action?type=share&title=Title&des=Description
```

通过这种方法进行了一次分享操作。

前端发起这种请求一般是通过创建一个新的iframe的方式实现的，如下：

```javascript
var url = 'jsbridge://doAction?title=分享标题&desc=分享描述&link=http%3A%2F%2Fwww.baidu.com';
var iframe = document.createElement('iframe');
iframe.style.width = '1px';
iframe.style.height = '1px';
iframe.style.display = 'none';
iframe.src = url;
document.body.appendChild(iframe);
setTimeout(function() {
    iframe.remove();
}, 100);
```

这种方法相比下面我们要讲的注入API的方式有一些缺陷：

1.创建iframe有一定的时间花费

2.拼接URL的方式传递参数，URL会有长度限制

之前使用这种方式是因为这种方法支持iOS6，现在已经不需要支持iOS6了，所以我们可以抛弃这种方式，换用更好的方法。

【注】：有些方案为了规避 url 长度隐患的缺陷，在 iOS 上采用了使用 Ajax 发送同域请求的方式，并将参数放到 head 或 body 里。这样，虽然规避了 url 长度的隐患，但是 WKWebView 并不支持这样的方式。

【注2】：为什么选择 iframe.src 不选择 locaiton.href ？因为如果通过 location.href 连续调用 Native，很容易丢失一些调用。

#### 注入API

注入API是向WebView的JS运行环境的window全局对象上添加方法。JS能够调用被注入的方法，从而实现客户端和Web的交互。

对于iOS的UIWebView：

```objectivec
JSContext *context = [uiWebView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];

context[@"postBridgeMessage"] = ^(NSArray<NSArray *> *calls) {
    // Native 逻辑
};
```

前端调用：

```javascript
window.postBridgeMessage(message);
```

iOS的WKWebView：

```objectivec
@interface WKWebVIewVC ()<WKScriptMessageHandler>

@implementation WKWebVIewVC

- (void)viewDidLoad {
    [super viewDidLoad];

    WKWebViewConfiguration* configuration = [[WKWebViewConfiguration alloc] init];
    configuration.userContentController = [[WKUserContentController alloc] init];
    WKUserContentController *userCC = configuration.userContentController;
    // 注入对象，前端调用其方法时，Native 可以捕获到
    [userCC addScriptMessageHandler:self name:@"nativeBridge"];

    WKWebView wkWebView = [[WKWebView alloc] initWithFrame:self.view.frame configuration:configuration];

    // TODO 显示 WebView
}

- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
    if ([message.name isEqualToString:@"nativeBridge"]) {
        NSLog(@"前端传递的数据 %@: ",message.body);
        // Native 逻辑
    }
}
```
前端调用：

```javascript
window.webkit.messageHandlers.nativeBridge.postMessage(message);
```

对于Android：

```java
public class JavaScriptInterfaceDemoActivity extends Activity {
	private WebView Wv;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        Wv = (WebView)findViewById(R.id.webView);     
        final JavaScriptInterface myJavaScriptInterface = new JavaScriptInterface(this);    	 

        Wv.getSettings().setJavaScriptEnabled(true);
        Wv.addJavascriptInterface(myJavaScriptInterface, "nativeBridge");

        // TODO 显示 WebView

    }

    public class JavaScriptInterface {
         Context mContext;

         JavaScriptInterface(Context c) {
             mContext = c;
         }

         public void postMessage(String webMessage){	    	
             // Native 逻辑
         }
     }
}
```
前端调用：

```javascript
window.nativeBridge.postMessage(message);
```

由于方法一的缺陷，现在都采用注入API的方式来实现JSBridge。此处我们实现的JSBridge也是采用的注入API的方式完成的。

### Native调用JS
客户端调用JS比较简单，有现成的API可用。

iOS的UIWebView：

```objectivec
result = [uiWebview stringByEvaluatingJavaScriptFromString:javaScriptString];
```

iOS的WKWebView：

```objectivec
[wkWebView evaluateJavaScript:javaScriptString completionHandler:completionHandler];
```

Android 4.4之前：

```java
webView.loadUrl("javascript:" + javaScriptString);
```

Andorid 4.4及以后：

```java
webView.evaluateJavascript(javaScriptString, new ValueCallback<String>() {
    @Override
    public void onReceiveValue(String value) {

    }
});
```

【注】：使用 loadUrl 的方式，并不能获取 JavaScript 执行后的结果。

## 前端JSBridge实现

下面要讲的是前端实现JSBridge的思路。
前端和客户端一次完整的交互包括：
前端执行一个动作，告诉客户端需要执行的操作，以及把客户端所需的信息传递给客户端；客户端执行完操作后，把客户端所需的信息传递给前端。

以一次分享操作为例：
```
前端：客户端醒醒，我要执行分享操作啦
客户端：啥？
前端：我要执行分享操作啦，给你看我要的操作：{type: share}
客户端：好了好了，我知道了
客户端：正在分享，哎，你要分享啥东西
前端：这是我的信息：{title: 'I am title', des: 'I am description'}
客户端：ok
客户端：...
前端：？？？
前端：？？？
前端：醒醒，刚才我交代的事办的怎么样了
客户端：啥？我想想...
客户端：记不起来了...（挠头）
前端：...
```
上面我们的例子中不能接收客户端的执行结果，所以要加上一个回调，在客户端执行完相应的操作后，执行这个回调。
我们并不需要把回调函数传递给客户端，注意我们上面讲的Native执行JS代码的一节。我们可以把回调函数注册在全局对象的一处，然后把函数名称传递给客户端。
这样客户端在执行完操作后，直接执行对应的回调函数，把结果放在回调的参数中，这样我们就能够拿到客户端返回的结果了。

所以按照这种思路，我们就可以这样执行一次分享操作：

```javascript
window.jsBridge.postMessage({
    type: 'share',
    option: {
        title: 'I am title',
        des: 'I am description'
    },
    cbId: 'cb001'
})

window.cb001 = function(data) {
    console.log(data)
}
```

这样就完成了一次操作。

对于实际的前端调用，从易用性的角度考虑，我们可能会这样调用：

```javascript
JSBridge.share({
    title: 'I am title',
    des: 'I am description',
    cb: function(data) {
        console.log(data)
    }
})
```

观察这两种用法，我们的JSBridge库要完成两件事：
1.执行JSBridge.share转换成执行window.jsBridge.postMessage({type: 'share'})
2.把cb注册在window下的某处地方，然后把对应的id放入postMessage方法。
所以我们的库可以这么写：

```javascript
var i = 0
window.jsBridge_cbObj = window.jsBridge_cbObj || {}

var JSBridge = window.JSBridge = {
    share: function(data) {
        window.jsBridge_cbObj['cb'+(i++)] = data.cb
        window.jsBridge.postMessage({
            type: 'share',
            option: {
                title: 'I am title',
                des: 'I am description'
            },
            cbId: 'cb'+(i-1)
        })
    }
}
```

接下来我们就可以按照这种方式使用JSBridge：

```javascript
import JSBridge from './JSBridge.js'
JSBridge.share({
    title: 'I am title',
    des: 'I am description',
    cb: function(data) {
        console.log(data)
    }
})
```

上面是对特定的操作进行处理，实际中我们的JSBridge需要对外提供多种方法，我们可以对上面的JSBridge库的代码做一定的优化：

```javascript
;(function(window){
    var i = 0;
    var JSBridge = window.JSBridge = {}
    window.cbFns = {}
    var type = [
        'toast',
        'alert',
        'confirm',
        'share',
        'setTitle'
    ]
    function init(obj, array) {
        array.forEach(function(item){
            obj[item] = function(data){
                var id = 'cb'+i
                i++
                var cbFn = function(resData){
                    data.cb && data.cb(resData)
                    delete window.cbFns[id]
                }
                window.cbFns[id] = cbFn
                window.__JSBridge.postMessage({
                    type: item,
                    option: data,
                    cid: id
                })
            }
        })
    }
    init(JSBridge, type)
})(window)
```

上面就是初步的JSBridge雏形。我们对type数组内的方法遍历，添加到JSBridge上。

进一步，我们可以区分回调的类型：

像这样调用：

```javascript
JSBridge.share({
    title: 'title',
    des: 'description',
    success: function(data) {
        console.log('success', data)
    },
    fail: function() {
        console.log('fail', data)
    },
    cancel: function() {
        console.log('cancel', data)
    }
})
```

客户端返回回调内容时，给我们一个type指示操作的结果。该type可以为'success', 'fail', 'cancel'等，分别
指示回调成功、失败、取消。

另外，JSBridge的功能有很多，我们可以对功能进行一定的归类。例如：getLocation和getUserInfo我们
可以统一使用getData作为postMessage的外层类型。可以这样：

```javascript
postMessage({
    type: 'getData',
    option: {
        type: 'location'
    },
    cb: 'cb0'
})
```

客户端解析时，对getLocation和getUserInfo这种统一解析，后续需要额外的信息时，通过getData接口进行拓展。
同时，对于前端调用的方法来说，从易用性方面考虑，对外提供JSBridge.getLocation和JSBridge.getUserInfo方法，同时也提供JSBridge.getData方法，
方便前端获取自定义信息。

## 参考资料

[移动混合开发中的 JSBridge](https://mp.weixin.qq.com/s/I812Cr1_tLGrvIRb9jsg-A)

[H5与Native交互之JSBridge技术](http://tech.youzan.com/jsbridge/)

[微信JS-SDK说明文档](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421141115)