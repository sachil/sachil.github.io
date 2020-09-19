---
layout: post
title:  Android基础-Intent和Intent Filter的匹配规则
mathjax: false
date: 2020-08-31 20:34:23
urlname:  Android基础-Intent和Intent Filter的匹配规则
categories:
  - Android基础
tags:
 - Android基础
---

# Android基础-Intent和Intent Filter的匹配规则

当收到隐式 Intent 以启动 Activity 时，系统会根据以下三个方面将该 Intent 与 Intent 过滤器进行比较，搜索该 Intent 的最佳 Activity：

- 操作：action
- 类别：category
- 数据：data

## 操作(action)的匹配

action的匹配规则是：**intent中的action至少有一个与过滤器(intent filter)中的action匹配，才能调用过滤器所在的组件，否则将无法命中。**所以，intent或者Intent filter中任意一个中不包含action，都无法通过测试。

## 类别(category)的匹配

category的匹配规则是：**intent中的每一个category都必须与过滤器(intent filter)中的category相匹配，否则无法命中。而相反的，intent filter中的category可以超出intent中的category的数量，intent是可以通过测试的。**

## 数据(data)的匹配

data的匹配规则是：

- **如果过滤器(intent filter)中未指定任何URI和MIME类型，只有不包含URI和MIME类型的intent才会通过测试。**
- **如果intent中包含URI而不包含MIME类型，则只有当intent filter中也不包含MIME类型，且intent filter中的URI和intent中的URI相匹配时，intent才会通过测试。**
- **如果intent filter中只包含MIME类型而不包含URI，只有当intent中的MIME类型与intent filter中的MIME类型相匹配，并且intent包含`content:`或者`file`的URI，intent才会通过测试。**
- **如果intent filter中同时包含MIME类型和URI，则intent也必须包含与之相匹配的MIME类型和URI，intent才会通过测试。**

URI通常包含这些属性：scheme(架构)，host(主机)，port(端口)，path(路径)。URI的格式为：`<scheme>://<host>:<port>/path`，URI的匹配规则是：

- 如果过滤器仅指定架构，则具有该架构的所有 URI 均与该过滤器匹配。
- 如果过滤器指定架构和权限，但未指定路径，则具有相同架构和权限的所有 URI 都会通过过滤器，无论其路径如何均是如此。
- 如果过滤器指定架构、权限和路径，则仅具有相同架构、权限和路径的 URI 才会通过过滤器。

## 注意事项

如果当前设备中没有能够匹配你发送到 startActivity() 的隐式 Intent，则调用将会失败，且应用会崩溃。这时候需要调用Intent的resolveActivity()方法进行测试，若其返回结果不为null，则可以安全调用。

```kotlin
val intent = Intent(Intent.ACTION_EDIT)
intent.addCategory(Intent.CATEGORY_APP_EMAIL)
val uri = Uri.parse("content://baidu.com")
intent.setDataAndType(uri,"text/plain")
if (intent.resolveActivity(packageManager) != null) {
    Log.e("Test","Start activity")
    startActivity(intent)
}
```

