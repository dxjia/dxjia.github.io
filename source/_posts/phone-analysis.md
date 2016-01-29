title: phone-analysis
date: 2016-01-29 20:35:12
tags: [Android, Phone]
categories: 技术学习
description: Android5.0 Phone框架分析
---

# persistent属性
　　要想了解phone的框架，首先需要了解android app的persistent属性。在AndroidManifest.xml定义中，application有这么一个属性android:persistent，被android:persistent=”true”修饰的应用会在系统启动之后被AM(ActivityManagerService)启动。
　　AM首先在systemready后去PM(PackageManagerService)中查找设置了android:persistent的应用，代码如下：
```java
public void systemReady(final Runnable goingCallback) {
............
        synchronized (this) {
            if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
                try {
                    List apps = AppGlobals.getPackageManager().
                        getPersistentApplications(STOCK_PM_FLAGS);
                    if (apps != null) {
                        int N = apps.size();
                        int i;
                        for (i=0; i<N; i++) {
                            ApplicationInfo info
                                = (ApplicationInfo)apps.get(i);
                            if (info != null &&
                                    !info.packageName.equals("android")) {
                                addAppLocked(info, false, null /* ABI override */);
                           }
                        }
                    }
                } catch (RemoteException ex) {
                    // pm is in same process, this will never happen.
                }
            }
........

}
```
addAppLocked方法会检测应用是否有起来，如果没有将启动，这样persist属性的应用就跑起来了。注意上面的判断：
```
!info.packageName.equals("android")
```
因为name为android的persist app是 framework-res，所以就排除在外。

Android5.0中有persistent=true的模块有下面几个：
>**./packages/services/Telecomm/AndroidManifest.xml:            android:persistent="true"<br>./packages/services/Telephony/AndroidManifest.xml:                 android:persistent="true"<br>./packages/apps/Nfc/AndroidManifest.xml:                 android:persistent="true"<br>./frameworks/base/packages/FakeOemFeatures/AndroidManifest.xml:        android:persistent="true"<br>./frameworks/base/packages/Keyguard/AndroidManifest.xml:        android:persistent="true"<br>./frameworks/base/packages/SystemUI/AndroidManifest.xml:        android:persistent="true"<br>./frameworks/base/core/res/AndroidManifest.xml:                 android:persistent="true"<br>./hardware/intel/common/utils/ituxd/AndroidManifest.xml:     android:persistent="true">**

跟Telephony框架有关的就是前面两个，Telecom和Telephony，这两个模块的代码位置及在手机中实际编译出的apk如下：
>**./packages/services/Telecom  --  `Telecom.apk`<br>./packages/services/Telephony  -- `Teleservice.apk`**

Telephony模块包含全部的PhoneApp及整个Phone框架，而Telecom主要是一些call相关的receiver、activity以及service等，看起来google是想把应用与Phone框架分的开一些，原先都是在一起的，只是从目前5.0的代码来看，还只是弄了一小部分，估计是在进行中，可能后续版本这里还会有变化。
# Phone框架
从Telephony模块的manifest.xml里可以看出其实他目前还是叫做PhoneApp，这主要是因为之前一直以来Phone框架跟phone app是揉在一起的，估计等以后版本将Telecom和Telephony分的更开的时候，这里就不会叫这个名字啦。
```xml
<application android:name="PhoneApp"
             android:persistent="true"
             android:label="@string/phoneAppLabel"
             android:icon="@mipmap/ic_launcher_phone"
             android:allowBackup="false"
             android:supportsRtl="true">
```
我们先从PhoneApp的启动看起
## PhoneApp

 >**(packages\services\telephony\src\com\android\phone)**


```java
public void onCreate() {
        if (UserHandle.myUserId() == 0) {
            // We are running as the primary user, so should bring up the
            // global phone state.
            mPhoneGlobals = new PhoneGlobals(this);
            mPhoneGlobals.onCreate();
 
            mTelephonyGlobals = new TelephonyGlobals(this);
            mTelephonyGlobals.onCreate();
        }
}
```

TelephonyGlobals 是5.0新增的，初步来看是跟账户控制以及tty有关的。后面再来研究他的作用，先看PhoneGloabals。

## PhoneGloabals

