title: ContentProvider详解
date: 2016-01-27 19:28:45
tags: [Android,Provider]
categories: 技术学习
description: 本文将从跨进程(IPC)调用的角度来分析Android中是怎么完成一次跨进程的provider数据访问的。。。
---
本文将从跨进程(IPC)调用的角度来分析源码，详细说明Android中是怎么完成了一次跨进程的provider数据访问的，基于android6.0。。。

<!--more-->

## 引子
先来看一段APP使用provider的代码示例，十分的常见，比如在我们的APP中查询一个指定号码的联系人的名字：
```java
String number = "13900001234";  
Uri uri = Uri.parse("content://com.android.contacts/data/phones/filter/" + number);  
ContentResolver resolver = getContext().getContentResolver();  
Cursor cursor = resolver.query(uri, new String[]{"display_name"}, null, null, null);  
if(cursor.moveToFirst()){  
    String name = cursor.getString(0);
    // do our things...
}
// dont forget to close cursor after using
cursor.close();
```
步骤：
- 组织一个严格正确的`uri`；
- 通过本`app`所在的应用上下文`context`获取到一个`ContentResolver`实例；
- 调用`resolver.query(uri, .....)`得到数据结果集合`cursor`；
- 从得到的数据集合中提取数据；

所以从这里来看，通过`context`拿到的`resolver`就可以直接访问`contact`进程的`provider`啦；所以`IPC`应该发生在`context`、`resolver`和`query`之间。
## 认识Context
先来理解一下什么是`Context`，看看`Android`给出的注释：
```java
/*
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
```
从上我们可以得到一些概括性的信息：
`Context`描述的是一个应用程序环境的信息，即`上下文`，该类是一个抽象(abstract class)类，Android提供了该抽象类的具体实现类(`ContextImpl`类)，通过它我们可以获取应用程序的资源和类，也包括一些应用级别操作，例如：启动一个Activity，发送广播，接受Intent信息等。于是，我们就可以利用该`Context`对象去构建`应用级别操作`(application-level operations) 。

这里要提出的是，Application、Activity以及Service都间接继承自`Context`，他们的创建都会为自己创建`ContextImpl`实例，所以一个`APP`中的`context`个数并`不`只有一个，`最大个数=1+Activity个数+Service个数`，而且是随着Activity和Service的启动与销毁动态改变的。

既然不是一个，那么可能就会奇怪，为什么我们使用时总能通过不同`context`访问到相同的程序`res`、`assets`，以及都能做一些其他app-level的事情呢，这是因为虽然`ContextImpl`实例各有各的，但其内部所维护的`resourse manager`，`packageInfo`以及`main thread`却实打实的指向同一个，这个还是很好理解的，毕竟都是在同一个`application`里的啊，从`AndroidManifest.xml`的包含关系就能看出来，`activity`，`service`等标签都包含在`application`标签里面。

这里不再对`Context`进行过多的讲解，更多有关`Context`的理解可以参考<http://blog.csdn.net/singwhatiwanna/article/details/21829971> 这篇文章，写的不错。。。

## APP主线程(ActivityThread)
上面我们有提到`ContextImpl`里的`main thread`，由于接下来要讲的`contentresolver`的实现跟这个`主线程`息息相关，所以我们这里单独对其进行一下说明。
每个`APP`运行起来的时候，程序入口都是在`ActivityThread.java`的`main`函数里，(AMS启动进程通过`Process.start`通知`zygote`来`fork`进程这里就不详细说明了，`fork`之后，还是会调用`ActivityThread`里的`main`函数作为程序启动入口)，该入口会创建`ActivityThread`实例
```java
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Looper.loop();
```
这个`ActivityThread`实例会作为每个`APP`进程的唯一的一个`main thread`存在，并且其会根据AMS的要求（通过IApplicationThread接口，AMS为Client、ActivityThread.ApplicationThread为Server）负责调度和执行activities、broadcasts以及其它操作。
> 在Android系统中，在默认情况下，一个应用程序内的各个组件(如Activity、BroadcastReceiver、Service)都会在同一个进程(Process)里执行，且由此进程的**主线程**负责执行。如果有特别指定(通过`android:process`)，也可以让特定组件在不同的进程中运行。无论组件在哪一个进程中运行，默认情况下，他们都由此进程的**主线程**负责执行。

**主线程**的主要职责：
- `快速处理UI事件`, 并且只有它才能处理UI事件， 其它线程都不能存取UI画面上的对象(如TextView等)，此时， 所以一般也被称为`UI主线程`。基本上，Android希望UI线程能根据用户的要求做出快速响应，如果UI线程花太多时间处理后台的工作，当UI事件发生时，让用户等待时间超过5秒而未处理，就会产生`ANR`。
- `快速处理Broadcast消息`，**主线程**除了处理UI事件之外，还要处理Broadcast消息。所以在BroadcastReceiver的onReceive()函数中，不宜占用太长的时间，否则导致**主线程**无法处理其它的Broadcast消息或UI事件。如果占用时间超过10秒， 那么就`ANR`。

上一节中提到的`ContextImpl`，他的构造函数是私有的，并通过几个静态函数向外提供创建方法：
- `static ContextImpl createSystemContext(ActivityThread mainThread)`，提供给system_server进程使用的，在开机初始化时调用；
- `static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo)`，提供给`app`和`service`的context创建；
- `static ContextImpl createActivityContext(ActivityThread mainThread,            LoadedApk packageInfo, int displayId, Configuration overrideConfiguration)`，提供给`activity`的context创建。

可以看到其中都会传递 `ActivityThread mainThread`，他们都是`同一个`哦~


