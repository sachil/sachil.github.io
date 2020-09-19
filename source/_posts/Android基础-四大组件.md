---
layout: post
title: Android基础-四大组件
mathjax: false
date: 2020-09-01 19:05:09
urlname: Android基础-四大组件
categories:
  - Android基础
tags:
 - Android基础
---

# Android基础-四大组件

## Activity

Activity用于向用户展示内容并且接受用户操作的界面，实际上等同于app与用户之间进行交互的接口。

### Activity的生命周期

- onCreate()

  在这个方法中进行初始化操作，比如加载布局，绑定事件，初始化数据等。

- onRestart()

  在Activity由停止状态转变为运行状态前调用。

- onStart()

  当Activity又不可见变为可见时调用，虽然此时Activity未出现在前台，不能与用户进行交互，但是它已经是可见的了。

- onResume()

  Activity出现在前台，并且可以和用户进行交互。**此时Activity一定处于任务栈的栈顶，并且处于运行状态。**

- onPause()

  在系统准备启动或者恢复另一个Activity时调用。**我们通常在这个方法中将一些极其消耗CPU的资源释放掉，并且保存一些关键数据，但是速度一定要快，否则会影响新的栈顶Activity的使用。**

- onStop()

  在Activity**完全不可见时**调用。**与onPause()不同的是，如果启动的是一个对话框式的Activity或者背景透明的Activity，onPause()会被调用，而onStop()不会被调用。**

- onDestory()

  在Activity销毁前调用，该方法执行后，Activity即被销毁。

这几种方法，除了`onRestart()`，其他都是两两成对的，所以我们可以将Activity划分为3个时期：

- 完整生存期

  在`onCreate()`和`onDestory（）`方法之间所经历的，就是完整生存期。

- 可见生存期

  在`onStart（）`和`onStop()`方法之间所经历的，就是可见生存期。

- 前台生存期

  在`onResume()`和`onPause()`之间所经历的，就是前台生存期。

当Activity意外终止时，在`onStop()`方法调用之前，系统会调用`ononSaveInstanceState()`方法，用于保存Activity的状态，并在`onCreate()`方法调用后，在`onRestoreInstanceState()`方法中恢复状态。

## Service

service是一种可在后台执行长时间运行操作而不提供界面的应用组件,可以执行一些耗时的任务。service在其托管的进程的UI线程运行，默认情况下，它既不创建线程，也不在单独的进程中运行，如果执行一些CPU密集型操作或者阻止性操作，请在service中开启线程来进行这些操作，否则会出现ANR错误。

### 运行Service的方式

运行service有以下两种方式：

- **通过`startService()`启动**,一旦启动，Service则可以在后台无限期运行，即使启动它的组件已经销毁也不受影响。如果想停止service运行，可以通过`stopSelf()`方法自行停止运行，或者通过其他组件调用`stopService()`方法停止运行。并且在不需要Service工作时，记得将其停止，否则service将一直运行。

  `startService()`方法会调用Service的`onStartCommand()`方法，这个方法的返回值有特殊意义：

  - **`START_NOT_STICKY`**，当 Service 因内存不足而被系统 kill 后，即使系统内存再次空闲时，系统也不会尝试重新创建此 Service。这是最安全的选项。
  - **`START_STICKY`**，当 Service 因内存不足而被系统 kill 后，一段时间后内存再次空闲时，系统将会尝试重新创建此 Service，一旦创建成功后将回调 onStartCommand 方法，但其中的 Intent 将是 null，除非有挂起的 Intent。比较适合媒体播放器或类似的服务
  - **`START_REDELIVER_STICKY`**，当 Service 因内存不足而被系统 kill 后，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()，任何挂起 Intent 均依次传递。与 START_STICKY 不同的是，其中的传递的 Intent 将是非空，是最后一次调用 startService() 中的 intent。比较适合文件加载或类似需要恢复的服务。

