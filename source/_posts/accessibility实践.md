title: acessibilitySerVice豆芽抢红包
date: 2018-10-31 22:59:09
tags:
- Android
categories:
- Dev
---

迫于忘记写ITP要发红包, 顺手写了个自动抢红包的demo,关于AccessibilityService的介绍如下:
>AccessibilityService作为service服务运行在后台中，通过AccessibilityEvent接收指定事件的回调。这样的事件表示用户在界面中的一些状态转换，例如：焦点改变了，一个按钮被点击，等等。这样的服务可以选择请求活动窗口的内容的能力。简单的说AccessibilityService就是一个后台监控服务，当你监控的内容发生改变时，就会调用后台服务的回调方法

## 实现步骤:  
### manifest中注册服务
```
<service
        android:name=".DYHBService"
        android:enabled="true"
        android:exported="true"
        android:label="@string/douya_hongbao"
        android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService"/>
    </intent-filter>

    <meta-data
            android:name="android.accessibilityservice"
            android:resource="@xml/accessibility_service_config"/>
</service>
```
accessibility_service_config的配置如下
```
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/app_name"
    android:accessibilityEventTypes="typeWindowStateChanged|typeWindowContentChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagDefault"
    android:canRetrieveWindowContent="true"
    android:notificationTimeout="10"
    android:packageNames="com.suning.snmessenger" />
```

* android:description ：辅助功能描述
* android:accessibilityEventTypes：辅助功能处理事件类型，一般配置为typeAllMask表示接收所有事件
* android:packageNames ：需要监听的应用包名
* android:accessibilityFlags：辅助功能查找截点方式，一般配置为flagDefault默认方式。
* android:accessibilityFeedbackType：反馈类型,包括声音,震动等等,一般指定GENERIC
* android:notificationTimeout：设置同一事件响应的间隔,避免频繁的调用
* android:canRetrieveWindowContent：是否允许辅助功能获得窗口的节点信息,配置为true 
###  开启辅助功能
判断是否开启,跳转设置界面
```
val isOpen = isAccessibilitySettingsOn(this, DYHBService::class.java.name)
if (!isOpen) {
    val intent = Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)
    startActivity(intent)
}
```
```
private fun isAccessibilitySettingsOn(context: Context?, className: String): Boolean {
    if (context == null) {
        return false
    }
    var accessibilityEnable: Int = 0
    val serviceName: String = context.packageName + "/" + className
    try {
        accessibilityEnable =
                Settings.Secure.getInt(context.contentResolver, Settings.Secure.ACCESSIBILITY_ENABLED, 0)
    } catch (e: Exception) {
        Log.e("mainActivity", "get accessibility enable failed, the err:" + e.message)
    }
    if (accessibilityEnable == 1) {
        val mStringColonSplitter = SimpleStringSplitter(':')
        val settingValue =
            Settings.Secure.getString(context.contentResolver, Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES)

        if (settingValue != null) {
            mStringColonSplitter.setString(settingValue)
            while (mStringColonSplitter.hasNext()) {
                val accessibilityService: String = mStringColonSplitter.next()
                if (accessibilityService.equals(serviceName, true)) {
                    Log.v("mainActivity", "We've found the correct setting - accessibility is switched on!");
                    return true
                }
            }
        }
    } else {
        Log.d("mainActivity", "Accessibility service disable")
    }
    return false
}

```
###  实现辅助功能的逻辑
这里需要重写onAccessibilityEvent()方法
* 窗口的内容出现了变化,即TYPE_WINDOW_CONTENT_CHANGED
通过DDMS查到红包对应的resourceId, 调用performAction模拟点击红包view
![](https://upload-images.jianshu.io/upload_images/4960358-df714e2ffc32c057.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
if (event.eventType == AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED) {
    val s = "xxxx" //包名
    val weikaiList: List<AccessibilityNodeInfo> =
        mRootNodeInfo!!.findAccessibilityNodeInfosByViewId("$s:id/receive")

    for (nodeInfo in weikaiList) {
        if (nodeInfo.text == "领取红包") {
            nodeInfo.parent.performAction(AccessibilityNodeInfo.ACTION_CLICK)
        }
    }
}
```
* 点击领取红包后,会弹出红包的dialog,会触发TYPE_WINDOW_CONTENT_CHANGED
同样通过DDMS可以查到红包dialog对应的viewId, 调用performAction模拟点击红包view
![](https://upload-images.jianshu.io/upload_images/4960358-e392e936f697a88d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
val clickedWindowList =
    mRootNodeInfo!!.findAccessibilityNodeInfosByViewId("$s:id/redpackets_open")
if (clickedWindowList.size > 0) {
    val curNodeInfo1 = clickedWindowList[0]
    curNodeInfo1.performAction(AccessibilityNodeInfo.ACTION_CLICK)
}
```
* 领取红包后, 会跳到红包的领取详情界面,会触发TYPE_WINDOW_STATE_CHANGED
同样调用performAction模拟点击箭头退出页面, 回到主页面继续抢红包
```
val backlist =
    mRootNodeInfo!!.findAccessibilityNodeInfosByViewId("$s:id/back_btn")
if (backlist.size > 0) {
    val curNodeInfo1 = backlist[0]
    curNodeInfo1.performAction(AccessibilityNodeInfo.ACTION_CLICK)
}
```




