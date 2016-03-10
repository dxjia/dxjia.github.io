title: Android 5.0 Default SMS App以及运营商授权SMS App
tags:
  - Android
  - SMS
categories:
  - Programmer
date: 2016-03-02 14:37:35
---
**Android**中短信的接收是这样的一个过程：：
 
- 底层先将短信报给FW，FW处理过后，会将短信通过intent广播的形式广播出来，而注册了接收短信广播的APP们，就能收到并处理短信。

<!--more-->

# Default SMS App

而android在**`4.2`**开始，对操作SMS的app进行了限制，增加了`default SMS APP`的概念，只有default APP才可以操作短信，而且`default SMS APP`可以由用户来指定。

　　先来看看整个系统初始化时，是如何来初始化`default SMS APP`的：

　　依然是在FW的phone框架初始化里，`PhoneFactory. makeDefaultPhone`:
```
	// Ensure that we have a default SMS app. Requesting the app with
	// updateIfNeeded set to true is enough to configure a default SMS app.
	ComponentName componentName = SmsApplication.getDefaultSmsApplication(context, true /* updateIfNeeded */);
	String packageName = "NONE";
	if (componentName != null) {
		packageName = componentName.getPackageName();
	}
	Rlog.i(LOG_TAG, "defaultSmsApplication: " + packageName);

	// Set up monitor to watch for changes to SMS packages
	SmsApplication.initSmsPackageMonitor(context);
```
首先调用一次`SmsApplication.getDefaultSmsApplication`方法，并且指定第二个参数`updateIfNeeded`为`true`，就是如果没有设置过就自动指定一个。

