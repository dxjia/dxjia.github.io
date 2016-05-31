title: iOS 后台任务设计指导
tags:
  - iOS
  - Apple
  - Guideline
categories:
  - Programmer
date: 2016-05-26 16:50:19
---

在 [上一篇](http://dxjia.cn/2016/05/23/ios-app-life-cycle/) 文章里介绍了 iOS APP 的状态切换，其中一个状态是 `Background` ， 也就是后台，在这个状态，程序只被允许执行非常有限的一点点时间，然后就会随时被挂起，不再执行任何代码，但显然是有应用场景`需要`不断地在后台执行一些任务，比如 音乐播放 APP， 健康记步软件等。本文就来介绍 iOS 所提供的实现这些场景的技术。
<!--more-->

## 后台任务分类
首先 Apple 官方为我们界定了 **`3`** 类后台执行任务的场景：
- **`Background Tasks`**：APP 在前台时启动某项任务，然后在未结束之前突然 切换到了后台，那么 APP 可以在切换回调里使用某些 API 来继续向系统请求一些时间来继续完成这个任务；完成之后通知系统，之后系统会将 APP 挂起；
- **`Downloading`**：在后台启动从网络下载文件的任务 -- 对于文件下载，iOS 有专门的机制；
- **`Specific Backgournd Tasks`**：应用需要在后台一直执行代码；

这三种类型的后台任务，在实现时各有不同，下面来一一介绍。

## Background Tasks
Apple 文档建议，如果要启动一个后台任务（异步任务），可以使用 API `beginBackgroundTaskWithExpirationHandler`来指定，即使启动任务的时候，程序是处在前台的，也没有关系，当位于前台时，该方法请求得到的时间是`DBL_MAX`，也就是 double 数据类型最大值，你可以认为是无限大，当任务执行过程中 APP 被切换到后台时，任务还没有完成，这个时间又会自动调整为一个时间片段（具体多少我没找到文档说明，都是说可以通过`backgroundTimeRemaining` 属性得到）。需要注意的是， 这个方法是成对使用的，对于一个固定 task ，每次调用 `beginBackgroundTaskWithExpirationHandler`，都会产生一个 token 值(`UIBackgroundTaskIdentifier` 实际是个整型)，必须在任务执行结束时，调用 `endBackgroundTask` 并传递这个 token，来结束后台任务。另外，作为最佳实践，都应该传递一个 超时 handler，以防申请到的时间片段内，还是没能完成任务的话，做最后的清理和标注工作！如果不传的话，那么结果就是 iOS 直接 kill 掉你的APP，闪退咯，因为它觉得我们骗了它嘛，哈哈。。。
下面是一段在进入后台时启动异步任务的例子；
```objectivec
// 在某处定义一个 token 变量
UIBackgroundTaskIdentifier _bgTaskToken;

// 进入后台 委派方法回调 
- (void)applicationDidEnterBackground:(UIApplication *)application
{
    _bgTaskToken = [application beginBackgroundTaskWithName:@"MyTask" expirationHandler:^{
        // 时间到了，任务还没完成，只能清理
        ...
        // 取消后台任务
        [application endBackgroundTask:_bgTaskToken];
        _bgTaskToken = UIBackgroundTaskInvalid;
    }];
 
    // 异步启动任务，这样不会阻塞 本委派方法回调
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 巴拉巴拉，做自己的任务
        ...
        
        // 任务在时间限制内结束啦，取消后台任务 
        [application endBackgroundTask:_bgTaskToken];
        _bgTaskToken = UIBackgroundTaskInvalid;
    });
}
```

## Background Downloading
这类后台任务，必须使用 iOS 指定的机制才可以，那就是 `NSURLSession `。使用 NSURLSession  建立的下载任务，会被系统直接在另外一个独立的系统进程里进行管理，不会因 APP 进入后台或挂起等而受到影响，iOS 会统一管理所有的下载任务。并且，即使你的 APP 已经挂掉啦，下载任务还是会继续，等到下载完成啦，系统会唤起你的 APP 进程，并通知你，但如果是用户主动杀掉的你的进程，那么系统会自动取消下载任务。
具体使用方法：
- 1, 使用 `NSURLSessionConfiguration` 类的 `backgroundSessionConfigurationWithIdentifier` 方法创建一个 NSURLSessionConfiguration 对象，参数为一个字符串，作为一个 token ，完成时会用到，不能为空或 nil；
- 2, 设置上一步创建出的对象的 `sessionSendsLaunchEvents` 属性为 `YES`；
- 3, 如果开启下载任务时，是位于前台的，将 ` discretionary` 属性也设置为 `YES`；
- 4, 设置你需要的其他属性值；
- 5，使用配置好的 `NSURLSessionConfiguration` 对象，作为参数，创建 `NSURLSession ` 实例对象；
- 6，使用 `NSURLSession` 开始下载task，这个这里不细讲啦，需要看 [《URL Session Programming Guide》](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html) ;

如果在下载完成之前，你的APP已经挂起或者死掉啦，那么当系统完成下载之后，系统会唤醒你的 APP，并回调 你的 app 委托方法 `application:handleEventsForBackgroundURLSession:completionHandler:`，在这其中，参数会传进来一个 `token`，这个就是你第一步里 传入的 字符串，使用这个 字符串，再重新创建一个 `NSURLSessionConfiguration`，并进行与开始任务之前一样的配置，那么就可以使用这些对象来获取已经完成的任务的详细情况了。

## Background Long-Running Tasks
在 iOS 里只有特定的一些应用类型才会被允许可以在后台一直运行，APP 必须显式的声明一些特定权限，才可以在后台进行长时间运行而不被挂起。
一些应用类型有 **`6`** 种：
- 需要在后台播放音频 -- 如 Music Player；
- 需要在后台录音；
- 在后台时也需要不断通知用户位置变动的，比如导航；
- 支持 VoIP 电话的 -- 如 skype 网络电话；
- 需要在后台有规律的下载和处理网络内容的；
- 在后台有规律的从其他外设(第三方配件)获取并更新数据的；

要实现这些类型服务的 APP，需要进行专门的声明，这样系统才会采取相应的操作。

先来看看怎么声明。
### 声明后台服务类型
通过 XCode 的 project setting 里就可以配置类型，选择之后会自动 在你 工程的 Info.plist 文件里 增加 `UIBackgroundModes ` 键值对；一个 APP 可以同时声明多种支持的后台长期任务类型，在 XCode 里勾选上即可；
下表给出了所有 在 XCode 可选的 类型 及 具体含义；

| Xcode background mode|  UIBackgroundModes 值|   描述|
| :-------- | :--------| :------ |
| Audio and AirPlay|   audio|  应用可以在后台播放或录制音频，包括 Apple 自家的 AirPlay 流媒体音视频；对于录制，需要在APP 第一次运行时，用户授予权限才可进行。|
| Location updates    |   location |  APP 不断更新 GPS 位置信息，并通知给用户，即使 APP 处于后台  |
| Voice over IP    |   voip |  APP 提供通过网络连接来打电话的功能  |
| Newsstand downloads    |   newsstand-content |  杂志应用，可以在后台下载杂志并处理  |
| External accessory communication    |   external-accessory |  一些外设控制 APP， 比如一些控制 第三方 MFI 配件的应用，声明这种 类型，可以让APP 在后台不断的与 外设进行沟通 |
| Uses Bluetooth LE accessories    |   bluetooth-central |  iPhone 作为蓝牙中心设备使用，也就是做为 server；需要在后台不断更新蓝牙状态的 |
| Acts as a Bluetooth LE accessory    |   bluetooth-peripheral |  iPhone 作为蓝牙外围设备使用，也就是做 client，需要在后台不断的访问其他蓝牙设备获取数据的  |
| Background fetch    |   fetch |  APP 需要在后台不断地 频繁有规律的从网络获取数据  |
| Remote notifications    |   remote-notification |  APP 先在后台关注某个 push 推送，但这个 push 推送到达的时候，及时在后台开始对应的下载任务，以尽可能减少用户直接点开 通知 后 查看内容的等待时间  |

### Playing and Recording Background Audio
一些典型的应用例子：
- 音乐播放软件
- 录音APP
- 支持 AirPlay 音视频播放的APP
- 网络通话软件

当你在 Info.plist 里声明了 `UIBackgroundModes` 为 `audio` 的时候，在后台进行 audio 的相关操作时，系统 audio API 会自动阻止系统将你的 APP 进程挂起，所以不需要 APP 自己再进行其他额外的处理，只需要处理自己的软件逻辑即可。

【Note】：手机上是有可能会有多个 APP 同时拥有后台 audio 操作权限的，这时候系统会根据 每个 APP 开始操作音频时的 audio session 配置来决定如何进行操作，而且你应该非常小心的处理一些中断事件，如来电，其他系统提示音等，这些都有相关的 API 和机制，可以参考 [《Audio Session Programming Guide》](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/Introduction/Introduction.html) 

### Tracking the User’s Location
有三种方式来实现 位置的访问：
- The significant-change location service（这也是官方推荐的方式）
- Foreground-only location services
- Background location services

前两种都不需要在 Info.plist 里声明 `UIBackgroundModes` ，只有最后一种需要。

`The significant-change location service` ，字面理解，就是只有位置有变化时才会发出通知，有人说这个时机是依据基站，切换了基站时，就会发出一次通知，所以频率会受基站的密度影响，所以市区更新频率会比郊区高。但`好处` 是这个服务不管你的 APP 是在前台还是后台，不管是否已经被挂起，或已经死掉了，他都会唤醒你的进程进行相应处理，所以应该是最省电的。

后两种都是标准的定位服务，只不过一个只能工作在前台，而一个可以在后台工作；
【Note】：官方对于使用后台定位服务的 APP 审核是非常严格的，所以使用时一定要小心，并提供足够的说明和解释。

至于如何实现一个定位 APP ，请看 [《Location and Maps Programming Guide 》](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/LocationAwarenessPG/Introduction/Introduction.html)

### Implementing a VoIP App
网络通话软件，skype 就是其中一个。这样的软件使用 internet 连接来进行语音通话，为了提供健全的电话功能，这类软件必须一直保持一个长期的网络连接，以便监听到来电。实现类似的功能，APP 自己并非一直在后台不被挂起，而是交由系统监听 网络连接，有数据进来时，系统会唤醒 APP，并将 socket 转交给 APP 进行处理；
大致步骤：
- 在 Info.plist 里进行`UIBackgroundModes`配置；
- 配置一个 socket 连接用于 VoIP；
- 在进入后台时，调用 `setKeepAliveTimeout:handler: `方法传递一个回调，用来处理事件；
- 配置要使用到的 audio session；

【Note】：貌似对于 VoIP 的实现， iOS 8 有变化，改为使用 remote notification 的方式来做啦，谁说 iOS 没有碎片化的啊，有！具体实现请参考 [Tips for Developing a VoIP App](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/StrategiesforImplementingYourApp/StrategiesforImplementingYourApp.html#//apple_ref/doc/uid/TP40007072-CH5-SW13)


### Downloading Newsstand Content in the Background
杂志应用，居然还有专门的处理。但我看介绍，跟前面讲解的 后台下载文件没啥区别啊！！另外好像也是用 通知推送 触发啊。[About Newsstand Kit Framework](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/NewsstandKit_Framework/index.html#//apple_ref/doc/uid/TP40010838)


### Communicating with an External Accessory
外设设备有很多，比如一些心率监控器，会在必要的时候向手机推送数据。声明了`UIBackgroundModes` 为 `external-accessory` 后，系统就不会主动关闭 APP 与 外设之间的连接，而是替 APP 监视这个连接，但有数据过来时，会唤醒 APP 进行处理，每次唤醒 APP 只有 `10 S` 种时间进行数据处理，所以应当越快越好，万不得已，如果10S不够，需要使用 `beginBackgroundTaskWithExpirationHandler:` 方法再申请一段时间进行处理；
【Note】：Apple 要求此类应用 需要提供一个 开启 和 关闭 连接的界面供用户使用；

### Communicating with a Bluetooth Accessory
类似上一节的 配件，如果心率监控器跟 手机之间使用的连接方式是蓝牙，那么就一模一样啦，连 唤醒的时间限制都一样，都是 `10 S`！！！略啦。。。

### Fetching Small Amounts of Content Opportunistically
有人依靠这种手段来实现后台永存，但现在不好使啦，除非你是真的每次都在下载东西，而且每次时间都很短。用户的流量啊。因为声明了这个 mode 之后，并不保证 系统一定会给你分配时间来执行后台任务，因为它自己有一套逻辑，如果你经常性唤醒，但却每次都耗时很久，又没有做从网络下载东西的操作，那么以后你被分配给唤醒的几率就会越来越小。另外还有审核！！！！

正常情况下，声明了这个类型之后，系统在你的 APP 进入后台后，会间隔性的给机会将你的 APP 唤醒，并回调你的 委托方法 `application:performFetchWithCompletionHandler:`，你需要在这个回调里检查是否有新内容可用，如果有，就开启后台下载，推荐使用 `NSURLSession` 来建立，下载完成后，你必须调用这个方法出入 的 `completionHandler` 并传入一个 整型值  来表示 你的处理是否正常，UI是否已经更新，让系统来决定更新 snapshot等；

### Using Push Notifications to Initiate a Download
这个方式，是你的应用中包含通知功能时，你在服务端推送的通知内容里加入 键值对 `content-available` = `1` ，那么 手机收到这个通知后，会自动启动 APP 到后台，或 唤醒（依旧保持后台执行），并回调 委托方法 ` application:didReceiveRemoteNotification:fetchCompletionHandler:` ，在这个方法里进行内容下载。

【Note】：需要服务端推送配合

## 哪些情况系统会唤醒挂起进程
当一些特定事件发生时，系统会唤醒已经被挂起的进程，转换到后台运行状态，这些事件针对不同类型的APP 有所不同：
### location apps
- 系统产生了符合 APP 配置的定位要求的位置更新；
- 设备进入或离开了一个网络注册的区域，你可以理解为基站；
### audio apps
- audio framework 需要 app 处理数据的时候--任何 播放、录制；
### Bluetooth apps
- 当手机扮演中心设备时，收到了其他蓝牙设备发来的数据；
- 当手机扮演外围设备时，收到了蓝牙服务端发来的数据；
### background download apps
- 本应用的一个包含 `content-available` = `1` 的推送通知到达了手机；
- background fetch 类型，系统给予了 APP 唤醒的机会；
- 使用 `NSURLSession` 进行后台下载的APP，在下载过程完成或出现问题时，系统会主动唤醒对应 APP；
- 杂志应用，下载完成时唤醒 APP；

【Note】：绝大多数情况下，系统不会重启被用户手动强制关闭的 APP，但在 iOS 8 之后， location apps 是个例外。其他的所有被用户手动强制关闭的APP 都不会被系统主动唤起，直到 用户再次 主动启动这个 APP，或者手机重启并在用户输入了解锁密码之后才会恢复机制。

## 做一个尽责的后台APP
Apple 教育我们，如果你要实现一个后台 APP，应该做一个有责任的APP，不要乱搞，哈哈。
- 不要在后台调用任何 OpenGL ES 接口，在进入后台之前也要保证这些调用都已结束，否则你的 APP 将直接被 kill；
- 取消所有 `Bonjour` 相关的操作，还不清楚这个是啥东西，不过 Apple 说即使你不取消，它在把你挂起之前也会都给你取消；
- 如果有网络操作，做好容错处理；
- 保存 APP 状态，进入后台前持久化一些数据，以便恢复；
- 尽可能多的释放内存，尤其是强引用；
- 停止使用共享的系统资源，比如 电话本，日历等，进入后台前，release他们；
- 不要在后台进行 UI 的更新操作；
- 做好对外设配件的 连接 和断开 事件的响应；这个是 外设编程的机制啦，需要 参考 [External Accessory Programming Topics](https://developer.apple.com/library/ios/featuredarticles/ExternalAccessoryPT/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009502) ；
- 关闭弹出窗口和弹出菜单等；
- 移除窗口上的一些敏感信息；
- 在后台的执行尽可能小的任务；

最后， Apple 建议能不后台就不后台，那当然。。。

## Reference
[1] [App Programming Guide for iOS - Background Execution](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html)

