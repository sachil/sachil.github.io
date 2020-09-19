---
layout: post
title: Android基础-Activity的四种启动模式
mathjax: false
date: 2020-08-20 13:10:40
urlname: Android基础-Activity的四种启动模式
categories:
  - Android基础
tags:
 - Android基础
---

# Android基础-Activity的四种启动模式

## 四种启动模式

在Android中Activity有4种启动模式，现在总结一下，以便以后查阅使用。

1. standard模式，这是默认的启动模式，每次启动Activity都会创建一个新的Activity置于任务栈的栈顶，这是使用最多的启动模式。
2. singleTop模式，当使用该模式启动Activity时，如果在任务栈的栈顶正好存在该Activity的实例，则会调用该Activity的OnNewIntent()方法重用该Activity，而不会新创建该Activity，否则行为和standard相同。
3. singleTask模式，就像其名字一样，在同一个任务栈中，该Activity只能是以单例的形式存在。只要在任务栈中存在该Activity的实例，则会调用该Activity的OnNewInternet()方法重用该Activity，**在重用的同时，会移任务栈中该Activity上面的所有Activity实例，从而使该Activity置于任务栈的栈顶**。在Android官方文档中对于该模式的描述存在偏差(*或许我没有理解到官方文档的真实意思*)，singleTask模式在常规使用下，并不会新建一个任务栈并实例化根Activity，但是如果当taskAffinity不同时，则会新建任务栈并且实例化根Activity。同样的，如果当第三方app调起该Activity时(**任务栈中不存在该Activity的实例，并且taskAffinity不同时**)，也是如此。另外需要注意的是，当前台第三方app调起该Activity时，如果后台任务栈中已经存在该Activity的实例，则系统会将整个后台任务栈转到前台运行。
4. singleInstance模式，从名字中就可以看出，在整个系统中，该Activity都只是以单例的形式存在。**在一个新的任务栈中创建该Activity的实例**，系统不会将其他任何其他Activity起到到包含该实例的任务中。**该 Activity 始终是其任务唯一的成员(不论taskAffinity是否相同)**；由该 Activity 启动的任何 Activity 都会在其他的任务中打开。需要注意一下的是，在singleInstance模式下，如果存在任务栈task1(A->B->C)，C启动D(D为singleInstance模式)，则会创建任务栈task2(D)，然后D再启动C，这时任务栈task1为A->B->C->C，如果这时候按返回键，是不会返回到task2(D)的，而是任务栈task1回退(A->B->C)，只有当任务栈task1清空时再按返回，才会返回到任务栈task2(D)。

可以使用ADB命令:`adb shell dumpsys activity`或者`adb shell dumpsys activity activities`查看Activity任务栈的信息。

## 配置启动模式

配置启动模式有两种方式：

1. 在文件清单中配置，可以在manifest.xml中的activity标签中使用android:launchMode进行配置。这是大家经常使用的方式。

   ```xml
   <activity  android:launchMode="standard|singleTop|singleTask|singleIntance"/>
   ```

2. 使用Intent flags进行指定，在startActivity()时，指定Intent的flags进行配置。flags有3种：

   - [ ] Intent.FLAG_ACTIVITY_SINGLE_TOP，与使用singleTop行为相同
   - [ ] Intent.FLAG_ACTIVITY_NEW_TASK，在同一任务栈中单独使用时，看不到任何效果。当第三方app使用该flag调起Activity时(*taskAffinity不同*)，如果该Activity实例不存在，则会创建新的任务栈并实例化该Activity，如果该Activity实例存在，则会将包含该实例的任务栈转到前台运行，并且新建该Activity实例。该flag通常和 Intent.FLAG_ACTIVITY_CLEAR_TOP一起使用，
   - [ ] Intent.FLAG_ACTIVITY_CLEAR_TOP，单独使用并且没有指定任何启动方式(standard除外)时，如果要启动的Activity已经在任务栈中，则会销毁包括它自己以及它之上的所有其他Activity，并重新创建该Activity实例。如果搭配Intent.FLAG_ACTIVITY_SINGLE_TOP一起使用或者指定任一启动方式(standard除外)，则不会销毁它自己，而是通过`onNewIntent()`回调进行重用。

***注意事项:如果在使用中两种方式都有指定，则使用Intent flags的优先级更高。***

## 关于taskAffinity

taskAffinity表示"亲和性"，表示 Activity 倾向于属于哪个任务。默认情况下，同一应用中的所有 Activity 彼此具有亲和性。因此，在默认情况下，同一应用中的所有 Activity 都倾向于位于同一任务。您可以使用 `<activity>`元素的 `taskAffinity`属性修改任何给定 Activity 的亲和性。

 `taskAffinity`属性采用字符串值，该值必须不同于 `<manifest>`元素中声明的默认软件包名称，因为系统使用该名称来标识应用的默认任务亲和性。

taskAffinity可以在两种情况下发挥作用：

1. 当启动 Activity 的 intent 包含 `FLAG_ACTIVITY_NEW_TASK` 标记时。

   默认情况下，新 Activity 会启动到调用 `startActivity()` 的 Activity 的任务中。它会被推送到调用方 Activity 所在的返回堆栈中。但是，如果传递给 `startActivity()` 的 intent 包含 `FLAG_ACTIVITY_NEW_TASK` 标记，则系统会寻找其他任务来容纳新 Activity。通常会是一个新任务，但也可能不是。如果已存在与新 Activity 具有相同亲和性的现有任务，则会将 Activity 启动到该任务中。如果不存在，则会启动一个新任务。

2. 当 Activity 的 `allowTaskReparenting`属性设为 `"true"` 时。

   在这种情况下，一旦和 Activity 有亲和性的任务进入前台运行，Activity 就可从其启动的任务转移到该任务。

   举例来说，假设一款旅行应用中定义了一个报告特定城市天气状况的 Activity。该 Activity 与同一应用中的其他 Activity 具有相同的亲和性（默认应用亲和性），并通过此属性支持重新归属。当您的某个 Activity 启动该天气预报 Activity 时，该天气预报 Activity 最初会和您的 Activity 同属于一个任务。不过，当旅行应用的任务进入前台运行时，该天气预报 Activity 就会被重新分配给该任务并显示在其中。