---
layout: post
title: "iOS之WebViewJavascriptBridge原理及源码分析"
date: 2017-07-10 11:14:06 +0800
comments: true
categories: 
---

## 前言

在iOS开发中经常会用到native和h5的交互，本文主要介绍UIWebView（iOS8中可以用WKWebView，本文以UIWebView为例）和JavaScript交互的原理以及WebViewJavascriptBridge的源码分析。

## 1.WebViewJavascriptBridge原理

在梳理WebViewJavascriptBridge原理之前，先要明确两点：

- JavaScript不能直接调用native的方法

- native能直接调用JavaScript的代码，具体是通过下面的方式

  ```objective-c
  [webView stringByEvaluatingJavaScriptFromString:javascriptCommand];    
  ```

JavaScript虽然不能直接调用native的方法，但是可以通过一些其他的方式实现，对的，就是做url拦截（UIWebView是用的webView: shouldStartLoadWithRequest: navigationType:的代理方法，而WKWebView是用的webView: decidePolicyForNavigationAction: decisionHandler:的代理方法），webView每次发起网络请求的时候都会走上面的代理方法，如果此时我们在这里返回的是我们自定义的url，那么拦截到以后我们就可以处理自己的事情，既然要处理自己的事情，那就需要在h5中去触发webView的代理方法，这就牵扯到url的重定向，有很多办法可以解决这个问题，比如以下两种：

```
1. 创建 iframe 标签
    WebViewJavascriptBridge 中就是用的这种方法。
2. 设置 window 的 location
    window.location = "/www/phpStudy/JS/helloJS.html";
```

## 2.WebViewJavascriptBridge基本使用

### 在h5注册事件

1. 在h5端注册名为testJavascriptHandler的事件，并定义对来自native的消息处理函数function(message, responseCallback)，在处理完成以后，调用回调函数responseCallback(data)，将处理后的数据回调给native。

   ```objective-c
   bridge.registerHandler('testJavascriptHandler', function(data, responseCallback) {
   			log('ObjC called testJavascriptHandler with', data)
   			var responseData = { 'Javascript Says':'Right back atcha!' }
   			log('JS responding with', responseData)
   			responseCallback(responseData)
   		})		
   ```

2. 在native调用h5注册的testJavascriptHandler，将native的消息传递给h5，同时定义responseCallback块，来处理h5回调给native的数据。

   ```objective-c
   id data = @{ @"greetingFromObjC": @"Hi there, JS!" };
       [_bridge callHandler:@"testJavascriptHandler" data:data responseCallback:^(id response) {
           NSLog(@"testJavascriptHandler responded: %@", response);
       }];
   ```

### 在native注册事件

1. 在native注册名为testObjcCallback的事件，并定义对来自h5的消息的处理函数handler，并在处理完成后，调用回调函数responseCallback(data)，并将处理后的数据回调给h5。

   ```objective-c
    [_bridge registerHandler:@"testObjcCallback" handler:^(id data, WVJBResponseCallback responseCallback) {
           NSLog(@"testObjcCallback called: %@", data);
           responseCallback(@"Response from testObjcCallback");
       }];
   ```

2. 在h5调用native注册的testObjcCallback事件，将h5端的消息传递给native，同时定义responseCallback

   的block块，来处理native回调给h5的数据。

   ```objective-c
   var data = 'Hello from JS button'
   bridge.callHandler('testObjcCallback', {'foo': 'bar'}, function(response) {
   				log('JS got response', response)
   			})
   ```

## 3.WebViewJavascriptBridge的基本类

### WebViewJavascriptBridgeBase

#### 属性：

startupMessageQueue：在WebViewJavascriptBridge的js文件还没有载入以前，先把native发送给h5的message存储在startupMessageQueue数组中，一旦WebViewJavascriptBridge的js文件载入以后就遍历处理存储的消息。

responseCallbacks：用native调用h5时，定义的WVJBResponseCallback类型的block块，以objc_cb_xx（callbackId）为key，存储在responseCallbacks字典中。

messageHandlers：存储注册在native端的事件处理函数。以注册的事件名为key，以注册事件时定义的WVJBHandler类型的block块为value。

messageHandler：在native注册的默认的事件的处理函数。

