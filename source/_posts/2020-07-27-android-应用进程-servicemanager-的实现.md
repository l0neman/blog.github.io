---
title: Android 应用进程 ServiceManager 的实现
toc: true
date: 2020-07-27 00:44:50
categories:
- Android 应用开发
tags:
- Android
- Binder
---
实现一个普通应用进程中的 ServiceManager，可自由注册和获取 Binder 服务。

文末给出开源仓库地址。



Binder 相关基础可参考：[android-binder-设计分析](https://l0neman.github.io/2019/04/08/android-binder-设计分析/)

# 实名 Binder 与匿名 Binder

## 实名 Binder

在 Binder 通信模型中，存在一个 ServiceManager 的角色，它作为 Android 系统的服务总管，负责建立 Binder 名字和 Binder 实体的映射。

提示：ServiceManager 中的 Service 和 `android.app.Service` 组件不是同一个概念。
<!-- more -->
ServiceManager 中存在一个 `addService` 方法，使用它可将一个 Binder 实体通过一个唯一的字符串标识注册到 Binder 驱动中，同时 ServiceManager 会缓存 Binder 的引用，下次通过一个字符串标识来请求 ServiceManager 即可获得对应的 Binder 实体的引用。

Android Framework 层中的系统服务，都是通过 Java 层 ServiceManager 的 `addService` 方法将自己注册为服务的，例如著名的 `ActivityManagerService` 和 `WindowManagerService`：

```java
// AOSP 9.0.0_r3: android.server.am.ActivityManagerService.java

public void setSystemProcess() {
  ...
  ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
         DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
  ...
}
```

```java
// AOSP 9.0.0_r3: android.server.SystemServer.java

private void startOtherServices() {
  ...
  wm = WindowManagerService.main(context, inputManager,
         mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
         !mFirstBoot, mOnlyCore, new PhoneWindowManager());
  ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
         DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
  ...
}
```

当注册完成后，就可以通过字符串标识 `Context.ACTIVITY_SERVICE` 和 `Context.WINDOW_SERVICE` 来通过 ServiceManager 的 `getService` 方法获取相应 Binder 服务端的引用，对系统服务发起请求了。

像这样使用字符串标识明确的向 ServiceManager 注册的 Binder 实体，被称为实名 Binder。

提示：在 Android 系统中，四大组件的启动均是通过 `ActivityManagerProxy`（Binder 客户端）向 `ActivityManagerService`（Binder 服务端）发送进程间通信请求实现的。



## 匿名 Binder

还存在一种匿名 Binder，匿名 Binder 的创建依赖于一条已经建立的 Binder 连接，通过已建立的 Binder 连接，将 Binder 服务端的引用从服务端进程传递至另一端，此时另一端持有 Binder 引用就可以通过 Binder 驱动与服务端进行沟通了。

匿名 Binder 不通过明确的字符串标识进行注册，而是通过私有 Binder 连接通道传递 Binder 引用，所以匿名 Binder 无法通过枚举或者猜测得到。

除了 ServiceManager 自身通过 0 号引用注册的 Binder 连接（仅服务于实名 Binder 的注册），其他的就只有系统服务注册的实名 Binder 连接了。

所以匿名 Binder 的创建必须通过实名 Binder 来实现。



# 需求分析

如果现在要实现自己的 Binder 服务，服务存在于独立的进程中，且客户端可以获取服务端的 Binder 引用，自由地向服务端发出任务请求，要怎么办呢。

还要考虑动态注册 Binder 服务的需求，现实情况下，一个 Binder 服务可能无法满足业务需求，所以可以考虑以字符串标识对 Binder 服务端进行注册，那么客户端可以利用字符串标识查询对应的 Binder 服务端（类似于 Android 系统的 ServiceManager）。



那么需要分两部分考虑：

1. 实现的 Binder 服务需要在一个进程中运行，在 Android 中四大组件均可在独立进程中运行，首先 Activity 不能在后台运行，所以 Binder 服务不能在 Activity 中运行，BroadcastReceiver 只能执行短时间的任务，所以也不能放置 Binder 服务，那么放在 Service 和 ContentProvider 是可行的。

2. 如果传递 Binder 服务的引用给客户端，客户端持有 Binder 引用后即可向 Binder 服务端发起请求，那么有如下分析：

首先我们的应用处于 Android 普通应用进程中，并没有系统权限，所以不可能通过 Android 系统的 ServiceManager 注册自己的 Binder 服务，那么直接注册成为实名 Binder 服务的方法就不能采用了。

那么现在只能考虑通过匿名 Binder 来实现，先找到一个已建立的实名 Binder 连接，使用这个连接来传递自己的 Binder 服务端的引用给客户端使用即可。

Android 系统服务都是实名 Binder 连接，但是它们提供的接口功能都是固定的，肯定不能自由的被利用，所以只能先找到一个与匿名 Binder 相关的 API。同时为了实现 Binder 服务端，所以还要考虑后台运行的支持。

系统中的 Bundle 可以携带 Binder（使用 `putBinder` 方法），即 Intent 也能携带 Binder（Intent 可携带 Bundle）；还有 Service 组件的 `onBind` 方法，在客户端绑定时可以返回 Binder 引用。



那么基于这个条件就可以让的 Service 和 ContentProvider 组件具有了传送匿名 Binder 的功能，如下：

1. 对于 Service 组件，开发者可实现一个远程进程中的 Service，然后实现它的 `onBind` 方法，返回一个 Binder 对象，这个 Binder 对象的引用将会传递给客户端，客户端（例如 Activity）使用 `bindService` 方法，通过 `ServiceConnection` 中的 `onServiceConnected` 回调获取这个 Binder 引用后，即可向 Service 中的 Binder 对象发送处理相关任务的请求，即通过 `bindService` 可以获得一个匿名 Binder；同时也可以在启动 Service 时，通过 `Intent` 携带一个 `Bundle` 对象，`Bundle` 对象中携带一个 Binder 对象进行传递，即在启动服务时可通过 `Intent` 传递 Binder 对象。

2. 对于 ContentProvider 组件，开发者在实现了 `ContentProvider` 之后，可以在客户端使用 `context.getContentResolver()` 获取一个 `ContentResolver` 对象，利用这个 `ContentResolver` 对象向 `ContentProvider` 发出增删查改请求（`query`、`delete`、`insert`、`update`），还可以通过 `call` 方法发出自定义命令，其中 `call` 方法最后一个参数可以携带一个 `Bundle` 对象，这个 `Bundle` 允许携带 `Binder` 对象，即通过 `call` 方法可传递 Binder；同时 `call` 方法返回一个 `Bundle` 对象，即 ContentProvider 端也可直接返回 Binder 对象。

即 Service 和 ContentProvider 可以实现独立的 Binder 服务，或者有能力对 Binder 服务进行统一的管理。



分析了需求和实现方法，下面确定一下具体实施方案。



# 实现方案

要通过匿名 Binder 实现一个 Binder 服务，那么首先需要实现一个 Binder 服务端对象，将其放置在远程进程中，再通过匿名 Binder 通道，将 Binder 服务端引用传递给客户端，此时客户端使用 Binder 服务端的引用即可向服务端发起请求，这时就完成了一个 Binder 服务的建立；

如果要实现 Binder 服务的动态注册，Binder 服务可能分布在各个进程中，如果需要统一管理，需要首先建立一个类似于系统 ServiceManager 的角色，ServiceManager 运行在一个独立的进程，Binder 服务可通过进程间通信的方法将自己的 Binder 引用注册到 ServiceManager 中，客户端可以通过 ServiceManager 自由获取 Binder 服务端的引用。



现在有两种办法可以实现匿名 Binder，需要采用哪一种方案实现 Binder 服务呢。



## Service 方案

如果采用 Service 的方案实现一个 Binder 服务，首先需要实现一个 Service，然后在清单文件中配置它，然后实现一个自己的 Binder 服务端，放在这个远程 Service 中，客户端使用 `bindService` 获取 Binder 服务端的 Binder 引用即可，不过这样的话看起来和直接使用 Service 组件没有任何区别，属于脱裤子放屁；

如果实现 Binder 服务端的动态注册，将需要使用到的 Binder 服务端对象动态传递至 Service 中，然后客户端再想办法查询 Binder 服务端的引用。那么考虑借助 Intent 对象，利用 `Context` 的 `startService` 或 `bindService` 方法动态传递 Binder 对象至 Service 中，在 Intent 中携带字符串标识，然后在 Service 中建立一个用字符串作为标识的 `Map<String, IBinder>` 结构缓存这些 Binder 服务端，当客户端获取时，首先需要使用 `bindService` 方法，绑定 Service，然后 Service 返回一个 Binder 引用，不过这个 Binder 引用并不能是客户端想要获取的 Binder 服务端引用，因为每次绑定 Service，它都始终返回同一个 Binder，就是 Service 的 `onBinder` 方法第一次返回的那个 Binder，所以这个 Binder 需要作为一个中转的 Binder 服务端，需要先使用 `AIDL` 定义一个服务获取客户端想要的 Binder 服务端的方法，例如：

```java
// IBridge.aidl

interface IBridge {
  IBinder getBinder(String name);
}
```

那么客户端通过获取这个 `IBridge` 对象的 Binder 引用之后，再调用 `getBinder` 查询 Service 中 `Map<String, IBinder>` 缓存的 Binder 服务端引用即可。

这是使用 Service 组件的方案，看起来挺麻烦，下面再说使用 ContentProvider 组件的实现方案。



## ContentProvider 方案

如果采用 ContentProvider 的方案实现一个 Binder 服务，那么首先需要实现一个 ContentProvider，然后也需要清单文件中配置它，然后实现一个自己的 Binder 服务端，放在这个远程进程的 ContentProvider 中，客户端使用 `call` 获取返回的 Bundle 数据包，然后读取出其中携带的 Binder 服务端的 Binder 引用即可，ContentProvider 的本意是为了共享数据，但这样一来就被赋予了新的功能，用来做 Binder 服务端的支持。不过相对于使用 Service 实现一个 Binder 服务来说，也没有体现出太多优势；

如果实现 Binder 服务端的动态注册，将需要使用到的 Binder 服务端对象动态传递至 ContentProvider 中，然后客户端再想办法查询 Binder 服务端的引用。那么可以直接使用 `call` 方法将字符串标识和通过 Bundle 数据包将服务端 Binder 发送至 ContentProvider 中，ContentProvider 使用一个 `Map<String, IBinder>` 缓存即可，下次客户端可以直接使用字符串标识通过 `call` 方法请求获取对应的 Binder 服务端，那么 ContentProvider 直接返回携带对应 Binder 服务端的引用即可，这样来看就比 Service 实现动态注册 Binder 服务有优势的多。



## 最终结论

通过对两种方案的描述，谁更有优势显而易见。

对于 Service 来说，它还存在诸多限制，例如 `bindService` 绑定存在限制，BroadcastReceiver 组件无法绑定、绑定后必须取消绑定、或者服务不允许在后台运行，除非加上前台通知才允许后台运行，即使抛开这些限制，使用它实现 Binder 服务也是很麻烦的（需要借助 `ServiceConnectation`）。

反观 ContentProvider 均不存在上述限制，无需绑定，直接使用 `call` 请求即可，API 足够简洁，看起来可以完美的实现 Binder 服务。



# 实现

根据上面的方实现案分析（最终确定了使用 ContentProvider 组件实现），下面来真正实现 Android 应用进程中的 Binder 服务。



## 一个 Binder 服务

首先定义一个 Binder 服务的 AIDL 协议文件：

```java
// IFoo.aidl
package io.l0neman.example;

interface IFoo {
    int add(int x, int y);
}
```

它是一个叫 `IFoo` 的 Binder 服务，它内部有一个负责两数相加的 `add` 方法

然后实现一个 ContentProvider，叫做 `ServiceProvider`，然后直接将 IFoo 服务内置进去。

```java
public class ServiceProvider extends ContentProvider {

  public static final String ACTION_GET_SERVICE = "act.ser";
  public static final String KEY_SERVER = "ser";

  public static final String URI = "content://io.l0neman.example.binder";

  private FooService mFooService = new FooService();

  // Binder 服务端的实现
  private static final class FooService extends IFoo.Stub {
    @Override public int add(int x, int y) {
      return x + y;
    }
  }

  @Nullable @Override
  public Bundle call(@NonNull String method, @Nullable String arg, @Nullable Bundle extras) {
    if (ACTION_GET_SERVICE.equals(method)) {
      // 获取 Service 则返回携带 FooService 的 Bundle
      Bundle result = new Bundle();
      BundleCompat.putBinder(result, KEY_SERVER, mFooService);
      return result;
    }

    return null;
  }

  @Override
  public int delete(Uri uri, String selection, String[] selectionArgs) {
    throw new UnsupportedOperationException("Not yet implemented");
  }

  @Override
  public String getType(Uri uri) {
    throw new UnsupportedOperationException("Not yet implemented");
  }

  @Override
  public Uri insert(Uri uri, ContentValues values) {
    throw new UnsupportedOperationException("Not yet implemented");
  }

  @Override
  public boolean onCreate() { return true; }

  @Override
  public Cursor query(Uri uri, String[] projection, String selection,
                      String[] selectionArgs, String sortOrder) {
    throw new UnsupportedOperationException("Not yet implemented");
  }

  @Override
  public int update(Uri uri, ContentValues values, String selection,
                    String[] selectionArgs) {
    throw new UnsupportedOperationException("Not yet implemented");
  }
}
```

清单文件：

```xml
<provider
  android:name="io.l0neman.example.ServiceProvider"
  android:exported="false"
  android:authorities="io.l0neman.example.binder"
  android:enabled="true" />
```

很简单的代码，当任何客户端对 `ServiceProvider` 发出 `call` 请求，且 `method` 参数为 `ACTION_GET_SERVICE` 时，就返回一个携带 IFoo 服务的 Binder 对象。

此时客户端即可使用 `call` 获取服务端的 Binder 向服务端请求了。

```java
// MainActivity.java

@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  setContentView(R.layout.activity_main);

  final Bundle result = getContentResolver().call(Uri.parse(ServiceProvider.URI), 
    ServiceProvider.ACTION_GET_SERVICE, null, null);
  if (result != null) {
    IBinder ref = BundleCompat.getBinder(result, ServiceProvider.KEY_SERVER);
    // 获取服务端 Binder
    IFoo foo = IFoo.Stub.asInterface(ref);
    if (foo != null) {
      try {
        // 调用服务端方法
        final int add = foo.add(1, 2);
        Log.d(TAG, "add: " + add);
      } catch (RemoteException ignore) {}
    }
  }
}
```

执行即可看到结果 3。



提示：这里有一个细节是，ServiceProvider 与应用主进程相同，此时里面的 `IFoo` Binder 服务端也和它处于同一进程，那么此时通过 Bundle 返回的 ref 并不是 Binder 引用，而是直接返回 ServiceProvider 的 `mFooService` 对象（Binder 类表示），如果 ServiceProvider 在另一个进程，那么通过 Bundle 返回的将是 Binder 引用（BinderProxy 类表示）。

```java
// IFoo - IFoo$Stub

...
public static io.l0neman.example.IFoo asInterface(android.os.IBinder obj) {
  if (obj == null) { return null;  }
  // 查询 Binder 是否在本地
  android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
  // 如果 obj 对象是 Binder 服务端对象，iin 将为 obj 自己，如果 obj 是 Binder 引用，则为 null
  if (iin != null && iin instanceof io.l0neman.example.IFoo) {
    // 本进程直接则直接返回 Binder 对象
    return (io.l0neman.example.IFoo) iin;
  }
  
  // 否则返回 Binder 的引用，通过 Proxy 提供的方法，可向 Binder 服务端发送请求
  return new io.l0neman.example.IFoo.Stub.Proxy(obj);
}
```

```java
// Binder.java

...
public IInterface queryLocalInterface(String descriptor) {
  if (mDescriptor.equals(descriptor)) {
    return mOwner;
  }
  return null;
}
```

```java
// Binder.java - class BinderProxy

...
public IInterface queryLocalInterface(String descriptor) {
  return null;
}
```



## 动态注册 Binder 服务

动态注册服务，就是可以自由的注册 Binder 服务，注册后，客户端可以依据标识获取对应功能的服务端 Binder 的引用，然后向对应 Binder 服务发起任务请求。

按照前面的想法，先在 ContentProvider 中创建一个 `Map<String, IBinder>`，当 Binder 服务注册时，使用 `call` 方法将 Binder 服务端对象传递过来，此时 ContentProvider 接收到 Binder 服务端对象的引用，使用 `Map` 缓存起来，下一次，客户端再使用 `call` 方法传入字符串标识，ContentProvider 接收到字符串标识从 `Map` 中取出 Binder 端服务对象的引用，返回给客户端。

一般方法是在 ContentProvider 的 `call` 方法中，实现多个 `method` 参数的处理，例如使用 `switch`。考虑到以后还可能实现取消注册 Binder 服务等方法，那么每次都需要在 `switch` 中添加分支处理，这样的话看起来太麻烦了，而且可读性极差，其实可以直接采用面向对象的方式来完成。

首先，先实现一个叫做 `ServiceManager` 的 Binder 服务，这个 Binder 服务将直接内置在 ContentProvider 中，负责管理其他 Binder 服务的注册、获取、和取消注册等一系列事务。

类似于 Android 系统的 ServiceManager，系统的 ServiceManager 是直接在 Android 系统启动时使用 0 号引用注册到驱动中的，那么我们的应用进程中的 ServiceManager 就直接内置在一个 ContentProvider 中，当 ContentProvider 启动，ServiceManager 也就可以使用了。

首先定义 ServiceManager 的 AIDL 协议文件：

```java
// IServiceManager.aidl

package io.l0neman.app;

interface IServiceManager {
    // 注册服务
    void addService(in String name, in IBinder binder);
    // 获取服务
    IBinder getService(in String name);
    // 移除服务
    void removeService(in String name);
}
```

然后实现 ServiceManager 的具体逻辑：

```java
// io.l0neman.app

public class ServiceManagerImpl extends IServiceManager.Stub {

  private static final String TAG = ServiceManagerImpl.class.getSimpleName();

  // 保存正在活动的 Binder 对象
  private Map<String, ServiceEntry> mAliveServices = new ArrayMap<>();
  // 保存已经死亡的 Binder 对象
  private Map<String, ServiceEntry> mDiedServices = new ArrayMap<>();

 @Override public void addService(final String name, IBinder binder) throws RemoteException {
    if (mAliveServices.containsKey(name)) {
      // 不允许重复注册
      ServiceEntry item = mAliveServices.get(name);
      throw new RuntimeException(String.format("Service %s has registed by pid: %s, uid: %s", name,
          item.callingPid, item.callingUid));
    }

    IBinder.DeathRecipient recipient = new IBinder.DeathRecipient() {
      @Override public void binderDied() {
        ServiceEntry entry = mAliveServices.remove(name);
        if (entry != null) {
          // 保存死亡的 Binder 对象
          mDiedServices.put(name, entry);
        }
      }
    };

    // 保存 Binder
    ServiceEntry entry = new ServiceEntry(name, Binder.getCallingPid(), Binder.getCallingUid(), binder, recipient);
    mAliveServices.put(name, entry);
    mDiedServices.remove(name);
    // Binder 的死亡通知
    entry.linkToDeath();

    Log.d(TAG, "#addService: " + name);
  }

  @Override public IBinder getService(String name) {
    ServiceEntry entry = mAliveServices.get(name);
    if (entry == null) {
      Log.d(TAG, "#getService form died: " + name);
      entry = mDiedServices.get(name);
    }

    if (entry != null) {
      return entry.binder;
    }

    Log.e(TAG, "#getService: " + name);
    return null;
  }

  @Override public void removeService(String name) throws RemoteException {
    ServiceEntry entry = mAliveServices.get(name);
    if (entry != null) {
      // 取消注册 Binder 死亡通知
      entry.unlinkToDeath();
    }

    mDiedServices.remove(name);
  }
}
```

逻辑并不难理解，其中 ServiceEntry 是为了保存 Binder 服务端的相关信息的类型，可以利用这些信息做权限控制，例如限制指定的客户端进程 ID 和客户端 uid 才能>使用，代码如下：

```java
private static final class ServiceEntry {
  final String name;     // Binder 服务端字符串标识
  final int callingPid;  // 客户端的进程 ID
  final int callingUid;  // 客户端的用户 ID
  final IBinder binder;  // Binder 服务端对象

  IBinder.DeathRecipient recipient;

  ServiceEntry(String name, int callingPid, int callingUid, IBinder binder, 
               IBinder.DeathRecipient recipient) {
    this.name = name;
    this.callingPid = callingPid;
    this.callingUid = callingUid;
    this.binder = binder;
    this.recipient = recipient;
  }

  void linkToDeath() throws RemoteException {
    if (binder != null && recipient != null) {
      binder.linkToDeath(recipient, 0);
    }
  }

  void unlinkToDeath() {
    if (binder != null && recipient != null) {
      binder.unlinkToDeath(recipient, 0);
    }
  }
}
```

然后创建一个 `ServiceManagerProvider`，将 `ServiceManagerImpl` 内置进去。

```java
public class ServiceManagerProvider extends ContentProvider {
  private static final String TAG = ServiceManagerProvider.class.getSimpleName();

  public static final String URI = "content://io.l0neman.app.provider";
  public static final String KEY_SM = "binder";
  public static final String METHOD_SM = "$";

  private ServiceManagerImpl mServiceManager = new ServiceManagerImpl();

  @Override public boolean onCreate() {
    Log.i(TAG, "#onCreate");
    return true;
  }

  @Override
  public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs,
                      String sortOrder) { return null; }

  @Override public String getType(Uri uri) { return null; }

  @Override public Uri insert(Uri uri, ContentValues values) { return null; }

  @Override public int delete(Uri uri, String selection, String[] selectionArgs) { return 0; }

  @Override
  public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) { return 0; }

  @Override public Bundle call(String method, String arg, Bundle extras) {
    if (METHOD_SM.equals(method)) {
      Bundle service = new Bundle();
      BundleCompat.putBinder(service, KEY_SM, mServiceManager);
      return service;
    }

    return null;
  }
}
```

清单配置：

```xml
<provider
  android:name="io.l0neman.app.ServiceManagerProvider"
  android:authorities="io.l0neman.app.provider"
  android:exported="false"
  android:process=":sm"
  android:grantUriPermissions="true" />
```

然后实现 ServiceManager 的客户端封装，作为 API 提供给服务端注册自身 Binder 对象或客户端获取 Binder 服务端引用。

```java
public class ServiceManager {

  private static final String TAG = ServiceManager.class.getSimpleName();

  private ServiceManager() { throw new AssertionError("no instance."); }

  private static IServiceManager sIServiceManager;

  // 获取 ServiceManager 的 Binder 引用（实则为 Binder 引用的封装，Proxy 对象）
  private static void ensureIServiceManager(final Context context) {
    Bundle bundle = context.getContentResolver().call(
        Uri.parse(ServiceManagerProvider.URI),
        ServiceManagerProvider.METHOD_SM, null, new Bundle());

    if (bundle == null) {
      throw new RuntimeException("Service Manager is null [Bundle]");
    }

    IBinder serviceManager = BundleCompat.getBinder(bundle, ServiceManagerProvider.KEY_SM);

    if (serviceManager == null) {
      throw new RuntimeException("Service Manager is null [getBinder]");
    }

    try {
      serviceManager.linkToDeath(new IBinder.DeathRecipient() {
        @Override public void binderDied() {
          // Binder 死亡则尝试重新获取
          ensureIServiceManager(context);
        }
      }, 0);
    } catch (RemoteException e) {
      Log.w(TAG, "ensureIServiceManager: linkToDeath", e);
    }

    sIServiceManager = IServiceManager.Stub.asInterface(serviceManager);
  }

  private static IServiceManager getIServiceManager(Context context) {
    ensureIServiceManager(context);
    return sIServiceManager;
  }

  public static void addService(Context context, String name, IBinder binder) {
    try {
      getIServiceManager(context).addService(name, binder);
    } catch (RemoteException e) {
      Log.w(TAG, "addService", e);
    }
  }

  public static IBinder getService(Context context, String name) {
    try {
      return getIServiceManager(context).getService(name);
    } catch (RemoteException e) {
      Log.w(TAG, "#getService", e);
      return null;
    }
  }

  public static void removeService(Context context, String name) {
    try {
      getIServiceManager(context).removeService(name);
    } catch (RemoteException e) {
      Log.w(TAG, "#removeService", e);
    }
  }
}
```

很简单的封装，下面直接测试就好了。



# 测试

首先定义两个服务 `foo` 和 `bar` 的 AIDL 接口，它们分别负责相加运算和相减运算。

```java
// IFoo.aidl

package io.l0neman.example;

interface IFoo {
    int add(int x, int y);
}
```

```java
// IBar.aidl

package io.l0neman.example;

interface IBar {
    int sub(int x, int y);
}
```

然后在 `MainActivity` 的 `onCreate` 方法中直接注册两个 Binder 服务。

在点击 `test` 按钮时分别获取两个服务的 Binder 引用，请求服务执行，获取结果。

```java
public class MainActivity extends AppCompatActivity {

  private static final String TAG = MainActivity.class.getSimpleName();

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    ServiceManager.addService(this, "foo", new IFoo.Stub() {
      @Override public int add(int x, int y) {
        return x + y;
      }
    });

    ServiceManager.addService(this, "bar", new IBar.Stub() {
      @Override public int sub(int x, int y) {
        return x - y;
      }
    });
  }

  // 点击测试按钮
  public void test(View view) {
    IFoo iFoo = IFoo.Stub.asInterface(ServiceManager.getService(this, "foo"));
    if (iFoo != null) {
      try {
        final int add = iFoo.add(1, 2);
        Log.d(TAG, "add: " + add);
      } catch (RemoteException ignore) {}
    }

    IBar iBar = IBar.Stub.asInterface(ServiceManager.getService(this, "bar"));
    if (iBar != null) {
      try {
        final int sub = iBar.sub(3, 1);
        Log.d(TAG, "sub: " + sub);
      } catch (RemoteException ignore) {}
    }
  }
}
```

代码也很简单，最终输出结果为 `add: 3` 和 `sub: 2`。

提示：

`ServiceManagerProvider` 运行在一个独立进程中，和 `MainActivity` 处于不同的进程中，那么 `MainActivity` 中的两个 Binder 服务端对象通过 Bundle 传递给 `ServiceManagerProvider` 时将变成一个 Binder 引用，此时 `ServiceManagerImpl` 可对这些 Binder 引用进行统一管理。对于另一个进程中的 ContentProvider，主进程对于 `call` 方法的调用，将穿过 `ActivityManagerProxy` 与 `ActivityManagerService` 建立的匿名 Binder 通道传递 Bundle 数据，此时在 Bundle 中携带的 Binder 对象将被自动转换为 Binder 引用，至于转化过程的细节，在此不再赘述，后面再分析。

下面 `test` 方法中的获取的 Binder 对象直接就是 `MainActivity` 中的两个 Binder 服务端对象并非引用，因为 Binder 服务端和 `test` 方法在同一个进程中，所以 `ServiceManager.getService` 返回的对象是 Binder 服务端对象（Binder 类表示），那么 `asInterface` 中的 `queryLocalInterface` 将返回 Binder 服务端对象自身。

当客户端在其他进程调用 `ServiceManager.getService` 获取 Binder 服务端引用，与 Binder 服务端进程不在同一个进程中时，那么客户端获取的就是服务端的 Binder 引用（BinderProxy 类表示）。



这样就实现了一个普通应用进程中的 ServiceManager，通过统一管理匿名 Binder 的方式，实现了一种伪实名 Binder。



# 开源仓库

[https://github.com/l0neman/AppServiceManager](https://github.com/l0neman/AppServiceManager)

应用场景：只要是涉及两个进程间的通信，都可以用到，例如插件端和宿主端的通信、或实现一个主控制端服务，根据请求执行不同的任务。



# 参考

- [https://github.com/cmzy/DroidService](https://github.com/cmzy/DroidService)
