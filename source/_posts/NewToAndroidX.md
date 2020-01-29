---
title: 初試 AndroidX，解決 Didn't find class "android.support.design.widget...." 問題
date: 2019-03-19 11:24:56
tags: Android native
categories: Android Note
---


某次Android Studio更新[^clipAnimation]之後，偶然發現在創建新專案時多了個沒看過的選項：
`Use AndroidX artifacts`

[^clipAnimation]: 寫這篇的時候使用的是Android Studio 3.3.2

<img src="https://i.imgur.com/7yTBrJc.png" width="75%">

依官網的[說明](https://developer.android.com/jetpack/androidx)，這是個加強版的 [Android Support Library](https://developer.android.com/topic/libraries/support-library/index)

勾勾看會發生什麼事吧！


<!-- more -->


### 問題很快的就來了！
在調用某些物件後，在執行時期出現了找不到各種class的情況。(Build沒有問題)

例如，從IDE中拖個AppBarLayout出來，Build完成後執行，App Crash：

> Error inflating class android.support.design.widget.CoordinatorLayout
> ...
> Didn't find class "android.support.design.widget.CoordinatorLayout" on path: ...

欸？
`android.support.design`不是應該很理所當然的已經加進Gradle了嗎？
打開app gradle file瞧瞧。

<img src="https://i.imgur.com/ytuQRpJ.png" width="90%">

發現相關的Library都被換成帶有AndroidX關鍵字的版本。

仔細瞧瞧XML檔，發現IDE為你準備的物件仍舊來自於 `Andorid.Support.XXX` ：

```xml
<android.support.design.widget.CoordinatorLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:fitsSystemWindows="true"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    <android.support.design.widget.AppBarLayout
            android:id="@+id/appbar"
            android:fitsSystemWindows="true"
            android:layout_height="192dp"
            android:layout_width="match_parent">
			
			......
```

那我們自己加吧！手動的把Android.support.xxx加進來了，問題仍舊存在，依然是可Build不可執行。

###  解決方式

忙了一天，終於發現，只需要把xml內的物件依照[這個列表](https://developer.android.com/jetpack/androidx/migrate)替換就解決了。

期望之後Android Studio更新後能自動抓取對應的Library，不用再手動更改了！