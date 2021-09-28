---
title: 基于protobuf & MethodChannel实现Flutter混合开发通信
date: 2020-08-04 11:06:01
tags:
---

Flutter和Native的通信是混合开发的一个关键问题，Google官方给出了MethodChannel这个方案，使用起来很简单：设置好通道名和方法名还有钩子函数，两边就可以互相调用方法。MethodChannel支持下图的基本数据类型：

![image-20200803172422873](https://tva1.sinaimg.cn/large/007S8ZIlly1ghdqzmwx7fj31el0pqtif.jpg)

MethodChannel对于简单的消息传递已经够用了，但如果想要传递一个复杂的消息需要将消息打包成一个json也就是Map进行传递，在打包和解包的过程中包含了很多类型检查及转换的工作，并且需要在native侧和flutter侧分别实现一次，人工实现还容易出错。

为了解决上述问题，我们开发了一款基于MethodChannel和Google的[protobuf](https://developers.google.com/protocol-buffers)的插件。

protobuf是一款跨平台、跨语言的序列化数据结构，编写一个.proto文件声明数据的结构，然后通过插件就可以将它转换成支持的目标语言代码。

使用protobuf作为flutter和native通信的介质主要有以下几个优点：

* 三端（iOS、flutter、Android）可以共用一个.proto文件
* 强类型支持，可以将类型检查封装在解包过程中
* protobuf编解码效率很高

------

实现上述功能需要以下几个步骤：

## PBMethodChannelCodc

这个类是个单例，是PBMethodChannel使用的编解码器，它负责将消息打包、解包并进行最基本的类型校验，即将消息从PB对象转换成二进制和将消息从二进制数据解包成一个PB对象。

通信传输一个基本的pb消息，里面包含两个字端：`method`、`param`

其中`method`是字符串类型，为方法名；`param`是Any类型，可以存放其他类型的pb消息。

当接收到消息的时候解码器会将二进制数据解码成上述基本pb消息并进一步转换成`FlutterMethodCall`，当调用一个方法的时候将`FlutterMethodCall`打包上述pb消息。

## **PBMethodChannel**

封装这样一个类，它有类似MethodChannel的接口，数据载体是protobuf，下面是iOS头文件示例：

```objc
// 调用Flutter回调闭包
typedef void  (^PBMethodChannelResult)(GPBAny *arg);

// 接收Flutter调用闭包
typedef GPBMessage * _Nonnull(^PBMethodChannelHandler)(GPBAny *arg);

@interface PBMethodChannel : NSObject

+ (instancetype)methodChannelWithName:(NSString *)name
binaryMessenger:(NSObject<FlutterBinaryMessenger>*)messenger;

- (void)invokeMethod:(NSString *)method;

- (void)invokeMethod:(NSString *)method arguments:(GPBMessage *)arguments;
- (void)invokeMethod:(NSString *)method arguments:(GPBMessage *_Nullable)arguments resultBlock:(PBMethodChannelResult _Nullable)resultBlock;

- (void)setMethodCallHandler:(NSString *)method handler:(PBMethodChannelHandler)handler;
- (void)removeHandler:(NSString *)method;

@end
```

## PBMethodChannelApi

为了实现类型检查，还需要封装一个类来方便使用PBMethodChannel，希望达到的效果是：声明一个方法的输入类型和输出类型然后可以直接调用，并且响应的结果自动经过类型转换和检查。

------

经过以上的努力，我们就可以在混合开发中愉快地使用MethodChannel了：

* 确定通道名、方法名
* 写一个pb消息并运行插件生成三端代码
* 分别在native和flutter之中实现调用/响应逻辑