PhoneGlobals继承ContextWrapper，而且还是单例，提供全局性的信息，包含的信息很多，下面是其内部管理的关键实例，这些实例相互关联才构建其手机的通信功能。
```
private static PhoneGlobals sMe;
CallController callController;
CallManager mCM;
CallNotifier notifier;
CallerInfoCache callerInfoCache;
NotificationMgr notificationMgr;
Phone phone;
PhoneInterfaceManager phoneMgr;
```
看其被PhoneApp.java调用的onCreate函数。
创建Phones
```java
            // Initialize the telephony framework
            PhoneFactory.makeDefaultPhones(this);
```
使用工厂模式，创建phones，PhoneFactory提供的都是static方法，所有都是直接静态调用。
### PhoneFactory
首先看看PhoneFactory维护的本地变量：
```
    static private PhoneProxy[] sProxyPhones = null;
    static private PhoneProxy sProxyPhone = null;
    static private CommandsInterface[] sCommandsInterfaces = null;
    static private ProxyController mProxyController;
    static private UiccController mUiccController;
    static private CommandsInterface sCommandsInterface = null;
    static private SubInfoRecordUpdater sSubInfoRecordUpdater = null;
    static private boolean sMadeDefaults = false;
    static private PhoneNotifier sPhoneNotifier;
```
可以注意到sProxyPhones 等几个变量，跟以往的版本比较，已经变成了数组了，这是因为google开始支持双卡啦。。。啦啦啦。。。
- 1 等待底层socket就绪，也就是init创建rild的socket完成；使用for循环+sleep的方式；
- 2 创建PhoneNotifier以及获取network mode和cdma subscription;
- 3 获取Phone个数设置
```java
 /* In case of multi SIM mode two instances of PhoneProxy, RIL are created,
    where as in single SIM mode only instance. isMultiSimEnabled() function checks
    whether it is single SIM or multi SIM mode */
            int numPhones = TelephonyManager.getDefault().getPhoneCount();
            int[] networkModes = new int[numPhones];
            sProxyPhones = new PhoneProxy[numPhones];
            sCommandsInterfaces = new RIL[numPhones];
```
getPhoneCount()从setting设置中取:
```java
   /**
     * Returns the multi SIM variant
     * Returns DSDS for Dual SIM Dual Standby
     * Returns DSDA for Dual SIM Dual Active
     * Returns TSTS for Triple SIM Triple Standby
     * Returns UNKNOWN for others
     */
    /** {@hide} */
    public MultiSimVariants getMultiSimConfiguration() {
        String mSimConfig =  SystemProperties.get(TelephonyProperties.PROPERTY_MULTI_SIM_CONFIG);
        if (mSimConfig.equals("dsds")) {
            return MultiSimVariants.DSDS;
        } else if (mSimConfig.equals("dsda")) {
            return MultiSimVariants.DSDA;
        } else if (mSimConfig.equals("tsts")) {
            return MultiSimVariants.TSTS;
        } else {
            return MultiSimVariants.UNKNOWN;
        }
}
```
- 4 创建对应数量的RIL实例
```java
for (int i = 0; i < numPhones; i++) {
    //reads the system properties and makes commandsinterface
    sCommandsInterfaces[i] = new RIL(context, networkModes[i], cdmaSubscription, i);
}
```
- 5 初始化 SubscriptionController 和UiccControler，这两个都是单例设计，这里创建好后，别的地方只需要getInstance进行调用;
- 6  创建Phone实例并保存;
```java
for (int i = 0; i < numPhones; i++) {
    PhoneBase phone = null;
    int phoneType = TelephonyManager.getPhoneType(networkModes[i]);
    if (phoneType == PhoneConstants.PHONE_TYPE_GSM) {
        phone = new GSMPhone(context, sCommandsInterfaces[i], sPhoneNotifier, i);
    } else if (phoneType == PhoneConstants.PHONE_TYPE_CDMA) {
        phone = new CDMALTEPhone(context, sCommandsInterfaces[i], sPhoneNotifier, i);
    }
    Rlog.i(LOG_TAG, "Creating Phone with type = " + phoneType + " sub = " + i);

    sProxyPhones[i] = new PhoneProxy(phone);
}
```
- 7 创建ProxyControler
```
    mProxyController = ProxyController.getInstance(context, 
                                    sProxyPhones, 
                                    mUiccController, 
                                    sCommandsInterfaces);
```
ProxyController也是5.0新增的，其作用是进行双卡控制，内部实例化对icccard dct(data connction) phonebook 以及sms这些跟卡关系比较密切的功能，如下：
```
mDctController = DctController.makeDctController((PhoneProxy[])phoneProxy);
mUiccPhoneBookController = new UiccPhoneBookController(mProxyPhones);
mPhoneSubInfoController = new PhoneSubInfoController(mProxyPhones);
mUiccSmsController = new UiccSmsController(mProxyPhones);
```
这些实例化的功能，会在各自的构造函数中将接口addService到servicemanager以便供APP调用。实现机制为binder.

