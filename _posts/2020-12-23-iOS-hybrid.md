
---
title:  "Hybrid 笔记"
date:   2020-12-22 22:11:36 +0530
author: taoye
categories: [iOS, 2020年复现总结]
tags: [iOS]
---

### 好文章分享
[hybrid三部曲](http://awhisper.github.io/2018/01/02/hybrid-jscomunication/)
hybrid方案 - 小钗老板的 

> 本人没咋搞过hybrid，估计很多内容都是错的😆

### wk和UI区别 （查一下）
多进程，在app的主进程之外执行

使用更快的Nitro JavaScript引擎

异步执行处理JavaScript
支持对错误的自签名安全证书和证书进行身份验证

### webview和js交互的几种方式（查一下）
**URL schema:**
有丢消息和消息长度的限制。
**javaScriptCore:**
JavaScriptCore可能导致的问题：
① 注入时机不唯一（也许是BUG）
② 刷新页面的时候，JavaScriptCore的注入在不同机型表现不一致，有些就根本不注入了，所以全部hybrid交互失效

OS 使用 WKWebView - scriptMessageHandler 注入，这种方式注入其实只给注入对象起了一个名字nativeObject，这种对象只有一个函数 postMessage
//准备要传给native的数据，包括指令，数据，回调等
```
var data = {
    method:'location',
    data:'http://baidu.com',
    callback:'1',
};
//传递给客户端
window.webkit.messageHandlers.nativeObject.postMessage(data);
```
客户端接收
```
-(void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
    NSDictionary *msgBody = message.body;
    NSString *method = msgBody[@"method"];
    NSString *data = msgBody[@"data"];
    // perform method with data
    // ...
}
```

### wk的白屏问题
    在 WKWebView 上当总体的内存占用比较大的时候，WebContent Process 会 crash，从而出现白屏现象。
    这个时候 WKWebView.URL 会变为 nil, 简单的 reload 刷新操作已经失效，对于一些长驻的H5页面影响比较大。
    解决方案：
    1. 当 WKWebView 总体内存占用过大，页面即将白屏的时候，系统会调用上面的回调函数，我们在该函数里执行[webView reload](这个时候 webView.URL 取值尚不为 nil）解决白屏问题。在一些高内存消耗的页面可能会频繁刷新当前页面，H5侧也要做相应的适配操作。    iOS 9以后 WKNavigtionDelegate 新增了一个回调函数：- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView API_AVAILABLE(macosx(10.11), ios(9.0));
    2. 检测 webView.title 是否为空， 并不是所有H5页面白屏的时候都会调用上面的回调函数，比如，最近遇到在一个高内存消耗的H5页面上 present 系统相机，拍照完毕后返回原来页面的时候出现白屏现象（拍照过程消耗了大量内存，导致内存紧张，WebContent Process 被系统挂起），但上面的回调函数并没有被调用。在WKWebView白屏的时候，另一种现象是 webView.titile 会被置空, 因此，可以在 viewWillAppear 的时候检测 webView.title 是否为空来 reload 页面。
    
### wk的cookie问题

传统的NSHTTPCookieStorage
通过 NSHTTPCookieStorage 设置的 Cookie ，这样设置的Cookie 无论是 UIWebView 页面请求还是 NSURLSession 网络请求，都会带上 Cookie，所以十分方便

WKWebView 发起的请求并不会带上 NSHTTPCookieStorage 里面的 Cookie, 而比如用户登陆状态token等，最基础的设计就是把 token 写到 cookie 里，如果 WebView 获取不到 Cookie 的登陆状态应该怎么办

简单的说就是把 WKWebView 发起的 NSURLRequest 拦截，MutableCopy 一个，然后手动在RequestHeader里从NSHTTPCookieStorage读取Cookie进行添加

```
-(void)syncRequestCookie:(NSMutableURLRequest *)request
{
    if (!request.URL) {
        return;
    }
    
    NSArray *availableCookie = [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:request.URL];
    NSMutableArray *filterCookie = [[NSMutableArray alloc]init];
 
    if (filterCookie.count > 0) {
        NSDictionary *reqheader = [NSHTTPCookie requestHeaderFieldsWithCookies:filterCookie];
        NSString *cookieStr = [reqheader objectForKey:@"Cookie"];
        [request setValue:cookieStr forHTTPHeaderField:@"Cookie"];
    }
    return;
}
```

当服务器发生重定向的时候，此时第一次在 RequestHeader 中写入的 Cookie 会丢失，还需要重新对重定向的 NSURLRequest 进行 RequestHeader 的 Cookie 处理 ，简单的说就是在 webView:decidePolicyForNavigationAction:decisionHandler: 的时候，判断此时 Request 是否有你要的 Cookie 没有就Cancel掉，修改Request 重新发起



### UA
iOS 8及 8 以下只能进行全局 UA 修改，可以通过 NSUserDefaults 的方式修改，一次修改每个WebView都有效（无论是常规 WebView 还是被你改造过的 Hybrid WebView）

NSDictionary *dictionary = [NSDictionary dictionaryWithObjectsAndKeys:UAStringXXX, @"UserAgent", nil];
[[NSUserDefaults standardUserDefaults] registerDefaults:dictionary];

iOS 9 有了独立UA，可以针对每一个 WKWebView 对象实例，设置专属的UA

if (@available(iOS 9.0, *)) {
    self.webView.customUserAgent = self.fullUserAgent;
}

### 离线能力建设
把 H5 打包下载到本地，然后使用 NSURLProtocol 拦截请求把本地资源替换线上资源。但是这种方案有个问题，需要使用私有 API，具有一定的风险。使用私有 API 进行拦截，还有另外一个问题，就是 POST 请求会丢失 body，所以尽量只拦截 GET 请求。

到了 iOS 11 就可以使用系统提供的 [setURLSchemeHandler:forURLScheme:] 实现离线。
但 WKURLSchemeHandler 不能处理 Http、Https 等常规 scheme，所以需要自定义 scheme。
基本方案就是，在 WebView loadRequest 前判断本地是否有离线资源，支持离线且有离线资源的时候，修改 Http/Https 为自定义的 scheme，然后在 NSURLProtocol 或者 WKURLSchemeHandler去实现对本地资源的加载。


### 如何拦截网络(搜一下)
UIWebview, NSURLProtocol
WKWebView WKSchemeHalder 好像不支持http/https?

### 耗时统计

