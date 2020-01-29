---
title: Android App 避免螢幕鍵盤蓋住RecyclerView底部內容的方法
date: 2019-03-19 11:24:56
tags: Android native
categories: Android Note
---


最近以練功為目的寫了一個簡單的[聊天室App](https://github.com/DarKArieS/ChatNow)，遇到了一個跟彈出鍵盤有關的問題：如何不讓鍵盤遮住RecyclerView最底部的內容。

<img src="https://i.imgur.com/x5Wh6Kx.gif" width="30%">

<!-- more -->

剛開始解決這個問題的思路是寫一個View繼承自editText View並改寫`onTouchEvent`內彈出鍵盤的方法。研究了下源碼後發現裡面所調用方法跟成員有不少是無法改寫的。

在網路上查了一下監聽螢幕鍵盤彈出的方法，最有效的方法為監聽Layout的改變。在屬於Layout的view中加入`LayouyChangeListener`，並在監聽內加入滑到最底部的方法。

對於每次都滑到最底部感到不滿意，於是改成Layout改變時自動滑動Layout的改變量，營造出內容隨著鍵盤被推起來的效果。
不過這個方法在滑到最底部的時候會有滑過頭的問題，而在ViewHolder大小為動態的情況下我們無法得知RecyclerView還有多少可以滑動，想了半天還是無法解決。

有一天，很偶然的，發現其實只要一行就可以解決這個問題。:unamused:
在RecyclerView的LayoutManager中加入這一行：

```
layoutManager.stackFromEnd = true
```

使得RecyclerView在生成的時候是以底部為基準。
當鍵盤推出時，為了不擋住最底部的View，自然就會跟著被推起來。

<img src="https://i.imgur.com/iNluFyT.gif" width="30%">

讓我們緬懷那些已經失去的時光 :expressionless: