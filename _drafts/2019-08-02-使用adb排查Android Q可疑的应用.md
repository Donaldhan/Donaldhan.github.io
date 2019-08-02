---
layout: page
title: 使用adb排查Android Q可疑的应用
subtitle: 使用adb排查Android Q可疑的应用
date: 2019-08-02 15:12:00
author: donaldhan
catalog: true
category: util
categories:
    - util
tags:
    - android
---

# 引言

首先安装adb：
https://www.xda-developers.com/install-adb-windows-macos-linux/

adb的使用参考
https://developer.android.com/studio/command-line/adb

1、根据包名查看进程命令adb shell ps|grep com.ott.android.TMC(包名) 直接输入会提示grep不是内部命令或外部命令解决办法，解决办法加上引号adb shell “ps|grep com.ott.android.TMC(包名)” 

windows使用

如下命令,查看指定进程
```
adb shell ps | findstr "logcat"
```

如果进程找不到的话再，查看激活应用的日志
```
adb logcat | findstr "ActivityManager"
```
找到可以的应用

2、根据包名直接杀掉进程命令 
adb shell am force-stop com.ott.android.TMC（包名） 


3、查看进程列表 
adb shell ps


如果出现ADB Android Device Unauthorized

https://stackoverflow.com/questions/23081263/adb-android-device-unauthorized
可以尝试
```
adb kill-server
adb start-server
```


然后授权即可真机只要授权即可。

在使用，adb 查看最顶层activity名称

```
windows环境下:
adb shell dumpsys activity | findstr “mFocusedActivity”
linux:
adb shell dumpsys activity | grep “mFocusedActivity”
```

使用如上命令没有找到，将活动窗口导到文件中
```
adb shell dumpsys activity > log.log
```
搜索“focus”关键字
可以看到
```
 ResumedActivity: ActivityRecord{9e7926c u0 com.snda.wifilocating/com.lantern.pseudo.app.PseudoLockFeedActivity t4020}
  mFocusedStack=ActivityStack{998cb8e stackId=388 type=standard mode=fullscreen visible=true translucent=false, 1 tasks} mLastFocusedStack=ActivityStack{998cb8e stackId=388 type=standard mode=fullscreen visible=true translucent=false, 1 tasks}
  mCurTaskIdForUser={0=4020}
  mUserStackInFront={}
  displayId=0 stacks=6
   mHomeStack=ActivityStack{9d6ffbc stackId=0 type=home mode=fullscreen visible=false translucent=true, 1 tasks}
  isHomeRecentsComponent=true  KeyguardController:
    mKeyguardShowing=false
    mAodShowing=false
    mKeyguardGoingAway=false
    mOccluded=true
    mDismissingKeyguardActivity=ActivityRecord{9e7926c u0 com.snda.wifilocating/com.lantern.pseudo.app.PseudoLockFeedActivity t4020}
    mDismissalRequested=false
    mVisibilityTransactionDepth=0
  LockTaskController
    mLockTaskModeState=NONE
    mLockTaskModeTasks=
    mLockTaskPackages (userId:packages)=
      u0:[]
```
最后发现是wifi万能钥匙应用导致的干：
```
com.snda.wifilocating
```

出现错误：This adb server's $ADB_VENDOR_KEYS is not set

https://stackoverflow.com/questions/33157299/this-adb-servers-adb-vendor-keys-is-not-set