delegate：指遵守WebViewJavascriptBridgeBaseDelegate协议的类，在这里指的就是WebViewJavascriptBridge/WKWebViewJavascriptBridge，这两个类都遵守该协议，并通过该协议的_evaluateJavascript方法，调用js命令。

#### 方法：

`+(void)enableLogging`：是否能够log

`+(void)setLogMaxLength:(int)length`：设置log最大长度

`-(void)reset`：重置方法

`-(void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName`：从native调用h5的方法

`-(void)flushMessageQueue:(NSString *)messageQueueString`：处理h5传递过来的消息字符串

`-(void)injectJavascriptFile`：注册js文件

`- (BOOL)isCorrectProcotocolScheme:(NSURL*)url`

`- (BOOL)isQueueMessageURL:(NSURL*)url`

`- (BOOL)isBridgeLoadedURL:(NSURL*)url`

`- (void)logUnkownMessage:(NSURL*)url`:打印未知消息

`- (NSString *)webViewJavascriptCheckCommand`：返回js命令，该命令用于检查js文件中WebViewJavascriptBridge的对象是否创建好。

`- (NSString *)webViewJavascriptFetchQueyCommand`：:返回js命令，该命令用于调用js中的_fetchQueue()函数

### WebViewJavascriptBridge

#### 方法：

`+ (instancetype)bridgeForWebView:(WVJB_WEBVIEW_TYPE*)webView`：WebViewJavascriptBridge的初始化方法

`+ (void)enableLogging`：是否能够log

`+ (void)setLogMaxLength:(int)length`：设置log最大长度

`- (void)registerHandler:(NSString*)handlerName handler:(WVJBHandler)handler`：注册事件方法

`- (void)callHandler:(NSString*)handlerName`；

`- (void)callHandler:(NSString*)handlerName data:(id)data`;

`- (void)callHandler:(NSString*)handlerName data:(id)data responseCallback:(WVJBResponseCallback)responseCallback`;

`- (void)setWebViewDelegate:(WVJB_WEBVIEW_DELEGATE_TYPE*)webViewDelegate`：设置代理

### WebViewJavascriptBridge_JS

#### 属性：

`sendMessageQueue`：将传递给native的消息，暂存在sendMessageQueue数组中

`messageHandlers`：存储注册在h5端的事件处理函数。以注册的事件名为key,以注册事件时定义的handler为value

`messagingIframe`：通过改变messagingIframe.src来调用webview的`- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType`方法，将处理流程回到native端

#### 方法：

registerHandler：注册事件

callHandler：调用native注册的方法

_fetchQueue：将sendMessageQueue中的消息序列化为字符串

_dispatchMessageFromObjC：处理来自objc的消息

## 3.WebViewJavascriptBridge源码解析

### 消息从native到h5

1. 以调用testJavascriptHandler事件为例（UIWebView），调用callHandler:data:responseCallback：函数

   ```objective-c
   id data = @{ @"greetingFromObjC": @"Hi there, JS!" };
       [_bridge callHandler:@"testJavascriptHandler" data:data responseCallback:^(id response) {
           NSLog(@"testJavascriptHandler responded: %@", response);
       }];
   ```

2. 调用`- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName` 方法，最终组装成message为

   { callbackId = “objc_cb_2”; data ={greetingFromObjC = “Hi there, JS!”;}; handlerName = testJavascriptHandler;}

   ```objective-c
   - (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString* )handlerName {
       NSMutableDictionary* message = [NSMutableDictionary dictionary];
       
       if (data) {
           message[@"data"] = data;
       }
       
       if (responseCallback) {
           NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
           self.responseCallbacks[callbackId] = [responseCallback copy];
           message[@"callbackId"] = callbackId;
       }
       
       if (handlerName) {
           message[@"handlerName"] = handlerName;
       }
       [self _queueMessage:message];
   }
   ```

3. 调用`_queueMessage:`方法

   ```objective-c
   - (void)_queueMessage:(WVJBMessage*)message {
       if (self.startupMessageQueue) {
           [self.startupMessageQueue addObject:message];
       } else {
           [self _dispatchMessage:message];
       }
   }
   ```

