---
title: '咖啡的故事 Ep.2： Reactive X 入門'
date: 2019-06-06 10:08:12
tags: 
- 'Android native'
- 'ReactiveX'
- 'RxJava'
categories: 'Android Note'
---

# Intro

剛開始處理非同步的問題時，想要自己寫一個可以完美處理各種 callback 的系統。雖然那時候已經知道有一套玩意兒叫 Rx ，但還是死撐著想自己寫。最後的結果當然是......寫出了一坨只有當下的自己才懂的東西。不過也學到用閉包處理 callback 以及用物件包裝 function 等等的...邪門歪道(?)。

最後在某人的大力推薦(?)下，還是回歸正途，來學學大家都在用的 ReactiveX 吧！

<!-- more -->

# Reactive programing

在學 Rx 之前，先來討論哲學問題。

~~想不到梗~~，讓我們回到咖啡機的故事，並且請暫且忘掉[上一篇](https://darkaries.github.io/2019/05/15/androidDI/)。(根本沒人記得)

這次登場的角色是水跟加熱器。

在煮咖啡的流程中，我們需要熱水，產生熱水的流程如下：


> 打開加熱器 -> 水從加熱器獲得能量 -> 水變熱了


寫成程式看看：

### 加熱器
```kotlin
class Heater{
    private var energy = 0

    fun on(){
        println("~ ~ ~ heating ~ ~ ~")
        energy = 60
    }

    fun off(){
        energy = 0
    }

    fun getEnergy() = energy
}
```

### 咖啡機
```kotlin
class CoffeeMaker(
    private val water: Water,
    private val heater: Heater,
    private val pump: Pump
){
    fun brew(){
        heater.on()
        water.degree += heater.getEnergy()
        ...
    }
}
```

咳，讓我們先忽略能量與溫度之間的轉換關係，就當作水有一公升這麼多，而且能量的單位是大卡吧。

看起來沒問題，執行起來也對。
但是有個小瑕疵：**打開加熱器跟水變熱其實並沒有明確的因果關係**。

會對的原因是因為程式**剛好**是一行一行往下執行的。

如果加熱器升溫是需要一段時間才能完成的呢？

聰明的你，看到這裡應該會想到同步異步的問題了，不過先別急。

---

如果有一天，隔壁鄰居的小屁孩來你家玩，並且看上了你的咖啡機。
他是這樣子使用這臺咖啡機的：


> 打開加熱器 -> 關閉加熱器 -> 打開加熱器 -> 關閉加熱器 ...


~~第一時間當然是巴蕊。~~

程式會變這樣：

```kotlin
class CoffeeMaker(
    private val water: Water,
    private val heater: Heater,
    private val pump: Pump
){
    fun whatTheFuckingKid(){
        heater.on()
        water.degree += heater.getEnergy()
        
        heater.off()
        water.degree = 25 //回到了常溫
        
        heater.on()
        water.degree += heater.getEnergy()
        ...
    }
}
```

每次加熱器的狀態改變，我們都必須要手動的改變水的狀態，超麻煩的啦。

正常來說，應該是加熱器打開後水就會自己變熱才對啊！是水主動反應 (React) 了加熱器加熱這件事。

關於 Reactive programing 網路跟書上有各種不同的描述跟定義，看得眼花撩亂頭昏眼花。

也許這裡舉的例子可能也沒有非常精確，但是如果你問我，我會說是有明確描述 **因為所以** ~~蟑螂螞蟻~~ 的程式，並且把因果關係包裝起來，只要發生了 **因為** ，就一定會觸發 **所以**。


> 因為 **加熱器打開了** 所以 **水變熱** 了。


讓我們試著來寫寫看。

## Observer pattern

Reactive programing 的實作方式有百百種，想怎麼寫就怎麼寫。

為了承上啟下，結合之後要討論的 RxJava ，我們用[觀察者模式](https://en.wikipedia.org/wiki/Observer_pattern)來做。

在這個場景中，水觀察(訂閱)加熱器，加熱器通知水要變熱或變冷。水是觀察者 (Observer) ，加熱器是被觀察者 (Observable) 。

因為所以的關係為： 因為 被觀察者 做了某些事 所以 觀察者 做出了某些反應。 

(有夠哲學的啦)

先來改造水，讓水有個方法可以主動反應「加熱」這件事。

```kotlin
class Water{
    var degree = 25
    
    fun beHeated(energy : Int){
        degree += energy
    }
}
```

接下來是加熱器，可以接受要被加熱的水，以及在打開之後「通知」目標變熱。

```kotlin
class Heater{
    private var energy = 0
    private var mToBeHeated : Water? = null

    fun setToBeHeated(water:Water){
        mToBeHeated = water
    }

    fun on(){
        println("~ ~ ~ heating ~ ~ ~")
        energy = 60
        mToBeHeated?.beHeated(energy)
    }

    fun off(){
        energy = 0
        // 降溫，拜託不要吐槽 :p
        mToBeHeated?.beHeated(-60)
    }

    fun getEnergy() = energy
}
```

咖啡機會變這樣：

```kotlin
class CoffeeMaker(
    private val water: Water,
    private val heater: Heater,
    private val pump: Pump
){
    fun whatTheFuckingKid(){
        heater.setToBeHeated(water)
        heater.on()
        heater.off()
        heater.on()
        heater.off()
        heater.on()
        heater.off()
        ...
    }
}
```

### 物件導向式的擴展

嗯......除了水以外好像也可以加熱其它東西啊！
來訂個規則吧！所有可以被加熱的東西都要遵守。

```kotlin=
interface Heatable{
    fun beHeated(energy : Int)
}
```

水必須遵守這個規則：

```kotlin=
class Water : Heatable{
    var degree = 25

    override fun beHeated(energy : Int){
        degree += energy
    }
}
```

於是我們可以這樣用加熱器：

```kotlin=
//加熱水
heater.setToBeHeated(water)
heater.on()
......
//加熱牛奶，牛奶這個class該長什麼樣子由你決定 :D
heater.setToBeHeated(milk)
heater.on()
...
//加熱不知道什麼東西，使用匿名物件
heater.setToBeHeated(object:Heatable{
    override fun beHeated(energy: Int) {
        //不知道會發生什麼事
    }
})
......
```

### 函式導向式的擴展

仔細想一想，其實加熱器根本不用在乎到底什麼東西可以被加熱什麼東西不可以被加熱啊！它只需要做到讓加熱後的事情發生就可以了！

我們可以透過 Lambda function 來告訴加熱器在加熱之後會發生什麼事情。

於是加熱器變成這個樣子：
```kotlin
class Heater{
    private var energy = 0
    private var afterHeatedCallback : ((Int)->Unit)? = null
    
    fun afterHeated(callback : ((Int)->Unit)):Heater{
        afterHeatedCallback = callback
        return this
    }

    fun on(){
        println("~ ~ ~ heating ~ ~ ~")
        energy = 50
        afterHeatedCallback?.invoke(energy)
    }

    fun off(){
        energy = 0
        afterHeatedCallback?.invoke(-50)
    }

    fun getEnergy() = energy
}
```

使用加熱器時是這個樣子的，由於`afterHeated`方法會回傳類別自己，所以可以把命令串起來：

```kotlin
heater.afterHeated { energy->
    water.beHeated(energy)
}.on()
...
```

<!-- 在這邊順帶一提，無論是物件導向式還是函數式擴展，都可以透過閉包 (closure) 機制來拿取屬於不同 scope 之中的值，來達到隨心所欲的境界！ -->


## 耗時操作

如果加熱器加熱水需要一段時間，該怎麼辦呢？
像是這個樣子：

```kotlin
fun on(){
    Thread{
        println("~ ~ ~ heating ~ ~ ~")
        Thread.sleep(1000)
        energy = 50
    }.start()
}
```

由於任務是由加熱器執行的而不是咖啡機，我們另開一條執行緒給它。

執行一下流程，原本的非reactive方法會立刻爆炸：

```kotlin
heater.on()
//heater 還沒完成加熱就會馬上往下執行
water.degree += heater.getEnergy()
//結果water的溫度還是沒有上去
```

有了以上兩種擴展，處理這件事情就變得比較容易了！
加熱器長這樣，把設定好的物件方法/匿名函式拿進來用：

```kotlin
fun on(){
    Thread{
        println("~ ~ ~ heating ~ ~ ~")
        Thread.sleep(1000)
        energy = 50
        //用匿名物件
        mToBeHeated?.beHeated(energy)
        //或是用匿名函式：
        //afterHeatedCallback?.invoke(energy)
    }.start()
}
```

我們只需要把後續發生的事情通通寫到匿名物件/匿名函式之中就可以了。藉由閉包 (closure) 機制，甚至可以隨意拿取屬於不同 scope 之中的成員，帥呀！

來看看程式碼會長什麼樣子：

```kotlin
fun brew(){
    //不小心放了個炸彈在這裡
    //這個炸彈所在的scope與 heater 不同
    val bomb = Bomb()
    heater.setToBeHeated(object:Heatable{
        override fun beHeated(energy: Int) {
            // 藉由 closure 機制把 bomb 傳進來
            // 把加熱器打開炸彈會爆炸！
            bomb.explode()
        }
    })
    heater.on() // 開開看吧，嘿嘿...
}
```

# Reactive stream

反應的問題處理好了，讓我們繼續關注咖啡該怎麼煮，總不能只是熱熱開水吧 :D

來複習(誤)一下煮咖啡的流程，以水的角度出發：


> 水 -> 加熱器打開 -> 熱水 -> 幫浦抽水沖咖啡 -> 變成咖啡



現在，水除了要對加熱器做出反應，同時也要對幫浦做反應了，水在此扮演了資料流 (stream) 的角色，被傳來傳去的。

用函式的方法做做看吧！


水

```kotlin=
class Water : Heatable{
    var degree = 25

    override fun beHeated(energy : Int){
        degree += energy
    }
}
```

加熱器：

```kotlin=
class Heater{
    private var energy = 0
    private var afterHeatedCallback : ((Int)->Unit)? = null

    fun afterHeated(callback : ((Int)->Unit)):Heater{
        afterHeatedCallback = callback
        return this
    }

    fun on(){
        Thread{
            println("~ ~ ~ heating ~ ~ ~")
            Thread.sleep(1000)
            energy = 60
            afterHeatedCallback?.invoke(energy)
        }.start()
    }

    fun off(){
        energy = 0
        afterHeatedCallback?.invoke(-60)
    }

    fun getEnergy() = energy
}
```

增加兩個類別：幫浦以及咖啡

幫浦：

```kotlin=
class Pump {
    private var afterPumpCallback : ((Coffee)->Unit)? = null

    fun afterPump(callback : ((Coffee)->Unit)):Pump{
        afterPumpCallback = callback
        return this
    }

    fun pump(water:Water){
        if (water.degree < 85){
            println("Degree of water is too low!")
        }else{
            Thread{
                println("=> => pumping => =>")
                Thread.sleep(1000)
                val coffee = Coffee()
                afterPumpCallback?.invoke(coffee)
            }.start()
        }
    }
}
```

咖啡，不解釋：

```kotlin=
class Coffee {}
```

那麼，該怎麼煮咖啡呢?

```kotlin=
class CoffeeMaker(
    private val water: Water,
    private val heater: Heater,
    private val pump: Pump
){
    fun brew(){
        pump.afterPump{ coffee->
            println(" [_]P coffee! [_]P ")
            heater.off()
        }

        heater.afterHeated { energy->
            water.beHeated(energy)
            // 由於降溫也會觸發這個函式，降溫的時候就不沖咖啡了！
            if (energy > 0)
                pump.pump(water)
        }.on()
    }
}
```

哇啊，好像變倒敘法了。 

當然，我們可以挪動一下程式碼，這不成問題。

糟糕的是，在這份程式碼中，總共使用了兩次`Thread{...}.start()`，開了兩個額外的執行緒。
如果為了不浪費計算資源，勢必得寫更多的程式來管理執行緒，光想就頭痛。

而且，並沒有妥善處理當加熱器或者幫浦不小心出事的例外情況。

這下子，該請大名鼎鼎的 ReactiveX 出來啦！

# ReactiveX

有關於 [ReactiveX](http://reactivex.io/) 這套程式，網路上可以找到相當多的簡介，就不在這裡再說一遍了。

它支援~~多國語言~~，不對，各式各樣的程式語言，我們寫 Kotlin 的人可以使用 [RxJava](https://github.com/ReactiveX/RxJava) 或是 [RxKotlin](
https://github.com/ReactiveX/RxKotlin) 。兩者的差別在於 RxKotlin 支援了更多 Kotlin 的語法糖。

在這篇文章中我們先使用 RxJava 就好。

## Dependency

在 Android 的世界中有許多的套件可以直接與 RxJava 串接，例如 Retrofit 、 Room 、 LiveData 等等。

礙於篇幅，本篇將不會介紹這些東西。

在 Gradle 中添加：

```groovy
implementation "io.reactivex.rxjava2:rxjava:2.2.7"
```

另外， RxJava 也提供了方便的接口來切換執行緒，在 Android 中請搭配 RxAndroid 使用。

```groovy
implementation 'io.reactivex.rxjava2:rxandroid:2.1.1'
```

這個 Library 中提供了 `AndroidSchedulers.mainThread()` 方法來直接拿取 Android 主執行緒，或是使用 Looper 等 Android 中特有的方法來處理執行緒。

## 用看看吧！

在 RxJava 中，主要使用的組件有的觀察者 (Observer) 、被觀察者 (Observable) 以及操作子 (Operator) ，以及 Subject ，本篇會跳過 Subject 的部分。

首先先介紹最基本的冷觀察者組合。
所謂的冷觀察是指被觀察者在被觀察的時候才會做事，並將結果通知觀察者。有夠被動的有沒有呀。

### Observer

來看看 Observer 長什麼樣子：

```kotlin
val observer = object: Observer<DataType> {
    override fun onSubscribe(d: Disposable) {}

    override fun onNext(output: DataType) {}
    
    override fun onComplete() {}
    
    override fun onError(e: Throwable) {}

}
```

``<DataType>`` 中放入 Observable 與 Observer 之間用來傳輸資料的類別。

Observer 是個 abstract class，開了四個窗口，對應 Observable 傳來通知的四種狀況：

- onSubscribe:
在訂閱的瞬間會觸發的事件，傳入的 Disposable 可以用來解除此次的訂閱。

- onNext:
Observable 完成階段性任務後將結果送過來，可以被觸發多次。

- onComplete:
當 Observable 完成了所有任務之後觸發，到此訂閱關係完滿結束。

- onError:
Observable 不小心出事了會觸發，必須小心地把 exception 接起來。

### Observable

Observable 會透過 Creating Operator 來創造。
創建最簡單的 Observable 的方式就是使用 `Create` 這個 Operator 。

```kotlin
val observable = Observable.create<DataType>{ emitter->
    // 被觀察者做事情
    emitter.onNext( ... )
    // 被觀察者做事情
    emitter.onNext( ... )
    ...
    // 被觀察者出包了
    if (...) emitter.onError(...)
    ...
    emitter.onComplete( ... )
}
```

Observable 中可以自由的放入各種邏輯運算，並在適當的時候把結果透過 emitter 傳遞出去。 Emitter 提供了 `onNext` 、 `onError` 、 `onComplete` 等方法，對應剛剛所提的 Observer 的其中三個方法。

### Subscribe

一鍵訂閱，順便解決掉線程問題！
```kotlin
observable
    .subscribeOn(Schedulers.io()) // Observable在
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(observer)
```

有個更簡潔的寫法！我們不做 observer 物件了，直接由 observable 所提供的方法來設定：

```kotlin
val disposable = observable
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ output->
        //onNext
    },{ e->
        //onError
    })
```

一口氣處理了 `onSubscribe` (使用這個方法最後會返回一個 Disposable 類別，也就是 `Observer.onSubscribe` 會收到的那一個)、 `onNext` 、 `onError`。

長的是不是跟剛剛寫過的煮咖啡程式有點像？

至於你說 `onComplete` ? Observable 中提供了一堆 `doOn...` 的方法，像是 `doOnComplete`，聰明如你，應該知道該怎麼做了吧！

不過使用 `doOn...` 這些方法請小心執行緒的問題，放在 `subscribeOn` 之後會在屬於 observable 的執行緒上跑，而放在 `observeOn` 則會在 observer 的執行緒上。

### Operator

操作子的目標對象為被觀察者 Observable ，藉由操作子可以將各式各樣的 Observable 串接成一條 Reactive Stream，讓資料流通過層層關卡，達成我們要的目的。

根據[文件](http://reactivex.io/documentation/operators.html)， Operator 可以分成幾個類：

- Creating 
- Transforming 
- Filtering 
- Combining 
- Error Handling
- ...

等等，也太多了吧？

沒關係，見一個學一個，讓我們繼續看下去。

## 咖啡機 ver. RxJava

不囉嗦，上程式。

Heater 以及 Pump ， 被觀察者一族：

加熱器

```kotlin
class Heater{
    private var energy = 0

    fun on(): Observable<Int> {
        return Observable.create<Int> {emitter->
            println("~ ~ ~ heating ~ ~ ~")
            Thread.sleep(1000)
            energy = 60
            emitter.onNext(energy)
            emitter.onComplete()
        }
    }

    fun off(){
        energy = 0
    }

    fun getEnergy() = energy
}
```

幫浦
```kotlin
class Pump {
    fun pump(water:Water):Observable<Coffee>{
        return Observable.create<Coffee> { emitter ->
            if (water.degree < 85){
                emitter.onError(Throwable("Degree of water is too low!"))
            }else{
                println("=> => pumping => =>")
                Thread.sleep(1000)
                val coffee = Coffee()
                emitter.onNext(coffee)
                emitter.onComplete()
            }
        }
    }
}
```

煮咖啡程式變成這樣，使用 [flatMap](http://reactivex.io/documentation/operators/flatmap.html) 算子把整個流程連接起來。

flapMap算子會提供上一個 Observable 的輸出，並要求返回一個 Observable ，型別隨需求決定。

最後長得像這樣：
```kotlin
class CoffeeMaker(
    private val water: Water,
    private val heater: Heater,
    private val pump: Pump
){
    private val disposable = CompositeDisposable()

    fun brew(){
        val procedure = heater.on().flatMap {energy->
            water.beHeated(energy)
            pump.pump(water)
        }

        disposable.add(
            procedure.subscribe {coffee->
                println(" [_]P coffee! [_]P ")
                heater.off()
            }
        )
    }

    fun cancel(){
        disposable.clear()
    }
}
```

順便追加一個 `cancel` 方法，藉由 disposable 來取消任務。
現在就算咖啡煮到一半，我們也可以強制的將還沒跑完的流程中斷了！

是不是很簡單呢？

這段範例程式中並沒有把 `heater.off()` 後，水降溫的事件接進來，讀者可以自己試試看！

# Outro

雖然這邊僅舉一個例子，不過 Rx 的精髓在於搭配博大精深的 Operator 們可以做出千變萬化的組合。

另外，本篇沒有提及的是，Rx還能處理「背壓」的狀況，像是第一章所提及的小屁孩所做的行為：Observable 推送資料的速度大過於 Observer 消化資料的速度。

想知道更厲害的用法嗎？時間不早了，就讓我們改天有緣再相會吧。