- **通过`bindService()`绑定**,在其它组件中通过`bindService()`方法也可以运行Service，并且通过这种绑定方式，组件可以获取到Service实例，从而可以进行交互。`bindService()`方法会调用Service的`onBind()`回调，在回调中可以返回一个IBinder的实例，从而实现绑定。组件可以使用`unBindService()`方法解除与Service间的绑定，它会调用service的`onUnBind()`回调，需要注意的是：

  - **当第一个组件调用`bindService()`方法时，会调用`onBind()`回调，然后后面的组件继续绑定Service，如果系统判断到Intent相同，则不再调用`onBind()`回调。**
  - **当所有组件都与该Service解绑后，Service的`onUnBind()`回调才会执行，然后Service才会销毁。**
  - **`unBind()`回调可以返回`true`，然后再次绑定时会回调`onRebind()`方法（前提是该Service仍然存在）。**
  - **当有组件绑定到Service时，即使调用`stopService()`方法尝试停止Service也不会成功。**
  - **Service的`onCreate()`回调只会执行一次。**

### Service的生命周期

- 通过`startService()`方式运行，**启动Service->`onCreate()`->`onStartCommand`->Service正在运行->`onDestory()`->Service销毁**
- 通过`bindService`方式运行，**绑定Service->`onCreate()`->`onBInd()`->Service正在运行->`onUnbind()`->`onDestory()`->Service销毁**

### IntentService

IntentService是service的扩展类，它会为Service单独创建一个线程，并且创建一个工作队列，用于将intent逐一地传递给`onHandleIntent()`方法，所以它是串行的执行任务，不用担心多线程问题，它还会在执行完所有任务后自动地停止Service，不用手动调用停止的方法。需要注意的是：**IntentService执行传递给`onStartCommand()`的Intent，所以需要调用`startService()`使其运行。**

### ForegroundService

Foreground Service(前台服务)是用户主动意识到的一种Service，因此在内存不足时，系统也不会考虑将其终止。**Foreground Service必须为状态栏提供显示优先级为`PRIORTY_LOW`或更高的通知**，以帮助用户知道应用正在执行的任务，除非停止Service或者将Service转为后台，否则无法清楚通知。

**从Android 9开始，使用Foreground Service必须请求`FOREGROUND_SERVICE`权限**，启动Foreground Service可以使用`startForeground()`方法，**该方法的整型ID不能为0**，移除Foreground Service可以调用`stopForeground()`方法，**该方法不会停止Service，而是将其从前台转为后台运行。**

跨进程调用可以使用**`Messenger`**和**`AIDL`**方式。

## Broadcast

广播(broadcast)是一个全局的监听器，可以监听整个系统或者整个app，能用于跨进程或者进程内通信，其采用观察者模式，基于消息的订阅/发布模型。这其中有3个角色：1.广播发送者 ，2.消息中心(Activity Manager Service)，3.广播接受者(Broadcast Receiver)。

### Broadcast的注册

- 静态注册，静态注册是指在AndroidManifest.xml中进行注册。

  ```xml
    <receiver
              <!-- Android7.0开始，如果为false，手机重启后未解锁前，广播访问的数据是FBE加密的，将不能启动  -->
          android:directBootAware=["true" | "false"]
          android:enabled=["true" | "false"]
          android:exported=["true" | "false"]
          android:icon="drawable resource"
          android:label="string resource"
          android:name=".MBroadcastReceiver"
          android:permission="string"
          android:process="string" >
        <intent-filter android:icon="drawable resource"
                  android:label="string resource"
                  android:priority="integer">
              <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
        </intent-filter>
    </receiver>
  ```

- 动态注册，动态注册是指在代码中通过`registerReceiver()`方法注册广播，如果是动态注册，则需要在组件的相应生命周期内使用`unregisterReceiver()`取消广播的注册。

### Broadcast的类型

按照不同的划分方法，可以将Broadcast划分为不同的类型。

- **自定义广播和系统广播**

- **普通广播和有序广播**

  普通广播以异步的方式发送给接收者，所有满足条件的接收者几乎可以同时收到广播，没有顺序之分，也不用想AMS返回处理结果。各接收者互不干扰，效率较高。

  有序广播是以一种同步的方式向接收者发送广播的。广播接收者会根据开发者设定的优先级进行排序，AMS会先给最高优先级接收者发送广播，该接收器处理完广播后需要返回处理结果给AMS，然后再向次优先级接收者发送广播，依此类推。广播接收者可以把处理结果传递给是下一个接收者，也可以终止广播的传递。如果某个接收者超时或者终止了广播，那么后面的接收者也将无法再收到广播，所以效率会比较低。 有序广播对比普通广播，有几个关键的地方需要注意：

  1. `android:priority`用于设置接收器优先级。
  2. `getResultData()`获取广播数据。
  3. `setResultData()`向下一接收者传递数据。
  4. `abortBroadcast()`用于终止广播的传递。