## ContentResolover
根据上面的讲解，我们知道开始的那个[例子](#1-引子)中的`getContext().getContentResolver()`最终会调用到当前类(当前类可能是个activity、service或者application)的`ContextImpl`实例中，我们来看看`ContextImple`里的`getContentResolver()`函数实现：
```java
    @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```
直接返回了本地变量`mContentResolver`，而这个变量是在构造函数中创建，真实类型为`ContextImpl`的内部类`ApplicationContentResolver `；
```java
private final ApplicationContentResolver mContentResolver;
......
......
mContentResolver = new ApplicationContentResolver(this, mainThread, user);
```
`ApplicationContentResolver` 继承自虚基类`ContentResolver`，实现了几个`acquire provider`的虚函数，而实现内部都是通过构造函数里传入的`mainThread`进程转包；
```java
        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
```

到这里我们可以清楚的看到，`getContentResolver`拿到了自己的`ContextImpl`实例内的`ApplicationContentResolver`实例，其内部持有整个进程的`主线程`引用，所以可以访问`主线程`里的函数。也就是说
`【NOTE】`
> 通过`getContext().getContentResolver()`我们可以访问本进程的`主线程`里的函数啦

另外还会发现`ContentResolver`里实现了大量跟provider里的方法一样的函数，其目的是为了与`provider`的转接，后面会讲到，其真正的query是怎么转接到真正的provider上去的。

但目前为止，我们仍然还都是运行在自己的进程之内的，还没有真正的跨进程访问呢，继续。。。。。

## acquireProvider
[例子](#1-引子)中拿到`ContentResolver`之后，就直接开始使用`query`函数进行查询啦，来看看这个`query`实现：
```java
    public final @Nullable Cursor query(final @NonNull Uri uri, @Nullable String[] projection,
            @Nullable String selection, @Nullable String[] selectionArgs,
            @Nullable String sortOrder, @Nullable CancellationSignal cancellationSignal) {
        Preconditions.checkNotNull(uri, "uri");
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try {
            long startTime = SystemClock.uptimeMillis();

            ICancellationSignal remoteCancellationSignal = null;
            if (cancellationSignal != null) {
                cancellationSignal.throwIfCanceled();
                remoteCancellationSignal = unstableProvider.createCancellationSignal();
                cancellationSignal.setRemote(remoteCancellationSignal);
            }
            try {
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                // The remote process has died...  but we only hold an unstable
                // reference though, so we might recover!!!  Let's try!!!!
                // This is exciting!!1!!1!!!!1
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                if (stableProvider == null) {
                    return null;
                }
                qCursor = stableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
            if (qCursor == null) {
                return null;
            }

            // Force query execution.  Might fail and throw a runtime exception here.
            qCursor.getCount();
            long durationMillis = SystemClock.uptimeMillis() - startTime;
            maybeLogQueryToEventLog(durationMillis, uri, projection, selection, sortOrder);

            // Wrap the cursor object into CursorWrapperInner object.
            CursorWrapperInner wrapper = new CursorWrapperInner(qCursor,
                    stableProvider != null ? stableProvider : acquireProvider(uri));
            stableProvider = null;
            qCursor = null;
            return wrapper;
        } catch (RemoteException e) {
            // Arbitrary and not worth documenting, as Activity
            // Manager will kill this process shortly anyway.
            return null;
        } finally {
            if (qCursor != null) {
                qCursor.close();
            }
            if (cancellationSignal != null) {
                cancellationSignal.setRemote(null);
            }
            if (unstableProvider != null) {
                releaseUnstableProvider(unstableProvider);
            }
            if (stableProvider != null) {
                releaseProvider(stableProvider);
            }
        }
    }
```
从实现来看，先通过调用`acquireUnstableProvider`尝试得到一个`不稳定`的`IContentProvider`实例(从名字来看，ContentProvider肯定是使用`binder`来实现IPC的)，何为`不稳定`呢？我的理解是，并非是`为了得到不稳定`的实例，而是说去尝试得到一个实例，而这个实例可能会是`不稳定`、`不可用`的，为什么会不可用呢，答案是，`主线程`内部都会对本进程所曾经访问到过的所有`provider`本地缓存一个访问对象，维护在一个`ArrayMap`中，这样，每次来`aquire provider`的时候，都先从本地缓存中找，但由于对方`provider`所在进程不一定都还活着，所以本进程内部缓存的`provider`引用，就不一定都可用，如果不可能，就会在后面的操作中，触发`DeadObjectException`异常；而如果可用，这样的机制无疑会大大提高效率。

所以上面的代码中也看出，`acquireUnstableProvider`拿不到provider或者拿到的provider执行`query`失败，就会重新调用`stableProvider = acquireProvider(uri);`重新去得到一个`provider`，所以如果这是我们的进程首次访问对方`provider`，肯定会走到`acquireProvider`的，我们重点来看这个函数的实现。
```java
    public final IContentProvider acquireUnstableProvider(Uri uri) {
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            return null;
        }
        String auth = uri.getAuthority();
        if (auth != null) {
            return acquireUnstableProvider(mContext, uri.getAuthority());
        }
        return null;
    }
```
可以看出以下三点：
- 只处理`content`打头的uri；
- 将uri的`authority`作为`provider`的区分(`上一篇文章里也提到这个authority必须唯一，原因就在这里`)；
- 调用子类`ApplicationContentResolver`的`acquireProvider(Context context, String auth)`方法。

接下来看`ApplicationContentResolver`里的`acquireProvider(Context context, String auth)`方法实现：
```java
        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
```
转包到`主线程`中了：
```java
    public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        // There is a possible race here.  Another thread may try to acquire
        // the same provider at the same time.  When this happens, we want to ensure
        // that the first one wins.
        // Note that we cannot hold the lock while acquiring and installing the
        // provider since it might take a long time to run and it could also potentially
        // be re-entrant in the case where the provider is in the same process.
        IActivityManager.ContentProviderHolder holder = null;
        try {
            holder = ActivityManagerNative.getDefault().getContentProvider(
                    getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) {
        }
        if (holder == null) {
            Slog.e(TAG, "Failed to find provider info for " + auth);
            return null;
        }

        // Install provider will increment the reference count for us, and break
        // any ties in the race.
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```
从上面的调用流程上，我们可以提取出`三个`关键点：
- `acquireExistingProvider` 从本地缓存中查找有无对应的provider，有就直接返回；这里只考虑首次访问的情况，所以肯定返回Null，分析略过；
- `ActivityManagerNative.getDefault().getContentProvider`，调用`AMS`的`getContentProvider`，最终会通过`binder`通信调用到`ActivityManagerService`的`getContentProviderImpl`，这块的逻辑比较复杂，涉及到了Android组件启动的过程，我们只需知道这个调用会阻塞直到返回；这里第一次出现IPC了，之前还都是在自己进程内部执行，这里跨进程到AMS获取一个IContentProvider对象；
- `installProvider`，将得到的`IContentProvider`进行本地缓存，增加引用计数等，以便以后直接使用。
> 要特别`【注意】`，这里最后的`installProvider`调用还是在本地进程中，并且注意`第二个`参数，是`holder`，不为null，这里特地强调这个，是因为等会对方provider所在的进程的`主线程`(也是`ActivityThread`文件)里流程上同样会调用到这个函数，只是`第二个`参数不一样，切记切记。。。。

## 调用者进程在AMS中的流程
前面分析到流程，通过IPC调用，进入了AMS中。
AMS其本身运行在`system_server`进程中，实现了`binder`通信机制，所以可以被大家进行`IPC`调用，而且其作为所有进程的一个大总管，其内部维护着各个进程对应的一些数据结构，如`ActivityRecord`、`BroadcastRecord`以及`ContentProviderRecord`，以及与之相应的map存储结构，维护这些信息可以是它能够管理进程的各个组件之间的关系，甚至是各个进程之间的关系；比如对于本文的provider，AMS就能做到，如果`provider`所在的进程crash掉了，那么AMS可以去杀死所有正在使用这个`provider`的进程，做的有点绝吧？让他自己死也没什么问题啊。
接下来我们主要来看`getContentProvider()`在AMS中都干了啥；签名也提到了`ActivityManagerNative.getDefault().getContentProvider`的调用最终进入到`ActivityManagerService`的`getContentProviderImpl`中，函数太长了，我这里只截取几个关键的地方，并在关键点上进行了注释：
```java
private final ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, boolean stable, int userId) {
    ContentProviderRecord cpr;
    ContentProviderConnection conn = null;
    ProviderInfo cpi = null;

    synchronized(this) {
        ProcessRecord r = null;
		// 确保调用者有效
        if (caller != null) {
            r = getRecordForAppLocked(caller);
			......
        }

		// 检查是否已经在AMS注册过，name参数为目标ContentProvider的authority
        // First check if this content provider has been published...
        cpr = mProviderMap.getProviderByName(name, userId);

		// 如果能找到，说明目标provider已经存在
        boolean providerRunning = cpr != null;
        if (providerRunning) {
			......// 如果该provider已经存在，则进行相应处理
        }

		// 如果目标provider所在进程还未启动
        if (!providerRunning) {
            try {
				// 从PackageManagerService中获取到这个provider的ProviderInfo，
				// 应该就是写在AndroidManifest里的provider声明
                cpi = AppGlobals.getPackageManager().
                      resolveContentProvider(name,
                      STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
                }
				......

            ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
			// 从AMS维护的map中尝试取出ContentProviderRecord，如果目标provider所在的进程
			// 还没有使用过这个provider，那么肯定是取不到的
            cpr = mProviderMap.getProviderByClass(comp, userId);

            final boolean firstClass = cpr == null;
			// 所以第一次时 firstClass肯定为true
            if (firstClass) {
                try {
                    ApplicationInfo ai =
                            AppGlobals.getPackageManager().
                                getApplicationInfo(
                                        cpi.applicationInfo.packageName,
                                        STOCK_PM_FLAGS, userId);

                        ai = getAppInfoForUser(ai, userId);
                        cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
                    }
                }

				// 至此，我们变为目标provider构建了一个ContentProviderRecord对象
				
				// 判断这个provider的属性是否能够直接运行在调用者所在的进程里，
				// 比如 1. 目标provider显式声明了android:multiprocess="true"，这表明
				//         该provider允许每个访问进程都能持有一个单独provider实例；
				//         而默认情况下，android:multiprocess是为false的，也就是都是单例的，
				//         所有访问进程通过IPC调用最终使用的都是同一个provider实例，所以
				//         在实现provider时一定要注意访问控制，因为这种情况下，它不是线程安全的；
				//      2. 要使用的provider本身就是在自己进程内部的，这自然可以执行在其所在进程。
                if (r != null && cpr.canRunHere(r)) {
                    return cpr.newHolder(null);
                }

				// 如果不能直接运行在调用者所在的进程里，那么我们就从待启动的provider里去找，
				// 系统中所有正在加载的Content Provider都保存在mLaunchingProviders
				// 成员变量中。在加载相应的Content Provider之前，首先要判断一下它是可否
				// 正在被其它应用程序加载，如果是的话，就不用重复加载了，这种并发的情况肯定存在的。
				final int N = mLaunchingProviders.size();
                int i;
                for (i = 0; i < N; i++) {
                    if (mLaunchingProviders.get(i) == cpr) {
                        break;
                    }
                }

                // 如果i >= N，就说明目前还没有人要启动目标provider，也就是还没有实例化
                if (i >= N) {

                    try {

                        // 判断目标provider所在的进程是否已经在运行了
                        ProcessRecord proc = getProcessRecordLocked(
                                cpi.processName, cpr.appInfo.uid, false);
                        if (proc != null && proc.thread != null) {

                            if (!proc.pubProviders.containsKey(cpi.name)) {
                                proc.pubProviders.put(cpi.name, cpr);
                                try {
									// 如果进程已经在运行，但却在进程里没有找到对应的provider(这种情况比较少吧)，
									// 需要调用目标进程的installProvider来发布provider
                                    proc.thread.scheduleInstallProvider(cpi);
                                } 
                            } else ... // 存在，增加引用计数等直接使用
                        } else {
							// 如果还没有启动，那么就启动对方进程
                            proc = startProcessLocked(cpi.processName,
                                    cpr.appInfo, false, 0, "content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name), false, false, false);

                        }
                        cpr.launchingApp = proc;
                        // 把待启动的provider方法待启动map，告诉别人已经有人在启动它了，等着就好了
                        mLaunchingProviders.add(cpr);
                    }
                }


                // 将providerrecord加入map管理，以类名
                if (firstClass) {
                    mProviderMap.putProviderByClass(comp, cpr);
                }
                
                // // 将providerrecord加入map管理，以authority
                mProviderMap.putProviderByName(name, cpr);
				
                conn = incProviderCountLocked(r, cpr, token, stable);
				
                if (conn != null) {
                    conn.waiting = true;
                }
            }
        }

        // 通过synchronized (cpr)确保线程安全
        synchronized (cpr) {
            while (cpr.provider == null) {

                try {
                    if (conn != null) {
                        conn.waiting = true;
                    }
				// 通过wait()一直等在这里，前面我们已经知道，已经将这个cpr
				// 加入到了AMS的map中，所有，这里wait()后，肯定有哪个地方会notify释放掉，
				// 使这里可以继续执行下去，到底是哪里呢，后面我们会进行介绍；
                   cpr.wait();
            } catch (InterruptedException ex) {
            } finally {
                if (conn != null) {
                    conn.waiting = false;
                }
            }
            }
        }
    return cpr != null ? cpr.newHolder(conn) : null;
}
```
请仔细阅读上面的注释，总结下面有下面几个关键的点：
1. 在AMS中，设置了一个类型为`ProviderMap`的成员变量，用来保存所有启动了的provider的信息，`ProviderMap`被设计为可以通过`authoriry`值和`类名`来查找provider。AMS在自己维护的map中查找是否已经存在了可以使用的provider，如果有直接返回，这种情况下，目标进程肯定是运行状态，而且已经在AMS中注册过provider了；
2. 如果第1步中找不到，那么就先去`PKMS`中查询这个provider的一些信息，以备下一步使用；
3. 检查目标provider所在的进程是否已经启动了，如果已经启动了，那么就检查目标进程里是否已经有了要使用的Provider的信息了，如果没有(这个情况估计比较少)，就会调用目标进程的的`主线程`的`installProvider`来重新发布provider；发布provider，就是把实例化的provider对应的IContentProvider(`binder`机制，Bp客户端proxy)注册在AMS里；如果已经有了，那么就直接增加引用计数等，也就是直接使用啦。
4. 如果第3步中，发现对方进程没有启动，那么会使用`startProcessLocked`来启动对方进程。启动新进程的过程比较复杂了，可以去搜文学习一下，这里给出结论，目标进程启动后，会调用一系列AMS的方法，比如`attachApplicationLocked`等，这些函数最终会将它自己的provider发布到AMS的map里；
5. 调用启动进程成功后，AMS这里就往下继续执行了，`startProcessLocked`返回时，AMS这里只能知道对方进程已经起来了，至于对方执行到哪一步了，此时AMS并不会明确知道，因为对方已经是在一个独立的进程里自己跑了，所以接下来我们可以看到AMS通过`cpr.wait()`的方式等在那里，直到有人在`cpr`这个对象上进行`notify`，谁会`notify`呢，当然是对方进程，上一点里提到了，对方进程跑起来后，会去AMS里`attach`自己，并会最终将自己的provider实例的binder代理发布到AMS的map里，这个时候，它就会`notifyAll`，所以到这里的状况就是：`调用者进程通过IPC调用访问AMS，并wait在AMS的某一个变量上；provider所在进程被AMS启动起来，并将自己的provider代理在AMS中进行注册，随后notifyAll`，一定要注意哦，这里有`三个`进程，一个是`调用者进程`，一个是`provider所在进程`，一个是`AMS`自己（也就是system_server进程），`调用者进程`和`provider所在进程`，前者`wait()`在AMS的某个变量上，而后者会去`notify`;
6. 继续回到调用者进程IPC调用AMS上来，在被`notify`之后，就会将一个`ContentProviderHolder`返回给调用者进程了，至于这个`ContentProviderHolder`是啥就不再细致分析了，只需要知道其内部包含了IContentProvider就可以了，而客户端进程需要的就是`IContentProvider`;

## 目标进程以及其在AMS中的流程
也就是目标provider的发布过程，上一节中讲到了`provider`所在的目标进程被AMS start起来之后，目标进程自己会进行一些application的attach工作，这个章节就来讲解这个过程。

`【note】`，接下来的分析主体是`provider`所在的目标进程了，所以主体是执行在目标进程空间里的。

先看看目标进程的`主线程`入口都干了啥：
`ActivityThread.java (frameworks\base\core\java\android\app)`main函数：
```java
    private void attach(boolean system) {

        if (!system) {

            try {
	            final IActivityManager mgr = ActivityManagerNative.getDefault();
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }

        } ......
    }
```
调用了`attachApplication`，但传递的参数不是自己(ActivityThread)，而是一个成员变量mAppThread：
```java
final ApplicationThread mAppThread = new ApplicationThread();
```

这个是什么鬼？跟进去看一下，你就会发现 `ApplicationThread`继承自`ApplicationThreadNative`，而`ApplicationThreadNative`继承自`IBinder`，原来又传了一个可以跨进程的Binder客户端代理进去，本身这里就是跨进程调用AMS，结果又送给AMS一个可以跨进程回自己进程的代理过去，这是双通啊有木有~

继续回到`attachApplication`调用上来，这时候已经IPC到AMS进程里了，会在AMS进程空间中执行 `attachApplicationLocked`函数，其中关键的三处是：
```java
// 首先从AMS本地维护的进程信息列表中得到对应的 ProcessRecord
// AMS管理着所有的进程信息，所以在start进程的时候就已经put进去啦
ProcessRecord app;
if (pid != MY_PID && pid >= 0) {
    synchronized (mPidsSelfLocked) {
        app = mPidsSelfLocked.get(pid);
    }
 } else {
    app = null;
}

boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
// 生成目标provider所在进程里的所有provider信息，应该是从PKMS请求
List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

// IPC回到目标进程空间去，传了一大堆参数，其中第3个就是我们的provider列表
thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
        profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
        app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
        enableTrackAllocation, isRestrictedBackupMode || !normalMode, app.persistent,
        new Configuration(mConfiguration), app.compat,
        getCommonServicesLocked(app.isolated),
        mCoreSettingsObserver.getCoreSettingsLocked());
```
- 首先AMS从本地维护的进程map `mPidsSelfLocked`中获取到目标进程的进程record; 
- 然后再来看`generateApplicationProvidersLocked`函数，注意这时候还是在`AMS`进程空间：
```java
private final List<ProviderInfo> generateApplicationProvidersLocked(ProcessRecord app) {
    List<ProviderInfo> providers = null;
	
	// 果然是从PKMS里得到provider的信息列表
    try {
        ParceledListSlice<ProviderInfo> slice = AppGlobals.getPackageManager().
            queryContentProviders(app.processName, app.uid,
                    STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS);
        providers = slice != null ? slice.getList() : null;
    } catch (RemoteException ex) {
    }

    int userId = app.userId;
    if (providers != null) {
        int N = providers.size();
		// 确保目标进程的record对应的provider数组能够放得下
        app.pubProviders.ensureCapacity(N + app.pubProviders.size());
        for (int i=0; i<N; i++) {
            ProviderInfo cpi =
                (ProviderInfo)providers.get(i);
			......

            ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
			// 这里很重要哦
			// 从AMS维护的map中根据class名取出对应的一个ContentProviderRecord
            ContentProviderRecord cpr = mProviderMap.getProviderByClass(comp, userId);
            if (cpr == null) {
               cpr = new ContentProviderRecord(this, cpi, app.info, comp, singleton);
                mProviderMap.putProviderByClass(comp, cpr);
            }

			// 把取出的ContentProviderRecord加入到目标进程在AMS里的ProcessRecord的pubProviders
			// 注意，java里这样put来，get去的都是引用哦，也就是都是同一个object实例。
            app.pubProviders.put(cpi.name, cpr);
            ......
        }
    }
    return providers;
}
```
`【这里做了一件很重要的事情】`，就是从AMS的本地维护的`mProviderMap`里通过provider类名唯一取出了一个`ContentProviderRecord`对象，还记得这个对象是什么时候放入的吗？对了，就是在 `调用者进程` 获取provider时，AMS启动`目标进程`之后，把一个provider record加入了 map中：
```java
    // 将providerrecord加入map管理，以类名
    if (firstClass) {
        mProviderMap.putProviderByClass(comp, cpr);
    }
                
    // 将providerrecord加入map管理，以authority
    mProviderMap.putProviderByName(name, cpr);
```
取出之后，放入了把取出的`ContentProviderRecord`加入到目标进程在AMS里的ProcessRecord的`pubProviders`，
```java
	app.pubProviders.put(cpi.name, cpr);
```
注意，java里这样put来，get去的都是引用哦，也就是都是同一个object实例，那么这里就开始有点端倪了，因为在上一节的最后，已经通过在这个对象上`cpr.wait()`阻塞在那啦，接下来可能快到`notify`的地方了，激动啊。。。
- slow down，我们先继续`attachApplicationLocked`的流程，上面的`generateApplicationProvidersLocked`返回后，AMS得到了目标进程的provider列表，并且在AMS里`ProcessRecord` map里进行了数据更新，在其中对`mProvidersMap`中的provider进行了关联。


- 然后AMS调用了`thread.bindApplication`，这里的thread，就是传过来的`ApplicationThread`代理，哇靠，果然又跨进程调用回去啦，而且还传了这么一大坨参数，威武。。。这样又重新回到了目标进程空间里的`bindApplication`了，其内部最终会通过`handleBindApplication`函数进行处理，我们只需要关注下面的关键代码：
```java
    if (!data.restrictedBackupMode) {
        List<ProviderInfo> providers = data.providers;
        if (providers != null) {
            installContentProviders(app, providers);
        }
	}
```
**终于看到它开始install它的provider了。**
```java
    private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
        final ArrayList<IActivityManager.ContentProviderHolder> results =
            new ArrayList<IActivityManager.ContentProviderHolder>();

        for (ProviderInfo cpi : providers) {

            IActivityManager.ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try {
            ActivityManagerNative.getDefault().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
        }
    }
```
主要干了下面几件事：
- 通过for循环，使用`installProvider`将本APP所拥有的所有`provider`都进行安装，其实就是实例化，注意这里的`installProvider`的调用的第二个参数，传的是`null`，前面有强调过；
- 通过跨进程调用AMS，将所有`installProvider`的结果发布给AMS，也就是在AMS里进行注册，这里会notify哦，后面会分析这个`publishContentProviders`函数。

> `原来provider的实例化在这里，也就是一个APP的所有的provider都是在它的进程起来的时候被一次性实例化的，而且都是实例化在它所在的进程空间，这好像是废话，哈哈哈~`

先看 `installProvider`，前面强调过的，这里第二个参数为null：
```java
    private IActivityManager.ContentProviderHolder installProvider(Context context,
            IActivityManager.ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
		// 如果为第二个参数为null，或者其内的IContentProvider无效
		// 那么就说明还没有实例化过，需要我们来实例化
        if (holder == null || holder.provider == null) {

            Context c = null;
            // 下面是获取context，为下面获取对应的classloader做准备
			......

            try {
                final java.lang.ClassLoader cl = c.getClassLoader();
				// 通过loadClass().newInstance()实例化Provider
                localProvider = (ContentProvider)cl.
                    loadClass(info.name).newInstance();
				// 每个Provider都是继承自ContentProvider的，而ContentProvider内部实现了
				// 跨进程的binder，又是binder，通过getIContentProvider()可以返回它的
				// binder客户端代理。
                provider = localProvider.getIContentProvider();

                localProvider.attachInfo(c, info);
            }......
        } else {
            provider = holder.provider;
        }

        IActivityManager.ContentProviderHolder retHolder;

        synchronized (mProviderMap) {

		// 接下来就是加入本地map管理，并包装到ContentProviderHolder中返回
            IBinder jBinder = provider.asBinder();
            if (localProvider != null) {
                ComponentName cname = new ComponentName(info.packageName, info.name);
                ProviderClientRecord pr = mLocalProvidersByName.get(cname);
                if (pr != null) {
                    provider = pr.mProvider;
                } else {
                    holder = new IActivityManager.ContentProviderHolder(info);
                    holder.provider = provider;
                    holder.noReleaseNeeded = true;
                    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }
                retHolder = pr.mHolder;
            } else {
        }

        return retHolder;
    }
```
具体看上面代码的注释，通过这个函数，即实例化了provider，也得到了provider对应的binder客户端代理，也就是`IContentProvider`对象，这就是别人需要的，拿到这个就可以直接跟这个进程的provider进行交互啦，而这里这些完成之后都还是放在自己进程内部进行维护；

所以，别人要想使用这个`IContentProvider`对象，该怎么办呢，答案就是接下来的`publishContentProviders`，它的参数就是上一步`installProvider`之后的所有`IContentProvider`对象集合(封装了一层holder)；
```java
ActivityManagerNative.getDefault().publishContentProviders(
                getApplicationThread(), results);
```
可以看到是跨进程调用，并把结果传了进去，到AMS里来看看：
```java
public final void publishContentProviders(IApplicationThread caller,
        List<ContentProviderHolder> providers) {
	......
    synchronized (this) {
		// 从AMS维护的ProcessRecord map中取出进程的record，
		// 这里的caller可是我们要访问的provider的所在的目标进程哦
        final ProcessRecord r = getRecordForAppLocked(caller);
		......

        final int N = providers.size();
        for (int i=0; i<N; i++) {
            ContentProviderHolder src = providers.get(i);
            ......
			
			// 激动的地方
			// 从对应ProcessRecord里的pubProviders把ContentProviderRecord取出来了
			// 这个跟上面目标进程cpr.wait()的是同一个哦
            ContentProviderRecord dst = r.pubProviders.get(src.info.name);
            if (dst != null) {
				// 将providerrecord加入map管理，这里可能会奇怪，前面不是
				// 已经put过了吗，怎么又要加入一遍呢，这是因为AMS维护ProviderMap
				// 是区分userid的，也就不同的进程会有不同的map，但对于同一个provider
				// 却都是指向同一个provider record的
                ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                mProviderMap.putProviderByClass(comp, dst);
                String names[] = dst.info.authority.split(";");
                for (int j = 0; j < names.length; j++) {
                    mProviderMap.putProviderByName(names[j], dst);
                }

				// 从待启动provider列表mLaunchingProviders中将provider删除，因为已经准备好了
                int NL = mLaunchingProviders.size();
                int j;
                for (j=0; j<NL; j++) {
                     if (mLaunchingProviders.get(j) == dst) {
                        mLaunchingProviders.remove(j);
                        j--;
                        NL--;
                    }
                }
				// 终于到了notify的地方了，对这个ContentProviderRecord对象更新provider值
				// 并最终notifyAll()，注意这里的dst跟之前的cpr.wait的地方是同一个对象，前面有说明
				// 至此，目标进程对provider的publish工作就结束了，而调用者进程也得以继续执行...
                synchronized (dst) {
                    dst.provider = src.provider;
                    dst.proc = r;
                    dst.notifyAll();
                }
				......
            }
        }
    }
}
```
终于到尾声啦！上面的publish函数主要做了下面几件事：
- 从AMS维护的进程map中把自己的`ProcessRecord`取出，并从中取出之前设置的provider record；
- 将provider record重新加入map管理，这时候是附加在与自己的进程信息相关联的map中；
- 从待启动provider列表 `mLaunchingProviders`中将该provider信息删除，因为已经准备OK，不用再告诉别人已经有人在启动这个provider了；
- 最后一步，真正将IContentProvider赋值，并`notifyAll()`，publish结束，调用者线程得到了想要的IContentProvider并继续执行。。。。

调用者线程拿到`IContentProvider`之后就能直接与目标线程跨进程调用啦~不用AMS再在中间搞来搞去啦。。。。

## 总结
过程还是比较复杂的，毕竟跨越了3个进程，代码执行调来调去，我们用下面一个图来描述这个过程，希望可以概括明白：

![enter image description here](http://dxjia.cn/wp-content/uploads/2015/11/contentprovider.gif)

图里精简了大部分流程，以求能简洁的指出`ContentProvider`完成跨进程调用的关键。

## Reference
[1] http://blog.csdn.net/singwhatiwanna/article/details/21829971
[2] http://blog.csdn.net/qinjuning/article/details/7310620
[3] http://blog.csdn.net/luoshengyang/article/details/6963418


