---
layout: post
title: "Javascript原生交互解决方"
date: 2016-06-01 20:13:42 +0800
comments: true
categories: objecitve-c, javascript,native
---
by bupo.
####简介
在终端应用开发过程中经常需要在H5页面中调用原生接口来使用原生服务，所以就有了Javascript与原生代码交互的需求，这里终结一下以前在项目中使用的一种解决方案。
原理很简单，通过在UIWebView的代理中截获window.location.href 跳转请求来响应Javascript请求，并通过UIWebview 提供的执行JS代码的接口stringByEvaluatingJavaScriptFromString 将原始执行结果回调给H5页面。
<!-- more -->
####具体实施
首先终端和H5确定好调用协议，这里以获取经纬度为例

	jsbridge://getLocation/callback=xx#seq
其中jsbridge是我们自定义的一个协议名，用来区分是调用原生接口的请求还是网络请求，getLocation是接口名称，"/"和“#”之前是参数对，多个参数用&链接，“#”后的seq是调用回调函数的时候传回给JS代码的序列号，用于在多次调用时区分是那一次调用，如果不需要回调给JS可以不传。
下面是H5代码示例，页面中有一个按钮，点击按钮发起原生调用请求。
	
```html

<!doctype html public "-//w3c//dtd html 4.0 transitional//en">
<html>
<head>
 <title> JS原生交互示例 </title>
 <meta charset="utf-8">
 <meta name="generator" content="editplus">
 <meta name="author" content="">
 <meta name="keywords" content="">
 <meta name="description" content="">
 <script language="javascript" type="text/javascript">

function call_getLocation(){
	window.location.href="jsbridge://getLocation/callback=showData#1"
}
function showData(s,info){
	alert(JSON.stringify(info));
}
 </script>
</head>
<body>
<input type="button" onclick="call_getLocation()" value="location"/>
</body>
</html>
```

在UIWebviewController的代理中捕获请求，判断是原生请求并调用相应接口。代码如下：
```objectivec
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    NSURL *requestURL = request.URL;
    NSString *urlString = [requestURL absoluteString];
    BOOL shouldStartLoad = YES;
    
    if ([urlString hasPrefix:@"jsbridge://"]) {
        [self decodeCMDandParams:[urlString substringFromIndex:kJSBridgePre.length]];
        shouldStartLoad = NO;
    }
    
    return shouldStartLoad;
}
```

decodeCMDandParams 方法解析出接口名和参数，利用OC runtime特性执行对应原生代码。

```
- (void)decodeCMDandParams:(NSString *)cmdAndParams
{
    QBLogDebug(@"request cmd and params:%@",cmdAndParams);
    NSArray *data = [cmdAndParams componentsSeparatedByString:@"/"];
    NSString *cmd = [data firstObject];
    NSDictionary *params = nil;
    if ([data count] > 1) {
        params = [self parseParams:data[1]];
    }
    
    [self invokeWithCMD:cmd andParams:params];
}

- (void)invokeWithCMD:(NSString *)cmd andParams:(NSDictionary *)params
{
    if (cmd == nil || cmd.length == 0) {
        return;
    }
    SEL selector = NSSelectorFromString([NSString stringWithFormat:@"Test_%@:",cmd]);
    if ([self respondsToSelector:selector]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [self performSelector:selector withObject:params];
#pragma clang diagnostic pop
        
    }
}
```
这样只需要和前端定好交互协议，并实现对应的原生接口就可以完成JS调用原生接口的功能。
接下来是将原生接口的执行结果回调给JS代码，这个比较简单，通过在参数中解析得到callback参数，将结果拼接成JS代码调用stringByEvaluatingJavaScriptFromString方法就行了。具体代码：
```
- (void)excuteCallback:(NSString *)callback argments:(NSDictionary *)dic
{
    if(callback == nil || callback.length <= 0){
        return;
    }
    if (!dic) dic = @{};
    NSInteger s = [[dic objectForKey:kSequence]integerValue];
    // dic to string
    NSString *arguments = [self serializeCallbackArgumentWithObject:dic];
    NSString *jsFunction = [NSString stringWithFormat:@"%@(%lu,%@)",callback,s,arguments];

    [self.webView stringByEvaluatingJavaScriptFromString:jsFunction];
}

- (NSString *)serializeCallbackArgumentWithObject:(NSDictionary *)object{
    NSData *data = [NSJSONSerialization dataWithJSONObject:object options:0 error:nil];
    NSString *str = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    return str;
}
```

完整的代码可以在这里[下载](https://github.com/bupojung/BPJSNativeInteract)