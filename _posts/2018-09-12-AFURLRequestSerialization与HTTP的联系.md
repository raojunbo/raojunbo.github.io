---
layout: post
title: AFURLRequestSerialization与HTTP的联系
date: 2018-09-12 23:34:15.000000000 +09:00
tags: iOS技术
---
在阅读此部分前，需要将HTTP协议有个比较详细的了解。此部分就是对这HTTP头与体的一个包装过程。

## AFURLRequestSerialization
AFURLRequestSerialization顾名思义就是请求的序列化，用于帮助构建NSURLRequest，主要做了两个事情：
1. 构建普通请求：格式化参数，生成HTTP Header
2. 构建multipart请求

如下是我抓的”滴答清单“APP的一个借口

```
	GET /api/v2/user/preferences/settings/iOS HTTP/1.1
Host	dida365.com
Content-Type	application/json;charset=UTF-8
Accept	*/*
Cookie	t=EBD7D0AF0FE1D94C46DCD71D3873E5C7C56CEF8D5456D538057B76BB2D0DCC979341356EE2BAE2E6DFE4EA2AF3F00B7A569ABEFA649AE578ED8C5B714068DACF9EC18B99739EA18E0B4E1F1E5D2FE6AFD6411C1CD0870886847BCB312E593174A9B3E0ECE97A229EF34D86482F9AB7501CA21DB6C75447375915F30F79F90E1A7571BEE34B2A622B1E231D602F4941D0FE41AB93F1DEC0CC4EA47B456F671C69493AF5B02DCE4096E857C7465BADAFC6
Connection	keep-alive
X-Device	X-Device: iOS 10.3.3,iPhone,4551,07bffe2a0e6f4535b295380254618296,ios_iphone
locale	zh_CN
User-Agent	TickTick/4.5.51 (iPhone; iOS 10.3.3; Scale/2.00)
Accept-Language	zh-Hans-CN;q=1, en-CN;q=0.9
Authorization	OAuth EBD7D0AF0FE1D94C46DCD71D3873E5C7C56CEF8D5456D538057B76BB2D0DCC979341356EE2BAE2E6DFE4EA2AF3F00B7A569ABEFA649AE578ED8C5B714068DACF9EC18B99739EA18E0B4E1F1E5D2FE6AFD6411C1CD0870886847BCB312E593174A9B3E0ECE97A229EF34D86482F9AB7501CA21DB6C75447375915F30F79F90E1A7571BEE34B2A622B1E231D602F4941D0FE41AB93F1DEC0CCC4E27F93766D0416493AF5B02DCE4096E857C7465BADAFC6
Accept-Encoding	gzip, deflate
```
从上面可以看出，包括了请求方式GET,请求的URL,HTTP的版本，Host,Accept,Cookie,User-Agent,Accept-Language,Accept-Encoding,Connection，这写的构建工作都是在AFURLRequestSerialization来帮助开发者完成。

stringEncoding
allowCellularAccess
cachePolicy
HTTPShouldHandleCookies
HTTPShouldUsePipeling
timeoutInterval


- 构建请求头
请求头的构建AF给我们创建了Accept-Language,User-Agent,Authorization，Content-Type等

- 构建查询参数
若果是Get请求的话，需要将用户传给的字典拼接成?username=rjb&pwd=123456的形式。最终拼接到url后面。

- 构建请求体
若果是Post请求的话，请求体的形式一般有如下4种。

    1. 第一种：application/json：这是最常见的json格式如下
{"input1":"xxx","input2":"ooo","remember":false}
 
    2. 第二种：application/x-www-form-urlencoded：浏览器的原生 form 表单，如果不设置 enctype 属性，那么最终就会以 application/x-www-form-urlencoded 方式提交数
input1=xxx&input2=ooo&remember=false
 

    3. 第三种：multipart/form-data:这一种是表单格式的，数据类型如下


    4. 第四种：text/xml:这种直接传的xml格式

    所以整个构建Post请求体的过程也是AFURLRequestSerialization很大一部分工作。AF采用的是AFStreamingMultipartFormData类来构建的。
    
### AFStreamingMultipartFormData

```
@interface AFStreamingMultipartFormData ()
@property (readwrite, nonatomic, copy) NSMutableURLRequest *request;
@property (readwrite, nonatomic, assign) NSStringEncoding stringEncoding;
@property (readwrite, nonatomic, copy) NSString *boundary;
@property (readwrite, nonatomic, strong) AFMultipartBodyStream *bodyStream;
@end
```
```
@interface AFMultipartBodyStream : NSInputStream <NSStreamDelegate>
@property (nonatomic, assign) NSUInteger numberOfBytesInPacket;
@property (nonatomic, assign) NSTimeInterval delay;
@property (nonatomic, strong) NSInputStream *inputStream;
@property (readonly, nonatomic, assign) unsigned long long contentLength;
@property (readonly, nonatomic, assign, getter = isEmpty) BOOL empty;
```
```
@interface AFHTTPBodyPart : NSObject
@property (nonatomic, assign) NSStringEncoding stringEncoding;
@property (nonatomic, strong) NSDictionary *headers;
@property (nonatomic, copy) NSString *boundary;
@property (nonatomic, strong) id body;
@property (nonatomic, assign) unsigned long long bodyContentLength;
@property (nonatomic, strong) NSInputStream *inputStream;

@property (nonatomic, assign) BOOL hasInitialBoundary;
@property (nonatomic, assign) BOOL hasFinalBoundary;

@property (readonly, nonatomic, assign, getter = hasBytesAvailable) BOOL bytesAvailable;
@property (readonly, nonatomic, assign) unsigned long long contentLength;

- (NSInteger)read:(uint8_t *)buffer
        maxLength:(NSUInteger)length;
@end
```

从源码可以看见
AFStreamingMultipartFormData（POST多part体）管理着AFMultipartBodyStream，
AFMultipartBodyStream（Body流）管理着AFHTTPBodyPart（分隔的每一步部分）。
从下面就能看出序列化所做的工作，就是请求体的包装。其中有文件的二进制等，采用了流的读取方式（这里就不深入了，有机会在研究吧）。

构建完成后就是如下结构
post请求的时候需要设置 Content-Type 和 Content-Length 
Content-Type 为 multipart/form-data; boundary=边界字符串
Content-Length 为数据的data长度

```
--Boundary+72D4CD655314C423
Content-Disposition: form-data; name="userId"
   // 这里是空一行，必不可少！！

--Boundary+72D4CD655314C423
Content-Disposition: form-data; name="shopId"
      // 这里是空一行，必不可少！！

--Boundary+72D4CD655314C423   // 分割符，以“--”开头，后面的字随便写，只要不写中文即可
Content-Disposition: form-data; name="uploadFile"; filename="001.png"  // 这里注明服务器接收图片的参数（类似于接收用户名的userName）及服务器上保存图片的文件名
Content-Type:image/png  // 图片类型为png
Content-Transfer-Encoding: binary  // 编码方式
  // 这里是空一行，必不可少！！
... contents of boris.png ...  // 图片数据部分
--Boundary+72D4CD655314C423--  // 分隔符后面以"--"结尾，表明结束
```