4. 调用`_dispatchMessage：`最终在js中执行handleMessageFromObjC方法，执行流程进入js中

   ```objective-c
   - (void)_dispatchMessage:(WVJBMessage*)message {
       //序列化message
       NSString *messageJSON = [self _serializeMessage:message];
       [self _log:@"SEND" json:messageJSON];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
      //js命令
      NSString * javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
       if ([[NSThread currentThread] isMainThread]) {
           //js中调用_handleMessageFromObjC(messageJSON)方法
           [self _evaluateJavascript:javascriptCommand];

       } else {
           dispatch_sync(dispatch_get_main_queue(), ^{
               [self _evaluateJavascript:javascriptCommand];
           });
       }
   }
   ```

5. 接着执行`_dispatchMessageFromObjC(messageJSON)`，反序列化messageJSON得到message

   { callbackId = “objc_cb_2”; data ={greetingFromObjC = “Hi there, JS!”;}; handlerName = testJavascriptHandler;}

   ```objective-c
   function _dispatchMessageFromObjC(messageJSON) {
   		setTimeout(function _timeoutDispatchMessageFromObjC() {
   			var message = JSON.parse(messageJSON);
   			var messageHandler;
   			var responseCallback;
   			//此时responseId不存在
   			if (message.responseId) {
   				responseCallback = responseCallbacks[message.responseId];
   				if (!responseCallback) {
   					return;
   				}
   				responseCallback(message.responseData);
   				delete responseCallbacks[message.responseId];
   			} else {
   				if (message.callbackId) {
   					var callbackResponseId = message.callbackId;
   					//定义responseCallback
   					//responseCallback被调用，则调用_doSend函数
   					responseCallback = function(responseData) {
   						_doSend({ responseId:callbackResponseId, responseData:responseData });
   					};
   				}
   				//取出对应事件的处理函数
   				var handler = messageHandlers[message.handlerName];
   				try {
   				//调用handler(,)，如果native需要回调数据，那么会在handler中调用responseCallback();
   					handler(message.data, responseCallback);
   				} catch(exception) {
   					console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
   				}
   				if (!handler) {
   					console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
   				}
   			}
   		});
   	}
   ```

6. 调用`_doSend(message, responseCallback)`函数,此时message为

   { responseId:”objc_cb_2”, responseData:{ ‘Javascript Says’:’Right back atcha!’ }},并且把message存储到sendMessageQueue

   ```objective-c
   function _doSend ( message, responseCallback) {
   		if (responseCallback) {
   			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime()
   			responseCallbacks[callbackId] = responseCallback
   			message['callbackId'] = callbackId
   		}
   		sendMessageQueue.push(message)
   		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE
   	}
   ```

   注意：responseData为testJavascriptHandler的消息处理函数，处理后的数据

   ```objective-c
   bridge.registerHandler('testJavascriptHandler', function(data, responseCallback) {
   			log('ObjC called testJavascriptHandler with', data)
   			var responseData = { 'Javascript Says':'Right back atcha!' }
   			log('JS responding with', responseData)
   			responseCallback(responseData)
   		})
   ```

7. 由于上一步给messagingIframe.src重新赋值，所以url重定向，调用代理方法 `-  (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType`

   ```objective-c
   - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
       if (webView != _webView) { return YES; }
       NSURL *url = [request URL];
       __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
       if ([_base isCorrectProcotocolScheme:url]) {
           if ([_base isBridgeLoadedURL:url]) {
               [_base injectJavascriptFile];
           } else if ([_base isQueueMessageURL:url]) {
           //[_base webViewJavascriptFetchQueyCommand]:生成js命令,WebViewJavascriptBridge._fetchQueue();
           //执行js命令，得到序列化的消息字符串:messageQueueString
               NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
               [_base flushMessageQueue:messageQueueString];
           } else {
               [_base logUnkownMessage:url];
           }
           return NO;
       } else if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:shouldStartLoadWithRequest:navigationType:)]) {
           return [strongDelegate webView:webView shouldStartLoadWithRequest:request navigationType:navigationType];
       } else {
           return YES;
       }
   }
   ```

8.  调用`flushMessageQueue:`方法,得到message为 {responseId:”objc_cb_2”, responseData:{ ‘Javascript Says’:’Right back atcha!’ }}，并根据responseId，从responseCallbacks中取出对应的block，进行回调，然后删除这对键值。

   ```objective-c
   - (void)flushMessageQueue:(NSString *)messageQueueString{
       if (messageQueueString == nil || messageQueueString.length == 0) {
           NSLog(@"WebViewJavascriptBridge: WARNING: ObjC got nil while fetching the message queue JSON from webview. This can happen if the WebViewJavascriptBridge JS is not currently present in the webview, e.g if the webview just loaded a new page.");
           return;
       }
     //反序列化得到messages
       id messages = [self _deserializeMessageJSON:messageQueueString];
       for (WVJBMessage* message in messages) {
           if (![message isKindOfClass:[WVJBMessage class]]) {
               NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
               continue;
           }
           [self _log:@"RCVD" json:message];
           
           NSString* responseId = message[@"responseId"];
         //此时responseId存在
           if (responseId) {
             //取出存储的WVJBResponseCallback类型的block块
               WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
             //将从h5回调回来的数据传递给responseCallback
               responseCallback(message[@"responseData"]);
             //移除这对键值
               [self.responseCallbacks removeObjectForKey:responseId];
           } else {
               WVJBResponseCallback responseCallback = NULL;
               NSString* callbackId = message[@"callbackId"];
               if (callbackId) {
                   responseCallback = ^(id responseData) {
                       if (responseData == nil) {
                           responseData = [NSNull null];
                       }
                       
                       WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                       [self _queueMessage:msg];
                   };
               } else {
                   responseCallback = ^(id ignoreResponseData) {
                       // Do nothing
                   };
               }
               
               WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
               
               if (!handler) {
                   NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                   continue;
               }
               
               handler(message[@"data"], responseCallback);
           }
       }
   }
   ```

### 消息从h5到native

1. 以调用testObjcCallback为例

   ```objective-c
   bridge.callHandler('testObjcCallback', {'foo': 'bar'}, function(response) {
   				log('JS got response', response)
   			})
   //callHandler的定义	
   function callHandler(handlerName, data, responseCallback) {
   		_doSend({ handlerName:handlerName, data:data }, responseCallback)
   			}
   ```

2. 调用 `_doSend(message, responseCallback)`,最终message为

   {handlerName:@”testObjcCallback” , data:{‘foo’: ‘bar’}, callbackId: cb_1_1477301893983} ，并且把消息存储到sendMessageQueue中

   ```objective-c
   function _doSend(message, responseCallback) {
   		if (responseCallback) {
   			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime()
   			responseCallbacks[callbackId] = responseCallback
   			message['callbackId'] = callbackId
   		}
   		sendMessageQueue.push(message)
   		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE
   		}
   ```

3. messagingIframe.src被重新赋值，重定向到`- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType` 函数

   ```objective-c
   - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
       if (webView != _webView) { return YES; }
       NSURL *url = [request URL];
       __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
       if ([_base isCorrectProcotocolScheme:url]) {
           if ([_base isBridgeLoadedURL:url]) {
               [_base injectJavascriptFile];
           } else if ([_base isQueueMessageURL:url]) {
           //[_base webViewJavascriptFetchQueyCommand]:生成js命令,WebViewJavascriptBridge._fetchQueue();
           //执行js命令，得到序列化的消息字符串:messageQueueString
               NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
               [_base flushMessageQueue:messageQueueString];
           } else {
               [_base logUnkownMessage:url];
           }
           return NO;
       } else if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:shouldStartLoadWithRequest:navigationType:)]) {
           return [strongDelegate webView:webView shouldStartLoadWithRequest:request navigationType:navigationType];
       } else {
           return YES;
       }
   ```

