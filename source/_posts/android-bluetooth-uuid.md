title: 对Android蓝牙UUID的理解
date: 2016-01-29 15:42:13
tags: [Android, Bluetooth, UUID]
categories: Programmer
description: UUID是根据一定算法，计算得到的一长串数字，这个数字的产生使用了多种元素，所以使得这串数字不会重复，每次生成都会产生不一样的序列，所以可以用来作为唯一标识。
---
# UUID
先来段百度百科上的解释：
> UUID含义是通用唯一识别码 (Universally Unique Identifier)，这 是一个软件建构的标准，也是被开源软件基金会 (Open Software Foundation, OSF) 的组织应用在分布式计算环境 (Distributed Computing Environment, DCE) 领域的一部分。它保证对在同一时空中的所有机器都是唯一的。通常平台会提供生成的API。按照开放软件基金会(OSF)制定的标准计算，用到了以太网卡地址、纳秒级时间、芯片ID码和许多可能的数字
> UUID 的目的，是让分布式系统中的所有元素，都能有唯一的辨识资讯，而不需要透过中央控制端来做辨识资讯的指定。如此一来，每个人都可以建立不与其它人冲突的 UUID。

总结起来就是，UUID是根据一定算法，计算得到的一长串数字，这个数字的产生使用了多种元素，所以使得这串数字不会重复，每次生成都会产生不一样的序列，所以可以用来作为唯一标识。

# 蓝牙RFCOMM数据(SPP 串口)通信

在来看看在蓝牙中为啥会用到UUID。
在蓝牙协议中，UUID被用来标识蓝牙设备所提供的服务，`并非是标识蓝牙设备本身哦`，一个蓝牙设备可以提供多种服务，比如`A2DP`（蓝牙音频传输）、`HEADFREE`（免提）、`PBAP`(电话本)、`SPP`(串口通信)等等，每种服务都对应一个UUID，其中在蓝牙协议栈里，这些默认提供的profile是都有对应的UUID的，也就是默认的UUID，比如`SPP`，`00001101-0000-1000-8000-00805F9B34FB`就是一个非常 `well-known`的UUID，基本上所有的蓝牙板不修改的话都是这个值，所以，如果是与一个蓝牙开发板进行串口通信，而蓝牙侧又不是自己可以控制的，就可以试试这个值。

当然，我们进行串口通信的开发，一般都会自己同时开发两侧，因为串口传递的数据就是数据流，没有格式之说，具体发送的数据的意义需要自己来定义，就是说自己定义规则，这就要求一端发送的数据，另一端可以理解。两者的通信基于socket进行实现，所以必须有一端做服务端，另一端做客户端。

再来说下，android里进行蓝牙串口通信的接口，以android端作为客户端为例，也就是对方蓝牙设备作为server端，等着android端来连接。
那么android端需要通过
```java
public BluetoothSocket createRfcommSocketToServiceRecord (UUID uuid)
```
这个接口来创建一个socket，并用这个socket对connect()，UUID参数类似用来指定socket对应的端口，而server端必须也有在这个UUID上创建好server端socket上监听才可以成功连接上，两者用的UUID必须一样才可以。

所以UUID的用处就在这里。

# 如何使用UUID

根据上面的描述，要使用蓝牙进行串口通信，前提条件就是两侧都是你可以定义的（如果不可以，那么你可以尝试`well-known SPP UUID`，但双方的数据你是没办法控制的），两侧都可以自定义，这样的场景才是使用蓝牙串口通信解决问题的合理场景啊，要不然通信了，互发的数据不认识，没啥用啊。。。。

所以，这种情况，最好是使用自己生成的UUID，在windows下，命令行里执行`uuidgen` 就可以得到一个，类似：
```
14c5449a-6267-4c7e-bd10-63dd79740e5d
```
这样的好处就是，只有我们自己的设备可以通信，别人的UUID来连接是连接不上的，我们也不会连接上别人的设备。

当然，你也可以选择`well-known`的UUID，这么干的也不少，这样有个缺点就是可能会有干扰，比如别人的设备也正好使用这个UUID起了个server端，那么我们的设备也使用这个UUID去连，就连上了。。。但。。。互发的数据还是不认识哦，没用。。。。


本身UUID就是用来标识唯一性的，还是自己生成吧。。。

附上，Android developer上的一段`hint`:
> About UUID
> A Universally Unique Identifier (UUID) is a standardized 128-bit format for a string ID used to uniquely identify information. The point of a UUID is that it's big enough that you can select any random and it won't clash. In this case, it's used to uniquely identify your application's Bluetooth service. To get a UUID to use with your application, you can use one of the many random UUID generators on the web, then initialize a UUID with fromString(String).
> Hint: If you are connecting to a Bluetooth serial board then try using the well-known SPP UUID 00001101-0000-1000-8000-00805F9B34FB. However if you are connecting to an Android peer then please generate your own unique UUID.

意思跟我上面提的差不多。。。
