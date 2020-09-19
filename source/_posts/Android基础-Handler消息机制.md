---
layout: post
title: Android基础-Handler消息机制
mathjax: false
date: 2020-09-14 17:15:45
urlname: Android基础-Handler消息机制
categories:
  - Android基础
tags:
 - Android基础
---

# Android基础-Handler消息机制

## 设计Handler的主要目的

Handler的主要用途有两点：

- **规划`Message`或者`Runnable`在某个时间点执行**
- **在另外一个线程上执行代码**

众所周知，Android默认是运行在主线程，即UI线程，但是我们不能将耗时的任务放在UI线程中执行，否则会造成ANR错误，所以我们会新建一个线程，将耗时的任务交由这个线程去完成。但是当任务完成返回结果需要刷新UI的时候，由于UI线程不是线程安全的，在多线程并发的情况下访问UI线程，可能会造成UI控件处于不可预期的状态，所以Android不允许在其它线程中直接刷新UI。这样Handler就孕育而生了，通过Handler可以将有关UI的操作切换到主线程中去执行。

## Handler消息机制的主要构成

Handler消息机制主要由4部分组成：`Handler`、`MessageQueue`、`Looper`和`Message`。

### Handler

`Handler`用来发送和处理线程对应的`MessageQueue`中存储的`Message`的，每个Handler实例对应一个线程以及一个该线程的消息队列。当你新建一个`Handler`时，它会绑定到创建它的线程以及该线程的消息队列，然后它会向该消息队列发送`Message`或者`Runnable`，并且在它们离开消息队列时执行它们。

### MessageQueue

`Handler`其实相当于一个甩手掌柜，发送消息的任务其实是交给`MessageQueue`来完成的。**`MessageQueue`采用单链表结构来存储`Message`，并且是按照`Message`的触发时间的先后顺序来排序的。**`MessageQueue`调用`enqueueMessage()`方法将`Message`存储起来，而取出`Message`则时调用`next()`方法，该方法是一个死循环，不断的轮询`MessageQueue`中是否有`Message`，当没有`Message`时，为了避免消耗CPU，该方法就会阻塞，直到有新的`Message`到来，或则退出。

### Looper

虽然`MessageQueue`提供了`Message`入队、出队的方法，但是它并不是自动取出消息，想要从`MessageQueue`中取出消息，则需要`Looper`来执行。**创建`Handler`之前，必须先创建`Looper，否则会发生异常`**，一般线程默认是不会创建`Looper`的，需要我们主动创建，而主线程已经默认为我们创建了`Looper`，所以我们可以直接在主线程中创建`Handler`。创建`Looper`可以通过`Looper.prepare()`方法，创建`Looper`实例的时候，同时会创建`MessageQueue`实例，并且关联到当前线程，每个线程的`Looper`是通过`ThreadLocal`存储的，保证其线程私有。

想要使`MessageQueue`运转起来，则需要调用`Looper.loop()`方法，这是一个死循环，不停的从`MessageQueue`中取出消息，取出消息后交由`Handler`来分发，分发之后回收`Message`到消息池，以便重复利用。取出消息调用的是`MessageQueue`的`next()`方法，而分发消息则调用的是`Handler`的`dispatchMessage()`方法，该方法会调用`Handler`的`HandleMessage（）`方法。

### Message

`Message`实际就是消息的载体，它包含一些重要的属性：

```
int what ：消息标识
int arg1 : 可携带的 int 值
int arg2 : 可携带的 int 值
Object obj : 可携带内容
long when : 超时时间
Handler target : 处理消息的 Handler
Runnable callback : 通过 post() 发送的消息会有此参数
```

其中需要注意的是，**在`Handler`发送`Message`时，它会将`Message`的`target`属性设置为`Handler`自己**。

所以总结一下Handler的主要构成就是：

- Handler 被用来发送消息，但并不是真正的自己去发送。它持有 MessageQueue 对象的引用，通过 MessageQueue 来将消息入队。
- Handler 也持有 Looper 对象的引用，通过 `Looper.loop()` 方法让消息队列循环起来。
- Looper 持有 MessageQueue 对象应用，在 `loop()` 方法中会调用 MessageQueue 的 `next()` 方法来不停的取消息。
- `loop()` 方法中取出来的消息最后还是会调用 Handler 的 `dispatchMessage()` 方法来进行分发和处理。

## 一些常见的问题

- **一个线程能否创建多个Handler，Handler和Looper之间的对应关系是怎样的？**

  一个线程可以创建多个`Handler`，但是一个线程只能有一个`Looper`，所以也只对应一个`MessageQueue`。

- **`Looper.loop()`是死循环，为什么它不会卡死主线程？**

  其实两者并没有必然的联系，主线程中的`Looper.loop()`是让消息队列运作起来，一旦它退出了，程序也就退出了。它可能会阻塞主线程，但是只要消息循环没有阻塞，就不会引起ANR错误。造成ANR错误的不是主线程阻塞，而是主线程的`Looper`消息处理的时候发生了任务阻塞，无法响应及时刷新UI。

- **Handler发生泄露的原因以及解决方法？**

  `Handler`允许发送延时消息，而在延时的期间用户关闭了`Activity`，由于`Message`持有`Handler`对象，而由于Java特性，内部类持有外部类的引用，所以`Handler`持有`Activity`，所以`Activity`就泄露了。解决方案是将`Handler`改为静态内部类，其内部持有`Activity`的**弱引用**，并且在`Activity`的`onDestory()`回调中调用`handler.removeCallbacksAndMessages()`方法，及时移除所有消息。