比如UiccSmsController，这里是单实例的，但其内部接口使用带subid参数的方式来支持双卡
- 8 获取默认SMS应用
主动调用一次 SmsApplication.getDefaultSmsApplication(context, true );注意第二个参数为true，也就是如果手机没有获取到默认sms app，那么会尝试去设定一个。设定的规则如下：
>**1.  首先尝试从用户指定的默认app，对应的系统setting key为：sms_default_application；<br>2. 其次看是否有goole的官方 默认sms app；<br>3. 如果以上两个都没有，那么就从PM中获取所有注册有完整sms有关的broadcast receiver的app，从中找一个优先级最高的，并将其设定为default app。**

- 9 监控短信DefaultApp的变动
- 10 监控Subscription的变化，跟卡有关，用来控制整个FW层对双卡的区分。相关的几个类是：SubscriptionManager  SubscriptionController  SubInfoRecordUpdater，管理default subid等，其default的策略是第一个检测到的可用卡id；当然接口是Public的，也就是暴漏出来的，是可以在需要的地方进行setDefault来改变这个设定的，这些值最终都保存在setting数据库中。

至此，PhoneFactory.makeDefaultPhones完成，接下来再回到PhoneGlobals...
##Default Phone
```java
            // Get the default phone
            phone = PhoneFactory.getDefaultPhone();
```
上面创建完phones之后，接下来这里就取出一个defaultphone，这里说明一下default phone的设定，第一次在PhoneFactory中创建出phones之后，将实例保存在数组里;
```java
static private PhoneProxy[] sProxyPhones = null;  //用于保存创建的Phones
static private PhoneProxy sProxyPhone = null;   //用于保存default phone
```
sProxyPhone的第一次赋值只是简单的取 sProxyPhones[0]，但PhoneFactory提供了接口，可以对这个default phone进行设定，接口为：`setDefaultSubscription(int subId)`；当然其内部会根据双模的具体情况进行决策，比如如果是双模单通(同一时间只有一个active)，那么default phone会自动设定成那个active的，如果是双模双通，那么就设定成参数值int subId(当然会将subid转换为对应的Phoneid，也就是sProxyPhones数组下标)。
## 创建关键实例
接下来在PhoneGlobals中会创建很多关键实例，依次是：
>**`CallManager`<br>`NotificationMgr`<br>`PowerManager`<br>`KeyguardManager`<br>`CallLogger`<br>`CallGatewayManager`<br>`CallController`<br>`CallerInfoCache`<br>`BluetoothManager`<br>`PhoneInterfaceManager`<br>`CallNotifier`**

## 双模通道打通
上面的初始化过程结束之后，其实已经对各模建立起来各自的通道，尽管5.0还没有彻底支持完整，还是以一个例子来描述流程：

`PhoneInterfaceManager`是一个servcie，App可以远程访问其内部的接口，以下面这个接口为例，其实现中提供了一些以subId作为参数的接口，subId是long型，你可以把它看作是卡的身份标识，具体获得过程以后再分析。
```java
    public void toggleRadioOnOffForSubscriber(long subId) {
        enforceModifyPermission();
        getPhone(subId).setRadioPower(!isRadioOnForSubscriber(subId));
    }
```
 
GetPhone(subId)会获取到对应的Phone实例，并调用对应phone实例的setRadioPower接口，可惜这里5.0还没有进行实现，只是简单的return defaultPhone
```java
    // returns phone associated with the subId.
    // getPhone(0) returns default phone in single SIM mode.
    private Phone getPhone(long subId) {
        // FIXME: hack for the moment
        return mPhone;
        // return PhoneUtils.getPhoneForSubscriber(subId);
    }
```
前面说过了，会根据phone的个数创建对应数量的`RILJ`实例，也就是`CommandInterface`实例，并传给具体的Phone实例，CDMAPhone或GsmPhone，而各自的RILJ实例在初始化的时候，又会自动链接上`RILD`的对应socket[RILD由init进程启动，并根据`init.rc`里的设定创建对应的socket]，那么各自的phone跟底层modem的通信就建立了。
