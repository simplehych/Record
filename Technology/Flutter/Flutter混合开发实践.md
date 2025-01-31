
## 概述

1.  相对于单独开发Flutter应用，混合开发对于线上项目更具有实际意义，可以把风险控制到最低，也可以进行实战上线。所以介绍 **集成已有项目**

2. 混合开发涉及原生Native和Flutter进行通信传输，还有插件编写，所以介绍 **两端通信Flutter Platform Channel的使用**

## 说明

Flutter迭代变更频繁，部分API或已过期，请查阅官方文档

## 集成已有项目
  
[官方方案](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps) |  [闲鱼团队方案](https://yq.aliyun.com/articles/618599?spm=a2c4e.11153959.0.0.4f29616b9f6OWs) | [头条团队方案](https://mp.weixin.qq.com/s/wdbVVzZJFseX2GmEbuAdfA)

采用官方的实现方式，对个人和小团队相对简单易用，并没有实践闲鱼和头条等大项目的方式。个中利弊上述推荐均有提到。

开始

1. 在已有项目根目录运行命令

    `flutter create -t module my_flutter`

    *不一定非在项目下创建，兼顾ios，位置路径寻址对即可*

2. 在项目的`settings.gradle` 文件添加如下代码

    ```
    // MyApp/settings.gradle
    include ':app'                                     // assumed existing content
    setBinding(new Binding([gradle: this]))                                 // new
    evaluate(new File(                                                      // new      settingsDir.parentFile,                                               // new
      'my_flutter/.android/include_flutter.groovy'                          // new
    ))                                                                      // new
    ```

    *Binding飘红不予理会，强迫症的不用导包*
    *'my_flutter/.android/include_flutter.groovy'为my_flutter文件路径，报错找不到可写项目全路径'ChannelFlutterAndroid/my_flutter/.android/include_flutter.groovy'*

3. 在运行模块app的build.gradle添加依赖

  ```
  // MyApp/app/build.gradle
  dependencies {
      implementation project(':flutter')
  }
  ```
  
1. 在原生Android端创建并添加flutterView
  ```
   val flutterView = Flutter.createView(this, lifecycle, "route1")
   val layoutParams = FrameLayout.LayoutParams(-1, -1)
   addContentView(flutterView, layoutParams)
  ```

2. 在Flutter端入口操作

    ```
    void main() => runApp(_widgetForRoute(window.defaultRouteName));

    Widget _widgetForRoute(String route) {
      switch (route) {
        case 'route1':
          return SomeWidget(...);
        default:
          return Center(
            child: Text('Unknown route: $route', textDirection: TextDirection.ltr),
          );
      }
    }
    ```
    至此，集成完毕

## 两端通信Flutter Platform Channel的使用

本文主要是介绍使用方式，[闲鱼团队详尽的介绍了实现原理](https://yq.aliyun.com/articles/630105?spm=a2c4e.11153959.0.0.59eb616bHPgOl4)

Flutter提供 **`MethodChannel`**、**`EventChannel`**、**`BasicMessageChannel`** 三种方式

1. 类似注册监听，发送的模式 原则
2. 使用顺序：先注册，后发送，否则接收不到。尤其使用 `MethodChannel`、`EventChannel` 不符合该原则会抛出异常，`BasicMessageChannel`方式只是收不到消息

### 方式一 MethodChannel

#### 应用场景：Flutter端 调用 Native端

**Flutter端代码：**
```
 var result = await MethodChannel("com.simple.channelflutterandroid/method", 
                      StandardMethodCodec())
                      .invokeMethod("toast", {"msg": msg})
```
**Android端代码：**
 ```
   MethodChannel(flutterView, "com.simple.channelflutterandroid/method",
               StandardMethodCodec.INSTANCE)
              .setMethodCallHandler { methodCall, result ->
         when (methodCall.method) {
              "toast" -> {
                 //调用传来的参数"msg"对应的值
                 val msg = methodCall.argument<String>("msg")
                 //调用本地Toast的方法
                 Toast.makeText(cxt, msg, Toast.LENGTH_SHORT).show()
                 //回调给客户端
                  result.success("native android toast success")
              }
             "other" -> {
                // ...
              }
              else -> {
                // ...
              }
          }
    }
 ```

* 注意点1：两端需要对应一致的点
  方法名称：com.simple.channelflutterandroid/method，可以不采取包名，对应一致即可，不同的方式最好不要重复
  传参key："msg"

* 注意点2：以下为Flutter为例，Android同理
 `StandardMethodCodec()` 非必传，默认是 `StandardMethodCodec()`，是一种消息编解码器

  对于 `MethodChannel` 和 `EventChannel` 两种方式，有两种编解码器，均实现自`MethodCodec` ，分别为 `StandardMethodCodec` 和 `JsonMethodCodec`。

  对于 `BasicMessageChannel` 方式，有四种编解码器，均实现自`MessageCodec` ，分别为 `StandardMessageCodec`、`JSONMessageCodec`、`StringCodec`和`BinaryCodec`。

* 注意点3：setMethodCallHandler(),针对三种方式一一对应，属于监听
    `MethodChannel` - `setMethodCallHandler()`
    `EventChannel` - `setStreamHandler()`
    `BasicMessageChannel` - `setMessageHandler()`
    其中setStreamHandler()方式稍微特殊，具体查看上面提的闲鱼文章

* 注意点4：`FlutterView` 实现 `BinaryMessenger` 接口

以上，以下两种方法不再赘述


### 方式二 EventChannel

#### 应用场景：Native端 调用 Flutter端，方式稍微特殊

**Flutter端代码：属于接收方**

 ```
    EventChannel("com.simple.channelflutterandroid/event")
            .receiveBroadcastStream()
            .listen((event){
                    print("It is Flutter -  receiveBroadcastStream $event");
    })
 ```

*注意 `receiveBroadcastStream`*

 **Android端代码：属于发送方**

 ```
    EventChannel(flutterView, "com.simple.channelflutterandroid/event")
          .setStreamHandler(object : EventChannel.StreamHandler {
                override fun onListen(o: Any?, eventSink: EventChannel.EventSink) {
                      eventSink.success("我是发送Native的消息")
                }

                override fun onCancel(o: Any?) {
                        // ...
                }
    })
 ```

### 方式三 BasicMessageChannel

**应用场景：互相调用**

两种使用方式，创建方式和格式不一样

#### 第一种

Flutter端

```
_basicMessageChannel = BasicMessageChannel("com.simple.channelflutterandroid/basic", 
                                            StringCodec())

// 发送消息
_basicMessageChannel.send("我是Flutter发送的消息");

// 接收消息
_basicMessageChannel.setMessageHandler((str){
    print("It is Flutter -  receive str");
});
```

Android端
```
val basicMessageChannel = BasicMessageChannel<String>(flutterView, "com.simple.channelflutterandroid/basic", 
                                                      StringCodec.INSTANCE)

// 发送消息
basicMessageChannel.send("我是Native发送的消息")

// 接收消息
basicMessageChannel.setMessageHandler { o, reply ->
    println("It is Native - setMessageHandler $o")
    reply.reply("It is reply from native")
}
```

*其中StringCodec()，四种方式可选*

#### 第二种

Flutter端
 ```
    static const BASIC_BINARY_NAME = "com.simple.channelflutterandroid/basic/binary";

    // 发送消息
    val res = await BinaryMessages.send(BASIC_BINARY_NAME, ByteData(256))

    // 接收消息
    BinaryMessages.setMessageHandler(BASIC_BINARY_NAME, (byteData){
        println("It is Flutter - setMessageHandler $byteData")
    });
 ```

 Android端
 ```
    const val BASIC_BINARY_NAME = "com.simple.channelflutterandroid/basic/binary"

    // 发送消息
    flutterView.send(BASIC_BINARY_NAME,ByteBuffer.allocate(256));

    // 接收消息
    flutterView.setMessageHandler(BASIC_BINARY_NAME) { byteBuffer, binaryReply ->
        println("It is Native - setMessageHandler $byteBuffer")
        binaryReply.reply(ByteBuffer.allocate(256))
    }
 ```
*此组合可以进行图片的传递*

>**help** 在操作中使用此方式传输数据总报错，所以只传了默认ByteBuffer。

**如有错误之处，万望各位指出！谢谢**

### 其他

1. Native端接收方收到消息后都能进行回传信息，在Flutter进行异步接收
    MethodChannel：result.success("我是回传信息")；
    EventChannel：eventSink.success("我是回传信息")；
    BasicMessageChannel：binaryReply.reply(-)；- Flutter端：ByteData；Android端：ByteBuffer


### Q&A
以上均为理想情况，过程中还是存在许多问题，以下提供参考

**Q1：**
使用命令行flutter create创建的project或者module，文件android/ios为隐藏打点开头的.android/.ios文件
**A1：**
所以有的时候会出现问题，寻找不到文件的情况

**Q2：** 
flutter create module 和project的区别
**A2：**
目前没发现

**Q3：**
couldn't locate lint-gradle-api-26.1.2.jar for flutter project
**A3：** 
https://stackoverflow.com/questions/52945041/couldnt-locate-lint-gradle-api-26-1-2-jar-for-flutter-project

**Q4：** 
Could not resolve all files for configuration ':app:androidApis'.
Failed to transform file 'android.jar' to match attributes {artifactType=android-mockable-jar, returnDefaultValues=false} using transform MockableJarTransform
**A4：**
https://github.com/anggrayudi/android-hidden-api/issues/46


**Q5：**
Flutter Error: Navigator operation requested with a context that does not include a Navigator 
**A5：** 
https://github.com/flutter/flutter/issues/15919