- **全局广播和局部广播**，从机制上看，如果发送广播的可以被其他app（以app进程为限）接收，那么该广播为全局广播；如果只能被app内部接收，那么该广播为局部广播。

  全局广播在安全和性能方面存在一些问题，无疑增加了系统的负担，为了解决这方面的问题，可以通过设置“android:exported”，permission，setPackage等各种方式来限定接收范围，从而在安全和性能方面进行优化。

  局部广播，也叫做本地广播，Android v4 兼容包提供`android.support.v4.content.LocalBroadcastManager`工具类用于实现局部广播。局部广播有以下3点优势：

  - **你所传播的数据不会离开当前你的app，所以无需担心会泄漏私人数据。**
  - **其他app无法发送广播到你的app，所以你无需担心这些广播导致的安全漏洞。**
  - **比起通过系统来实现发送的全局广播，这种方式更高效（不需要发送给整个系统）。**

  局部广播相对于全局广播来说更加高效的原因是，其内部实现并不是采用Binder机制，而是采用Handler机制，发送局部广播，其实就是向Handler发送了一个Message，但是其需要注意的地方有:

  - **只能通过动态注册。**
  - **一定不要忘记前面提到的三个方法前加上LocalBroadcastManager.getInstance(mContext)，否则可能接收不到广播或者无法实现局部广播的效果。**
- **前台广播和后台广播**，根据发送的广播被接收的优先级，可以将广播分为前台广播和后台广播。

  前台广播是为了减少广播接收的延时，添加`Intent.FLAG_RECEIVER_FOREGROUND`这个flag，可以将广播设置为前台广播。前台广播的ANR时间限制是10s。未设置`Intent.FLAG_RECEIVER_FOREGROUND`flag的广播则是后台广播，后台广播的ANR时间限制是60s。

- **并行广播和串行广播**，根据接收和处理广播事件的方式是并行的还是串行的，可以把广播分为并行广播和串行广播。

  并行广播，在代码中注册的普通广播。

  串行广播，在文件清单中注册的普通广播，和有序广播。

- **粘性广播和非粘性广播**(Android5.0开始已经废弃掉)













### Broadcast Receiver的生命周期

广播接收器自然也有自己的生命周期。只是它的生命周期非常简单且非常短，只有onReceive一个回调方法，当收到广播产生onReceive回调开始，到onReceive方法执行完并return，广播接收器的生命周期便宣告结束。**Broadcast Receiver默认运行在UI线程，所以不要运行耗时的操作，也不要在receiver中开启线程进行耗时操作，因为当onReceive()返回时，Broadcast Receiver的生命周期就已经结束了，系统随时可能进行回收，线程可能无法按照预期运行。解决办法是可以使用：jobService，goAsync，startService(最常使用)。**

### onReceive()方法的上下文

广播接收器的Context：

- 局部广播：applicationContext实例
- 静态注册的广播：ReceiverRestrictedContext实例
- 动态注册的广播：组件(Activity,Service)的context实例

### 不同Android版本广播机制的重要变迁

- Android5.0，废弃掉粘性广播
- Android7.0，不在发送`ACTION_NEW_PICTURE`和`ACTION_NEW_VIDEO`系统广播，同时`CONNECTIVITY_ACTION`只能动态注册才能接收到。
- Android8.0，系统对清单声明的接收器施加了额外的限制，对于大多数隐式广播（没有明确针对您的应用的广播），您不能使用静态注册来声明接收器。
- Android9.0，`NEWWORK_STATE_CHANGED_ACTION`广播不再接收有关用户位置或个人身份数据的信息，此外，通过 WLAN 接收的系统广播不包含 SSID、BSSID、连接信息或扫描结果。

## Content Provider

