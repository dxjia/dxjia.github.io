title: iOS APP Life Cycle
tags:
  - iOS
  - APP
categories:
  - Programmer
date: 2016-05-23 15:56:44
---
本文是学习过程中的笔记，帮助理解 iOS APP 特点；跟 Android 类似，每个 iOS APP 也都有自己的生命周期，前台、后台、挂起等，理解这些状态转换过程，并处理相应事件，将有助于提供良好的用户体验。
<!--more-->
## 入口
跟 OS X 桌面程序一样，是 `main` 函数，不同的是，XCode 帮你直接实现一个默认的main 函数，而且任何情况下都不需要修改这个函数；
```
#import <UIKit/UIKit.h>
#import "AppDelegate.h"
 
int main(int argc, char * argv[])
{
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
可以看到 是直接将控制权交给 framework 的 UIApplicationMain 类；

## APP 结构
APP 启动过程中， `UIApplicationMain` 类会为 APP 创建一些重要的对象，并让整个 APP 运行起来，其中一个最重要的就是 `UIApplication` 对象，在这个对象里，不断循环并向 APP 派发各种系统事件，像不像 Android 的 `MainThread`， 太像啦。。。下面这个是官方给的图，`MVC` 结构，iOS 开发里这种设计模式贯穿始终。
![](http://7xqitw.com1.z0.glb.clouddn.com/blog/res/core_objects_2x.webp)

## The Main Event Loop
在 `UIApplication` 对象里的主循环，构成了APP 的主线程，系统产生的一些用户交互事件，通过在 APP 初始化时由 `UIKit` 生成的一个端口号，分发给 APP， `UIApplication` 对象里的主循环 是 APP 里第一个接收到这些事件的地方；**当然，并不是所有的交互事件都会走 main loop， 可以知道主循环的概念即可**
![](http://7xqitw.com1.z0.glb.clouddn.com/blog/res/event_draw_cycle_a_2x.webp)

## APP`S 状态
iOS 的 APP 有 **`5`** 种状态，任何时刻都只会是这 5 种中的一种！如下：

| 状态      |     说明 |
| :-------- | :--------|
| Not running    |   未运行状态，APP 还没有被启动过，或者 已经被系统从挂起状态清理掉了； |
| Inactive    |   未激活状态，APP 正在运行并处于前台，但是无法接收任何事件，通常发生在状态转换的时期，而且在这个状态停留的时间一般都非常短暂，过渡的； |
| Active    |   激活状态， 正在运行，并处于前台，与用户直接交互，可以接收所有事件；|
| Background    |   后台运行状态，程序处于后台并可以执行代码(用户切换到别的程序)，一个 APP 进入后台后，iOS 只会留给程序很短暂的一点时间进行数据清理或保存工作，之后就会将程序挂起；所以 iOS 上在这个时期所做的事情非常有限； |
| Suspended    |   挂起状态，APP还活着，在后台，继续占有内存，但不会执行任何代码，iOS 会保持这个状态一段时间，以便快速响应用户重新回到 APP，但如果系统内存吃紧，就会优先杀掉这个状态的进程； 注意，系统 从 Background 转到 Suspended 状态不会有任何通知|

官方给的图，**`从这个图上我们可以看出，Active 和 Background 状态之间的转换都会是经过一个短暂的 Inactive 状态的`**
![](http://7xqitw.com1.z0.glb.clouddn.com/blog/res/high_level_flow_2x.webp)

## 状态切换回调
APP 状态转换通过在 app delegate 类对象里进行回调一些固定的方法达到通知 APP 的效果，跟 Android 的 `onCreate, onStart, onResume....` 太像！

| 方法      |     说明 |
| :-------- | :--------|
|application:willFinishLaunchingWithOptions|程序正在被启动，但还没有到前台，可以在这里做第一次的初始化动作，如检查 options 等；|
|application:didFinishLaunchingWithOptions|程序已经完成系统初始化工作，但依然还没有到前台，可以在这里做最终的初始化动作；这两个回调都是 APP 被第一次 launch 时才会回调的，其他从后台挂起到前台是不会触发这两个的，为啥有俩？|
|applicationDidBecomeActive| 已经处于 Active 状态，这里就可以开始做事啦； |
|applicationWillResignActive| 程序马上就要进入后台啦|
|applicationDidEnterBackground| APP 已经进入后台，并可能随时会被挂起，做状态保存吧； 当然也可以在这里请求延长后台任务的执行，有 API； |
|applicationWillEnterForeground| 通知你 程序马上就要进入前台啦，但此时还不能响应事件，还不是 Active，正在 active 后 会回调 applicationDidBecomeActive 方法，所以先在这里恢复状态吧； |
|applicationWillTerminate| 程序将被杀掉（该回调只会在程序处于后台且非挂起的状态下被杀掉），系统内存不足时，系统会杀掉后台的程序，在这里做善后工作吧；而如果系统是杀掉的挂起的程序，则不会有任何通知；用户主动杀掉进程的机制跟此类似 |
