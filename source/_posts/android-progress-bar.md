---
title: Android 原生計量條動畫設計
date: 2019-01-14 14:03:05
tags: Android native
categories: Android Note
---

應用程式中，免不了需要進度條來顯示各種進度，像是下載進度、處理進度等。而遊戲中的計量條(例如血條)，在變化時更是需要輔以酷炫的動畫增進視覺效果。

這篇文章會使用android SDK原生功能以及ProgressBar來製作隨著倒數計時器變化的計量條，並且在增減時間時有酷炫的動畫。

<!-- more -->

預計會達到如下的效果：

- 一般情形：

![](https://i.imgur.com/ys9a4wW.gif)

- 時間增加：計量條閃爍，並且有像是格鬥遊戲扣血時的延遲動畫
![](https://i.imgur.com/y1nU64V.gif)

- 時間減少：倒扣時計量條黑色閃爍
![](https://i.imgur.com/tPyiEIc.gif)

簡單的程式架構如下：

![](https://i.imgur.com/UfGqUsA.png =300x150)

- Activity/Fragment中的按鈕送出更改剩餘時間的訊息給Timer、同時送出播放動畫的請求給Animator

- Timer負責倒數計時，並要求Animator更新Progress Bar

- Progress Bar被Animator控制，顯示剩餘時間以及播放動畫

原始碼可以在[這裡](https://github.com/DarKArieS/JettPuzzleGame2/tree/1.ProgressBar)找到

## 計量條外觀設置

原生提供的ProgressBar共有兩種繪製形式，一種是討厭的轉圈圈，我們需要的是另一種長條形的ProgressBar Horizontal。

![](https://i.imgur.com/HKk4Ivu.gif)


而ProgressBar又有分成計量型(Determinate)與無限~~吃到飽~~型(Indeterminate)，我們需要的是計量型來顯示進度。

![](https://i.imgur.com/gUgv83C.gif)


官方原生只有提供水平進度條。如果想使用圓形形式顯示進度，可以[參考這裡](https://github.com/Hellobird/CircleSeekBar-For-Android)。

原生的樣式有點細，讓我們重新設計ProgressBar的樣式。
建立一個time_bar.xml，程式碼如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" >
    <item android:id="@android:id/background">
        <shape>
            <corners android:radius="5dip" />
            <solid android:color="#88000000"/>
        </shape>
    </item>
    <item android:id="@android:id/secondaryProgress">
        <clip>
            <shape>
                <corners android:radius="5dip" />
                <gradient
                    android:angle="270"
                    android:centerColor="#C6B7FF"
                    android:centerY="0.75"
                    android:endColor="#C3B2FF"
                    android:startColor="#B9A4FF" />
            </shape>
        </clip>
    </item>
    <item android:id="@android:id/progress" >
        <clip>
            <shape>
                <corners android:radius="5dip" />
                <gradient
                    android:angle="270"
                    android:centerColor="#74EBFF"
                    android:centerY="0.75"
                    android:endColor="#8EEFFF"
                    android:startColor="#57E8FF" />
            </shape>
        </clip>
    </item>
</layer-list>
```

根據多方教學以及官方文檔，ProgressBar Drawable xml 必須為LayerDrawable，並包含三個部分：background(背景)、progress(主要進度條)、secondaryProgress(次要進度條)

進度條顯示計量進度是由clipDrawable控制，因此progress以及SecondaryProgress必須包在clip tag中。

值得注意的一點是，這邊的id形式為internal ID：
```
android:id="@android:id/progress
```
，並非我們在layout xml中使用的ID：
```
android:id="@+id/...
```
這類的id為android SDK已經事先定義好並用於SDK中的各個物件，無法隨意更改。

要取得擁有該id的物件沒辦法依靠常用的getViewById，方法如下：
```kotlin
private val progressBarDrawable = (progressBar.progressDrawable as LayerDrawable)
        .findDrawableByLayerId(Resources.getSystem().getIdentifier("progress","id","android"))
```
等會兒製作[閃爍動畫](#1)時會用上。

最後，在layout xml中需要注意的內容如下：
```xml
<ProgressBar
    android:id="@+id/timerProgressBar"
    android:progressDrawable="@drawable/time_bar"
    style="@android:style/Widget.DeviceDefault.Light.ProgressBar.Horizontal"/>
```


## 計時器

寫一個timer class，內容如下：

```kotlin=
import android.os.Handler

class GameTimer(private var timerBarController: GameTimer.TimerBarController){
    interface TimerBarController{
        fun timerOnUpdate()
        fun timesUp()
    }

    var secondsCount = 40f
    var maxTimeInSeconds = secondsCount
    var isOnStart = false
    private var stopTimer = false

    private lateinit var timerThread: Thread
    private var handlerUI = Handler()
    private lateinit var runnable: Runnable

    fun startTimer() {
        if(!isOnStart){
            isOnStart = true

            timerThread = Thread{
                while(secondsCount>=0 && !stopTimer){
                    Thread.sleep(10) // 0.01 second
                    secondsCount -= 0.01f
                }
                if(!stopTimer){
                    handlerUI.post{
                        timerBarController.timesUp()
                        stopTimer()
                    }
                }
                println("timer thread end")
            }

            runnable = Runnable {
                timerBarController.timerOnUpdate()
                handlerUI.postDelayed(runnable, 10)
            }

            stopTimer = false
            timerThread.start()
            handlerUI.postDelayed(runnable, 10)
        }
    }

    fun stopTimer() {
        stopTimer = true
        isOnStart = false
        handlerUI.removeCallbacks(runnable)
    }
}
```

當中設計了一個介面讓UI端程式實作計時器到期(`fun timesUp()`)以及更新UI(`fun timerOnUpdate()`)等事件。

計時的處理由一個thread負責，每0.01秒會更新一次數值。更新UI的部分則丟到handlerUI處理。
由於Thread並不建議被隨意地中止，因此在thread中由布林值`isOnStart`決定是否停止該計時器。

實例化計時器時，可以依不同情形配合progressBar設置。
像是這個樣子，同時設定計時器以及progressBar：
```kotlin=
// setup timer
val timeThreshold = 40f
timer = GameTimer(this)
timer.secondsCount = timeThreshold
timer.maxTimeInSeconds = timeThreshold
rootView.timerProgressBar.max = (timeThreshold*100).toInt() // timer bar resolution: 0.01 second
rootView.timerProgressBar.progress = (timeThreshold*100).toInt()
rootView.timerProgressBar.secondaryProgress = (timeThreshold*100).toInt()
```

計時器最大值為40秒。由於每0.01秒會更新一次計時器，progressBar的最大值為`40*100`。
每次計時器更新progressBar會使progress值減一，在[progressBar UI更新](#progressBarUI)會使用到這個特性。

## 延遲動畫

寫一個class，當中封裝了所需要用到的物件。
這邊主要解釋程式的想法，不把整段都貼出來。
有興趣的話請洽[完整程式碼](https://github.com/DarKArieS/JettPuzzleGame2/tree/1.ProgressBar):smiley:

<h3 id="progressBarUI">progressBar UI更新</h3>

由於原生的計量條顯示是由clip level所設定，因此將遭到變更的progress值更新到畫面上是一瞬間的事。[^clipAnimation]

[^clipAnimation]: Android API > 24 提供了`setProgress(int progress, boolean animate)`這個方法來設定進度條更新的動畫，不過只能選擇開啟跟關閉，而且動畫的時間長度是固定的(80ms)。


此處藉由控制progress值的增減時機來實現動畫效果。

首先，timer中的`fun timerOnUpdate()`負責更新progressBar，執行的內容如下：

```kotlin=
fun update(progressIncrement: Int, trueProgress: Int){
    if(isAnimatingUpdatingDelayed) progressBar.incrementProgressBy(progressIncrement)
    else progressBar.progress = trueProgress
    progressBar.secondaryProgress = trueProgress
}
```
這段程式中第二進度條是同步跟著timer走，而當沒有播放動畫時，主進度條亦然。

如果正在播放動畫，則由progressIncrement這個變數決定要增加或減少多少值，這個值取決於你的計時器以及progress的最大值如何設定。
這邊所營造的效果為：在動畫途中，progress仍然會以計時器該有的速度遞減，而不會完全停下來。
以此篇文章為例子，每0.01秒更新一次，每次更新progress的數值會減1，也就是`progressIncrement=-1`。


### 時間條增加時，延遲更新的動畫

先放上程式碼：

```kotlin=
fun animateUpdatingDelayed(delayTime: Long){
    if (!isAnimatingUpdatingDelayed){
        isAnimatingUpdatingDelayed = true
        Thread{
            Thread.sleep(delayTime)
            for (i in 1..9){
                progressBar.progress =
                        (progressBar.progress +
                                (progressBar.secondaryProgress - progressBar.progress)* i /10)
                Thread.sleep(50)
            }
            progressBar.progress = progressBar.secondaryProgress
            isAnimatingUpdatingDelayed = false
        }.start()
    }
}
```

設計思路是：開一個thread隨著時間一段一段更新progress的值。

開頭有個布爾值規定一次只能存在一個延遲動畫用的thread，避免多個thread控制同個物件所造成的crash。

在thread中，首先先等待一段時間，接下來將progress以及secondaryProgress之間的差值分成10等份依次更新，更新間隔為50 ms。
由於先前的`fun update()`仍然會不停的被計時器呼叫，因此每次更新都必須重新計算一次。
在thread運行中時，secondaryProgress是允許被改變的。在這段時間內再次按按鈕secondaryProgress會再度被更新，因次progress必須更快速的追上去。

程式碼當中有些Magic number(例如分成10等份)，看官有興趣可以將這些數字包成一個功能 :yum:

<h2 id="1">閃爍動畫</h2>

android中的View屬性動畫最基本的僅有移動、旋轉、縮放、淡入淡出，並沒有能改變已繪製物件「色調」的動畫。

<!-- Show個RPG製作大師改變色調的效果 -->

菜逼八的我想到這幾種策略：
- 1.在欲閃爍的物件上再繪製一層一模一樣的物件，改變該物件的顏色並使用淡入淡出動畫
- 2.在欲閃爍的物件上套上ColorFilter，隨著時間改變Filter的數值。
- 3.製作一個客製化的View繼承自ProgressBar，該View再多繪製一層遮罩，並且對遮罩使用淡入淡出動畫。

由於方案一似乎有點蠢，方案三我...還不會:sweat_smile:，在這裡我們使用方案二。

先放上程式碼：

```kotlin
fun progressShining(isDarker:Boolean = false, shiningTime: Long = 1000, shiningDegree: Float = 0.25f){
    animStepShining = 0
    mShiningTime = shiningTime/20
    mShiningDegree = shiningDegree
    mIsDarker = isDarker
    if(!isAnimatingShining){
        isAnimatingShining = true
        Thread{
            while(animStepShining<animMaxStepShining){
                val step : Float = when(mIsDarker){
                    true-> when(animMaxStepShining < 3){
                        true -> 1 - (mShiningDegree * (animStepShining.toFloat())/3)
                        false -> 1 - mShiningDegree + (mShiningDegree * ((animStepShining-3).toFloat())/17)
                    }
                    false-> when(animMaxStepShining < 3){
                        true -> 1 + (mShiningDegree * (animStepShining.toFloat())/3)
                        false -> 1 + mShiningDegree - (mShiningDegree * ((animStepShining-3).toFloat())/17)
                    }
                }
                val colorFilter = ColorMatrixColorFilter(
                    floatArrayOf(
                        1f*step,0f,0f,0f,0f,
                        0f,1f*step,0f,0f,0f,
                        0f,0f,1f*step,0f,0f,
                        0f,0f,0f,1f,0f
                    )
                )
                // To avoid thread conflicting OxO
                handlerUI.post{progressBarDrawable.colorFilter = colorFilter}
                animStepShining ++
                Thread.sleep(mShiningTime)
            }
            if(animStepShining>=animMaxStepShining){
                handlerUI.post{progressBarDrawable.clearColorFilter()}
                isAnimatingShining = false
            }
        }.start()
    }
}
```

設計思路雷同於延遲動畫，不過動畫分成兩段。
以閃白光為例，由20次loop來呈現突然變亮之後漸漸轉暗的效果：變亮(3)->變暗(17)，而黑色閃爍則反過來。

開頭三個變數是class的成員，寫在function外使得該他們可以在thread運行時被改變。同樣地，有個布爾值規定一次只能存在一個閃爍動畫用的thread。當在閃爍期間再次按下按鈕時，所有動畫進度相關的變數會被重置，再次播放全新的閃爍動畫。

簡單的解釋一下ColorMatrix的運作原理。
在Android drawable中，ColorFilter共分成三種：
ColorMatrixColorFilter、LightingColorFilter、PorterDuffColorFilter

本篇使用的ColorMatrix必須配合以下公式轉換整個drawable的像素顏色：

![](https://i.imgur.com/20YcKH6.png =300x200)

以白光閃爍為例，不改變Alpha值，最亮的白色為`(R,G,B)=(1f,1f,1f)`[^first]，因此ColorMatrix設計成使得改變後的色彩為原色彩值一同乘上大於一的值，並依照動畫進程計算乘上的大小。而變暗則反過來乘上小於一的值。

![](https://i.imgur.com/Bai5vHW.png =300x200)

簡化成上面的式子，白光閃爍所用的值為 R~R~ = G~G~ = B~B~ > 1。

如果想改成其他的閃爍顏色，只要調整 R~R~ , G~G~, B~B~ 之間的比例就可以了。

[^first]: 1f中的f代表浮點數的意思。使用整數表達則(R,G,B)=(255,255,255)(32bit 深度)

而其他兩種不會用到的功能簡單地說：
- LightingColorFilter: ColorMatrix特定公式的版本，等同於我們變亮變暗所使用的式子再加上(R~C~,G~C~,B~C~)這一項
- PorterDuffColorFilter: 依照不同的模式將特定範圍的顏色替換。由於不曉得閃光顏色的漸變時所呈現的每一種顏色，故無法實現我們想要的效果。而顏色的alpha值只會讓原物件變成半透明，背景會透出來。

較詳細的說明可以參考[這一篇](https://blog.csdn.net/allen315410/article/details/45059989)。



## 結語

將以上兩種動畫結合起來即可呈現出文章開頭的動畫效果。
方法老實說有點粗糙:confounded:，也許比較理想的方式是客製化的View，不過表現也算令人滿意(吧?)

第一次寫文，請多指教 :smiley: