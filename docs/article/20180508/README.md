<!-- README.md -->

# WKWebView简单使用

2018-05-08

## WKWebView介绍：

- 允许JavaScript的Nitro库加载并调用(UIWebView中限制)
- 支持更多的H5特性
- 将UIWebViewDelegate与UIWebView重构成了14个类与3个协议

```c
用法：
WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
//初始化偏好设置属性：preferences
config.preferences = [WKPreferences new];
//The minimum font size in points default is 0;
config.preferences.minimumFontSize = 10;
//是否支持JavaScript
config.preferences.javaScriptEnabled = YES;
//不通过用户交互，是否可以打开窗口
config.preferences.javaScriptCanOpenWindowsAutomatically = YES;
//通过JS与webView内容交互
config.userContentController = [WKUserContentController new];

//注册的方法
[config.userContentController addScriptMessageHandler:self name:@"js方法名"];

//初始化WKWebView
初始化方法:
WkWebView * wkwebView = [[WKWebView alloc] initWithFrame:frame configuration:config];
self.wkwebView.navigationDelegate = self;
self.wkwebView.UIDelegate = self;
NSURLRequest * request = [NSURLRequest requestWithURL:[NSURL URLWithString:str]];
[self.wkwebView loadRequest:request];


WKNavigationDelegate
//在响应完成时，调用的方法。如果设置为不允许响应，web内容就不会传过来
-(void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler{
   
   decisionHandler(WKNavigationResponsePolicyAllow);
}

//在发送请求之前，决定是否跳转
-(void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler{
   decisionHandler(WKNavigationActionPolicyAllow);  
}

// 页面开始加载时调用
- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(WKNavigation *)navigation;
// 当内容开始返回时调用
- (void)webView:(WKWebView *)webView didCommitNavigation:(WKNavigation *)navigation;

OC调用JS
// 页面加载完成之后调用
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation;
这个方法通常处理OC调用JS的方法
//javaScriptString是JS方法名，completionHandler是异步回调block
[self.webView evaluateJavaScript:javaScriptString completionHandler:completionHandler];

// 页面加载失败时调用
- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(WKNavigation *)navigation;


ScriptMessageHandler 
JS调用OC
接收到JS的方法名并处理,在使用这个方法之前一定要在初始化WKWebView的时候注册方法名，注册的方法名一定要和JS端的方法名一样

-(void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message{
NSLog(@"%@",message.name);
NSLog(@"%@",message.body);
}

一定要加上这句话
前端在配合使用WKWebView的时候
window.webkit.messageHandlers.<name>.postMessage(<messageBody>)


//处理内存的方法
@interface WeakScriptMessageDelegate : NSObject<WKScriptMessageHandler>

@property (nonatomic, weak) id<WKScriptMessageHandler> scriptDelegate;

- (instancetype)initWithDelegate:(id<WKScriptMessageHandler>)scriptDelegate;

@end

@implementation WeakScriptMessageDelegate

- (instancetype)initWithDelegate:(id<WKScriptMessageHandler>)scriptDelegate
{
   self = [super init];
   if (self) {
       _scriptDelegate = scriptDelegate;
   }
   return self;
}

- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message
{
   [self.scriptDelegate userContentController:userContentController didReceiveScriptMessage:message];
}

@end


WKUserContentController *userContentController = [[WKUserContentController alloc] init];    
[userContentController addScriptMessageHandler:[[WeakScriptMessageDelegate alloc] initWithDelegate:self] name:@"closeMe"];

然后在self的dealloc中添加
[[_webView configuration].userContentController removeScriptMessageHandlerForName:@"closeMe"];


```