```
    /**
     * Gets the default SMS application
     * @param context context from the calling app
     * @param updateIfNeeded update the default app if there is no valid default app configured.
     * @return component name of the app and class to deliver SMS messages to
     */
    public static ComponentName getDefaultSmsApplication(Context context, boolean updateIfNeeded) {
        int userId = getIncomingUserId(context);
        final long token = Binder.clearCallingIdentity();
        try {
            ComponentName component = null;
            SmsApplicationData smsApplicationData = getApplication(context, updateIfNeeded,
                    userId);
            if (smsApplicationData != null) {
                component = new ComponentName(smsApplicationData.mPackageName,
                        smsApplicationData.mSmsReceiverClass);
            }
            return component;
        } finally {
            Binder.restoreCallingIdentity(token);
        }
    }
```
关于getApplication函数我在 [《Android 5.0 phone初始化分析》](http://dxjia.cn/2016/01/29/phone-analysis/) 一文中有讲到，其指定`default SMS App`的规则如下：    
- 首先尝试获取用户指定的默认app，对应的系统setting key为：sms_default_application；
- 其次看是否有Google的官方 默认sms app；
- 如果以上两个都没有，那么就从PM中获取所有注册有完整sms有关的broadcast receiver的app，从中找一个优先级最高的，并将其设定为default app。

`default SMS App`的值保存在setting db中， `Settings.Secure.SMS_DEFAULT_APPLICATION`，当然也提供了set方法来让用户可以手动设置他想使用的`default SMS App`
```
    /**
     * Sets the specified package as the default SMS/MMS application. The caller of this method
     * needs to have permission to set AppOps and write to secure settings.
     */
    public static void setDefaultApplication(String packageName, Context context) {
        TelephonyManager tm = (TelephonyManager)context.getSystemService(Context.TELEPHONY_SERVICE);
        if (!tm.isSmsCapable()) {
            // No phone, no SMS
            return;
        }

        final int userId = getIncomingUserId(context);
        final long token = Binder.clearCallingIdentity();
        try {
            setDefaultApplicationInternal(packageName, context, userId);
        } finally {
            Binder.restoreCallingIdentity(token);
        }
    }
```
另外，在`PhoneFactory`初始化里我们还看到在调用一次`getDefaultSmsApplication`后，还调用了另外一个方法：
> **SmsApplication.initSmsPackageMonitor(context);**

这个方法会监听应用程序的安装与卸载，并在有应用被安装或者移除的时候，能够及时自动更新default sms app，已保证`default SMS App`是随时都有设定的。

# 运营商授权SMS App
　　后来的版本，android又增加了`运营商授权SMS APP`的实现，原则是如果所有的SMS APP里，如果有一个是运营商授权指定的短信处理 APP，那么它就会有第一优先级，不管default app设定的是谁，都会只使用这个授权app来收发和管理显示短信。
　　那么这个运营商授权APP是在哪里指定的呢？**`答案是：是固化在icc卡里的，也就是运营商给你的手机卡（no-uim和no-sim的手机目前是处理不了的），卡在出厂的时候，会在卡里的某个固定单元文件写上授权APP的package name以及其签名hash校验值，在卡初始化完成后读取这些值解析后保存，如果手机里有这个package name的app，并且签名hash也一致，那么就说明该App是运营商授权sms app。`**
　　完成这些信息初始化的类为 `UiccCarrierPrivilegeRules`，其内部完成对卡上文件进行读取和解析，保存信息，并提供对外接口。
因为跟卡直接相关，所以`UiccCarrierPrivilegeRules`在`UiccCard`被创建后初始化。
　　在`UiccCard.update()`函数里创建：
```java
// Reload the carrier privilege rules if necessary.
log("Before privilege rules: " + mCarrierPrivilegeRules + " : " + mCardState);
if (mCarrierPrivilegeRules == null && mCardState == CardState.CARDSTATE_PRESENT) {
    mCarrierPrivilegeRules = new UiccCarrierPrivilegeRules(this,
                        mHandler.obtainMessage(EVENT_CARRIER_PRIVILIGES_LOADED));
} else if (mCarrierPrivilegeRules != null && mCardState != CardState.CARDSTATE_PRESENT) {
    mCarrierPrivilegeRules = null;
}
```
## UiccCarrierPrivilegeRules
先看看该类的class注释:
```
/**
 * Class that reads and stores the carrier privileged rules from the UICC.
 *
 * The rules are read when the class is created, hence it should only be created
 * after the UICC can be read. And it should be deleted when a UICC is changed.
 *
 * The spec for the rules:
 *     GP Secure Element Access Control:
 *     http://www.globalplatform.org/specifications/review/GPD_SE_Access_Control_v1.0.20.pdf
 *     Extension spec:
 *     https://code.google.com/p/seek-for-android/
 *
 *
 * TODO: Notifications.
 *
 * {@hide}
 */
```
在构造函数中开启读取文件的流程，事件驱动。
```
    public UiccCarrierPrivilegeRules(UiccCard uiccCard, Message loadedCallback) {
        Rlog.d(LOG_TAG, "Creating UiccCarrierPrivilegeRules");
        mUiccCard = uiccCard;
        mState = new AtomicInteger(STATE_LOADING);
        mLoadedCallback = loadedCallback;

        // Start loading the rules.
        mUiccCard.iccOpenLogicalChannel(AID,
            obtainMessage(EVENT_OPEN_LOGICAL_CHANNEL_DONE, null));
    }
```
注释里对该类的功能进行了讲解，而且给出了使用到的icc card文件读取和解析的spec规范文档 `GPD_SE_Access_Control_v1.0.20.pdf`，可惜他给的链接无效了，可以在百度文库上找到该spec，地址: [GPD_SE_Access_Control_v1.0.20.pdf](http://wenku.baidu.com/link?url=4BZ92HfppXlHIc1jsKw0ob7n0ppdN68qlUq96Yosl124xDM5ab3a_zs84vQSx6bLAONKsN0BQTvDBwRc300X4H_tKnuW2dcY3UhaGxHRdcK)
　　具体读取icc文件和解析这里就不分析了，都是依照spec的实现。只说明下几个接口和内部变量：

| Field | Note |
| :-------- | :--------|
| AccessRule | 内部类，用来保存解析到的rules，内部维护单个rule的package name和签名hash值. |
| List<AccessRule> mAccessRules | 保存所有的rules在list，看来可以支持多个运营商指定app. |
|areCarrierPriviligeRulesLoaded | 是否已经准备好. |
|getCarrierPrivilegeStatus | 验证当前进程里是否存在有运营商授权的app（多个app可以通过共享id的形式运行在同一个进程里. |
|getCarrierPrivilegeStatusForCurrentTransaction | 验证当前进程里是否存在有运营商授权的app. |
|getCarrierPackageNamesForIntent | 通过从package manager中取出所有符合传入的Intent的app，也就是取出所有可以处理传入的Intent的app，并检查这些app里是否有符合运营商授权的，并返回符合的list. |


# 具体场景
　　以一条新短信的接收为例：
　　在`InboundSmsHandler`里的`processMessagePart()`函数中：　
> **processMessagePart()**函数用来将缓存的短信分段进行组装，如果已经收全，就会将短信广播出去，当然，如果是单段的独立短信该函数也就直接广播了.

来看打包Intent广播的部分：
```java
        Intent intent = new Intent(Intents.SMS_FILTER_ACTION);
        List<String> carrierPackages = null;
        UiccCard card = UiccController.getInstance().getUiccCard();
        if (card != null) {
            carrierPackages = card.getCarrierPackageNamesForIntent(
                    mContext.getPackageManager(), intent);
        }
        if (carrierPackages != null && carrierPackages.size() == 1) {
            intent.setPackage(carrierPackages.get(0));
            intent.putExtra("destport", destPort);
        } else {
            setAndDirectIntent(intent, destPort);
        }

        intent.putExtra("pdus", pdus);
        intent.putExtra("format", tracker.getFormat());
        dispatchIntent(intent, android.Manifest.permission.RECEIVE_SMS,
                AppOpsManager.OP_RECEIVE_SMS, resultReceiver, UserHandle.OWNER);
        return true;
```
首先是新建一个intent，而这个intent的action直接指定为**`Intents.SMS_FILTER_ACTION`**？这个是什么鬼，以前没见过啊。。。跳转过去：
```java
/**
 * Broadcast Action: A new text-based SMS message has been received
 * by the device. This intent will only be delivered to a
 * carrier app which is responsible for filtering the message.
 * If the carrier app wants to drop a message, it should set the result
 * code to {@link android.app.Activity#RESULT_CANCELED}. The carrier app can
 * also modify the SMS PDU by setting the "pdus" value in result extras.</p>
 * ....................
 */
 @SdkConstant(SdkConstantType.BROADCAST_INTENT_ACTION)
 public static final String SMS_FILTER_ACTION =
				"android.provider.Telephony.SMS_FILTER";
```
注释已经很清晰明了了，这个action只会发送给carrier app，而且carrier app可以通过set result为`RESULT_CANCELED`来终止这个广播，这样别的app就永远没有机会收到这个广播了。

　　回到之前的打包intent的代码，其会去UiccCard里通过 `getCarrierPackageNamesForIntent()`方法来得到可以处理`SMS_FILTER_ACTION`的符合运营商授权的app name list，如果能取到，那么就将intent的目标package直接设定为那个app，这样这个短信广播就只会发送给这个授权app；

> **intent.setPackage(carrierPackages.get(0));**

　　而如果没有运营商授权app，那么就会调用`setAndDirectIntent (intent, destPort);`来设定广播app，这里才轮到default sms app：
```
    /**
     * Set the appropriate intent action and direct the intent to the default SMS app or the
     * appropriate port.
     *
     * @param intent the intent to set and direct
     * @param destPort the destination port
     */
    void setAndDirectIntent(Intent intent, int destPort) {
        if (destPort == -1) {
            intent.setAction(Intents.SMS_DELIVER_ACTION);

            // Direct the intent to only the default SMS app. If we can't find a default SMS app
            // then sent it to all broadcast receivers.
            // We are deliberately delivering to the primary user's default SMS App.
            ComponentName componentName = SmsApplication.getDefaultSmsApplication(mContext, true);
            if (componentName != null) {
                // Deliver SMS message only to this receiver.
                intent.setComponent(componentName);
                log("Delivering SMS to: " + componentName.getPackageName() +
                    " " + componentName.getClassName());
            } else {
                intent.setComponent(null);
            }
        } else {
            intent.setAction(Intents.DATA_SMS_RECEIVED_ACTION);
            Uri uri = Uri.parse("sms://localhost:" + destPort);
            intent.setData(uri);
            intent.setComponent(null);
        }
    }
```
　　短信的`desPort`都是**-1**，所以可以只看上面这个if分支，首先先将intent的action修改为` Intents.SMS_DELIVER_ACTION`， 这个是android的新短信常规intent action，**顶替**掉之前的`SMS_FILTER_ACTION`；然后通过`getDefaultSmsApplication`获取到default sms app，如果能取到，那么通过`intent.setComponent(componentName)`设置目标package为这个app，如果没有，那么就`setComponent(null)`，这样就可以广播给所有可以接收**SMS_DELIVER_ACTION**的app。

 另外，提一点另外的细节，打包广播短信的地方：
> dispatchIntent(intent, android.Manifest.permission.RECEIVE_SMS, AppOpsManager.OP_RECEIVE_SMS, **`resultReceiver`**, UserHandle.OWNER);

第4个参数，传入的`resultReceiver`，是个内部类SmsBroadcastReceiver对象，用来处理短信广播的结果，对每种intent action广播出去之后的处理结果都有分别处理，如从缓存数据库中删除短信、更新短信状态等。这个广播也用来`SMS_DELIVER_ACTION`的广播结束后，重新将短信以`SMS_RECEIVED_ACTION`广播出去，下节对其进行说明。

## 三种SMS ACTION

所以到目前为止，我们可以看到总共有3种SMS的广播类型，`SMS_FILTER_ACTION`，`SMS_DELIVER_ACTION`以及`SMS_RECEIVED_ACTION`;
下面整理出了他们的区别：

| Actions | Note|
| :-------- | :--------|
| SMS_FILTER_ACTION | 运营商SMS APP专用，第一优先级发送，运营商APP可以修改这个短信的任何内容，也可以直接丢弃掉这个短信不让系统保存和继续广播. |
| SMS_DELIVER_ACTION | 第二优先级的广播，该广播会被显式指定给一个default SMS APP接受，而且系统又通过一定的机制保证了同一时刻只会有 <= 1个default SMS APP, 所以这个广播最多只会发送给1个APP，这个APP负责存储和通知新短信. |
| SMS_RECEIVED_ACTION | 最低优先级广播，在 SMS_DELIVER_ACTION 广播结束后触发，不指定APP Name，所以声明了该广播的所有APP都可以接收到新短信广播，注册该广播的应用并不被期望会去存储这条短信到短信数据库. |


所以，如果你要开发一个短信功能的APP，就要注意了，首先`SMS_FILTER_ACTION`只针对运营商应用，所以第三方用不了，也不用去管；其次，衡量一下你的SMS APP所要提供的功能，如果你想提供读写系统短信数据库（主要是写，读都可以读，写只有DEFAULT SMS APP可以写）的能力，想提供一个类似系统SMS APP的应用的话，就需要声明`SMS_DELIVER_ACTION`，并想办法提示用户把你设置为默认；
而如果你仅仅想监控一下你所关心的短信，并不关心保存，那么可以声明最低优先级的`SMS_RECEIVED_ACTION`广播，这个广播还能兼容低版本android。。。

# 总结
　　FW初始化时，首先尝试设定一个`default SMS APP`，同时，在卡槽的icc卡准备好后，开始读取卡上的运营商授权 APP 数据，并保存下来；新短信接收时首先通过接口获取到运营商授权APP，如果没有，再通过接口获取到default SMS APP，如果还没有，就直接广播啦。

　　运营商授权app的优先级大于default SMS APP。