4. 调用flushMessageQueue:方法,序列化得到message为 {handlerName:@”testObjcCallback” , data:{‘foo’: ‘bar’}, callbackId: cb_11477301893983} ，并根据callbackId,从responseCallbacks中取出对应的block，进行回调

   ```objective-c
   - (void)flushMessageQueue:(NSString *)messageQueueString{
       if (messageQueueString == nil || messageQueueString.length == 0) {
           NSLog(@"WebViewJavascriptBridge: WARNING: ObjC got nil while fetching the message queue JSON from webview. This can happen if the WebViewJavascriptBridge JS is not currently present in the webview, e.g if the webview just loaded a new page.");
           return;
       }

       id messages = [self _deserializeMessageJSON:messageQueueString];
       for (WVJBMessage* message in messages) {
           if (![message isKindOfClass:[WVJBMessage class]]) {
               NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
               continue;
           }
           [self _log:@"RCVD" json:message];
           
           NSString* responseId = message[@"responseId"];
          //此时responseId不存在
           if (responseId) {
               WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
               responseCallback(message[@"responseData"]);
               [self.responseCallbacks removeObjectForKey:responseId];
           } else {
               WVJBResponseCallback responseCallback = NULL;
               NSString* callbackId = message[@"callbackId"];
               if (callbackId) {
                 //定义responseCallback
                   responseCallback = ^(id responseData) {
                       if (responseData == nil) {
                           responseData = [NSNull null];
                       }
                       
                       WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                       [self _queueMessage:msg];
                   };
               } else {
                   responseCallback = ^(id ignoreResponseData) {
                       // Do nothing
                   };
               }
               //根据handlerName从messageHandlers取出handler
               WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
               
               if (!handler) {
                   NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                   continue;
               }
               //调用handler
               handler(message[@"data"], responseCallback);
           }
       }
   }
   ```

5. 从上一步回调responseCallback,从而调用queueMessage方法, 此时的message为{responseId:@”cb_11477301893983” , responseData:@”Response from testObjcCallback”}

   ```objective-c
   - (void)_queueMessage:(WVJBMessage*)message {
       if (self.startupMessageQueue) {
           [self.startupMessageQueue addObject:message];
       } else {
           [self _dispatchMessage:message];
       }
   }
   ```

6. 调用_dispatchMessage:方法

   ```objective-c
   - (void)_dispatchMessage:(WVJBMessage*)message {
       //序列化message
       NSString *messageJSON = [self _serializeMessage:message];
       [self _log:@"SEND" json:messageJSON];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
       messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
       //生成js命令
       NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
       if ([[NSThread currentThread] isMainThread]) {
           //调用_handleMessageFromObjC(messageJSON)方法
           [self _evaluateJavascript:javascriptCommand];

       } else {
           dispatch_sync(dispatch_get_main_queue(), ^{
               [self _evaluateJavascript:javascriptCommand];
           });
       }
   }
   ```

7. 调用_handleMessageFromObjC(messageJSON)方法

   ```objective-c
   function _handleMessageFromObjC(messageJSON) {
   		if (receiveMessageQueue) {
   			receiveMessageQueue.push(messageJSON)
   		} else {
   			_dispatchMessageFromObjC(messageJSON)
   		}
   	}
   ```

8. 调用dispatchMessageFromObjC(messageJSON),反序列化得到message为{responseId:@”cb_1_1477301893983” , responseData:@”Response from testObjcCallback”}，根据responseId,从responseCallbacks中取出对应的responseCallback函数，进行对应的回调处理。然后从responseCallbacks删除这对键值

   ```objective-c
   function _dispatchMessageFromObjC(messageJSON) {
   		setTimeout(function _timeoutDispatchMessageFromObjC() {
   			var message = JSON.parse(messageJSON);
   			var messageHandler;
   			var responseCallback;
             //responseId存在，responseCallbacks
   			if (message.responseId) {
   				responseCallback = responseCallbacks[message.responseId];
   				if (!responseCallback) {
   					return;
   				}
   				responseCallback(message.responseData);
   				delete responseCallbacks[message.responseId];
   			} else {
   				if (message.callbackId) {
   					var callbackResponseId = message.callbackId;
   					responseCallback = function(responseData) {
   						_doSend({ responseId:callbackResponseId, responseData:responseData });
   					};
   				}
   				
   				var handler = messageHandlers[message.handlerName];
   				try {
   					handler(message.data, responseCallback);
   				} catch(exception) {
   					console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
   				}
   				if (!handler) {
   					console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
   				}
   			}
   		});
   	}
   ```

   至此，native和h5互相之间调用的流程就全部梳理清楚了，如果文中有什么错误，欢迎大家指正。