---
title: '咖啡的故事： 依賴注入、Dagger、Android'
date: 2019-05-15 17:03:23
tags: 
- 'Android native'
- 'Dagger2'
- 'Dependency injection'
categories: 'Android Note'
---

[簡報版](https://drive.google.com/open?id=14P34DkHrt8KHfEYg-8EHOGk3P_cwSrt_)

最近還是在閱讀一些 [Google Android 範例](https://github.com/googlesamples/android-architecture-components)，有個叫做 Dagger 的東西常常出現，讓人以為又要開始算Hermitian $H^{\dagger}$ 矩陣了。
該框架引入了一堆Annotation，看得頭昏眼花， Ctrl+B 追了半天什麼心得都沒有(誤)

好啦，亂扯一通。大家都知道 Dagger 在做的事情是依賴注入 (Dependency Injection, DI) ，那它到底在幹嘛？我們幹嘛要用它？

網路上關於DI的文章隨便抓都一大把，看得頭昏眼花。
為了避免自己放棄，乾脆自己來寫一篇，希望不會誤人子弟。

由於剛開始研究 Dagger 的時候是直接從 Android 專案中的範例開始的，殊不知， Dagger 為了迎合 
Android App 的運行方式而用了一些~~旁門左道~~特別的方式來達成依賴注入，使上手難度又再提升一個檔次。

於是這篇文章先不從 Android App 的角度切入，有請 [Dagger](https://google.github.io/dagger/) 官方提供的 [Coffemaker](https://github.com/google/dagger/tree/master/examples/simple/src/main/java/coffee) 範例來現身說法先。

在這篇文中會使用 Kotlin 語言，~~接地氣~~，可以參考看看。

<!-- more -->

# Dependency 依賴的故事

## 做一台咖啡機

現在我們有一台咖啡機 (CoffeeMaker) ，組成的零件有幫浦 (Pump) 、加熱計 (Heater)。

咖啡的製作流程如下：
- 加熱器打開 (Heater on)
- 幫浦打水煮咖啡 (Pumping)
- 咖啡煮好了！
- 加熱器關掉 (Heater off)

```flow
st=>start: 開始
e=>end: 結束
op=>operation: 加熱器打開(Heater on)
op2=>operation: 幫浦打水煮咖啡(Pumping)
op3=>operation: 咖啡煮好了
op4=>operation: 加熱器關掉(Heater off)
cond=>condition: 加熱器是否打開

st->op->cond->op2->op3->op4->e
cond(yes)->op2
cond(no)->op1
```

首先，加熱器長這個樣子，我們選用了電子加熱器：

```kotlin
class ElectricHeater{
    private var heating = false

    fun isHot(): Boolean {
        return heating
    }

    fun off() {
        heating = false
    }

    fun on() {
        println("~ ~ ~ heating ~ ~ ~")
        heating = true
    }
}
```

接著登場的是幫浦，是個虹吸裝置。
這個裝置有防呆功能，防止阿呆忘記加熱就打水。

```kotlin
class Thermosiphon(val heater: ElectricHeater){
    fun pump() {
        if(heater.isHot()){
            println("=> => pumping => =>")
        }
    }
}
```

於是我們的咖啡機長這個樣子：
```kotlin
class Coffeemaker{
    private val heater = ElectricHeater()
    private val pump = Thermosiphon(heater)
    
    fun brew(){
        heater.on()
        pump.pump()
        println(" [_]P coffee! [_]P ")
        heater.off()
    }
}
```

很好，看起來沒什麼問題，故事結束了。

才不。

## 依賴關係

讓我們來分析一下各個零件之間的關係。

<!-- 
```flow
op=>operation: 咖啡機(Coffeemaker)
op2=>operation: 幫浦(Thermosiphon)
op3=>operation: 加熱器(Electric Heater)

op->op2->op3
```
-->

<img src="https://i.imgur.com/MUoGB04.png" width="80%">

咖啡機依賴了幫浦跟加熱器，而幫浦又依賴於加熱器。加熱器沒有幫浦跟咖啡機也可以運作，但咖啡機和幫浦則不能沒有加熱器。

我們會說，咖啡機是個高層模組，依賴於低層模組的幫浦跟加熱器。

回過頭來看看 Coffeemaker 這個 class。當我們要製造 (實例化，Instantiate) 出一台咖啡機的時候，同時也會製造出新的幫浦跟加熱器，屬於一條龍服務。

這樣會有什麼問題呢?
- 狀況一：某個收集咖啡機的愛好者想要擁有很多台咖啡機，但是不想要擁有很多的幫浦跟熱器。但是每一台咖啡機都自帶幫浦跟加熱器，浪費資源。

- 狀況二：廠商在研發咖啡機的時候，想要拿個假的加熱器跟幫浦來試試看咖啡製作流程有沒有錯誤。在一條龍服務下沒辦法在不動 Coffeemaker 配方 (程式碼) 的情況下完成。

- 狀況三：有一天，加熱器大改版，把on方法改成了open，咖啡機必須要配合加熱器跟著改版才能順利運作。

- 狀況四：有一天，歐盟心血來潮，端著反壟斷法來查水表，因為你的機器只接受電子加熱器。

...

從軟體工程的角度來看，我們會說這些組件有著高[相依性](https://en.wikipedia.org/wiki/Coupling_(computer_programming)) (High dependency, high coupling) ，牽一髮而動全身，而且物件無法被重用 (Reuse) 。

好，那我們來改個設計，解決這些問題。

## 解開依賴 (Decouple) - 依賴注入

首先解決狀況一，只要調整一下產線就可以了。

我們將幫浦跟加熱器委外生產，同時將咖啡機模組化。

現在我們的配方變成這樣：

```kotlin
class Coffeemaker(
    private val heater: ElectricHeater, 
    private val pump: Thermosiphon
){
    fun brew(){
        heater.on()
        pump.pump()
        println(" [_]P coffee! [_]P ")
        heater.off()
    }
}
```

很好，現在咖啡機收集狂可以只擁有一組加熱器跟幫浦，但擁有很多台咖啡機了。

```kotlin
class TheManHavingManyCoffeemakers {
    val heater = ElectricHeater()
    val pump = Thermosiphon(heater)
    val cofeemaker1 = Coffeemaker(heater,pump)
    val cofeemaker2 = Coffeemaker(heater,pump)
    ...
}
```

我們也可以設計成透過接口 (setter) 來獲得零件，這裡就不把程式碼秀出來了。

你有沒有發現，其實幫浦跟加熱器彼此之間早就是這種關係啦！如果幫浦跟加熱器組合是一條龍服務的話，我們的咖啡機就做不出來了呢。

如果有一天有個貪心的傢伙想做個有咖啡機功能的大機器也辦的到了。

## 依賴反轉

然後來解決問題二三四。

生產咖啡機的廠商想了想，決定訂個標準規則 (interface) ，規定加熱器跟幫浦的規格，比如說必須要有哪些功能等等。具體如何實現這個功能則不在標準之內。
於是，咖啡機以及其他組件都必須按照這個標準來設計。

```kotlin
interface Heater{
    fun on()
    fun off()
    fun isHot():Boolean
}

interface Pump{
    fun pump()
}
```

咖啡機配方變成這個樣子：
```kotlin
class Coffeemaker(
    private val heater: Heater, 
    private val pump: Pump
){
    fun brew(){
        heater.on()
        pump.pump()
        println(" [_]P coffee! [_]P ")
        heater.off()
    }
}
```
在這裡，所有的來自外部的零件必須符合規範，符合的規定 interface 。

讓我們來看看這樣如何解決問題。

現在有了這份標準，在測試的時候只要根據標準製作假的加熱器跟幫浦，便可以在不更動 Coffemaker 的情形下完成。

並且，任何廠商都可以生產幫浦跟加熱器，並將它用在咖啡機之中，不需要執著於電子加熱器。要換成 
MagicHeater 也不成問題。

```kotlin
class MagicHeater : Heater {
    private var heating = false

    override fun isHot(): Boolean {
        return heating
    }

    override fun off() {
        heating = false
    }

    override fun on() {
        println("~ ~ ~ This is Magic ~ ~ ~")
        println("~ ~ ~ heating ~ ~ ~")
        println("~ ~ ~ magic ~ ~ ~")
        heating = true
    }
}
```

而至於將 on 改成 open 這種改版，由於不符合規則，該廠商將會被業界淘汰。

看起來真不錯！

原本的狀況是，咖啡機必須要配合低層模組來設計。
現在的狀況則是高層模組以及低層模組都必須要配合 interface 來設計。

在軟體工程中，這種方式被稱作依賴反轉、相依性反轉、控制反轉 ... 等等。

## 降低成本、提高安全 (誤)

~~(標題要下什麼才好呀)~~

現在要製造咖啡機，我們最少要擁有三條供應鏈：生產咖啡機、生產幫浦、生產加熱器。
也就是說，每當在不同的地方要生出一台咖啡機時，我們也必須生出至少一組加熱器跟幫浦。

```kotlin
class PeopleInTainan {
    val heater = ElectricHeater()
    val pump = Thermosiphon(heater)
    val cofeemaker = Coffeemaker(heater,pump)
    ...
}

class PeopleInTaipei {
    val heater = ElectricHeater()
    val pump = Thermosiphon(heater)
    val cofeemaker = Coffeemaker(heater,pump)
    ...
}

...

```

好麻煩啊，一直寫重複的 code 稱不上是個好的工程師。
而且寫在建構子讓 Coffemaker 看起來很複雜，有沒有什麼更簡潔的方法？

更糟糕的是，幫浦跟加熱器暴露在外頭，使用者可以隨時取得它們，可以拿去做奇怪的事，例如拿去煮泡麵、燙他討厭的人。 
~~(基德：廠商沒有跟你說微波爐不能拿來烘乾貓咪)~~

我們引入一臺專門的機器 (Injector/Container/...) ，這臺機器負責將咖啡機需要的加熱器以及幫浦生出來並安裝到位。

```kotlin
class Injector{
    private val heater = ElectricHeater()
    private val pump = Thermosiphon(heater)
    
    fun provideHeater() : Heater{
        return heater
    }
    
    fun providePump() : Pump{
        return pump
    }
}
```

```kotlin
class Coffeemaker(injector:Injector)
{    
    private val heater = injector.provideHeater()
    private val pump = injector.providePump()
    
    fun brew(){
        heater.on()
        pump.pump()
        println(" [_]P coffee! [_]P ")
        heater.off()
    }
}
```

當咖啡機的依賴零件需要更換成其他廠商的零件或者是測試用的偽零件時，只要修改或換掉這臺機器就可以了。

有需要的話，也可以為這台機器定個規則 (interface)。

更進一步，還可以把整個製作流程打包起來。

```kotlin
class CoffeeMakerProvider{
    companion object{
        fun provideCoffeeMaker() = Coffeemaker(Injector())
    }
}
```

這樣一來，隨時隨地都可以取得一臺全新的咖啡機！而且內部組件也都被包裝起來不隨便給人用了。

而這些生成實體用的物件，還能視需求搭配單例模式來使用，使得每次提供的加熱器、幫浦或是 Injector 都是同一個，達到重用、節省運算資源。

註： 在這一章中， pump 與 heater 並沒有被解耦，而是一整組直接包進Injector中。需要的話我們可以如法炮製將 pump 與 heater 分開，程式會變得非常的長。


# Dagger 依賴注入框架


為了達到好擴充好維護好測試的目的，我們多寫了很多的程式碼，但自己寫的東西總是自己最明白。

有一天，一名新的工程師被聘進來，身為老鳥的我們必須費盡唇舌的解釋生產線如何運作。

有了依賴注入框架，我們只要把框架的文件丟給他就好了。

現在，來談談 Dagger 吧。

原本的 [Dagger](https://github.com/square/dagger/) 是由square公司所維護的專案，後來被 Google 接手，進化成 [Dagger2](https://github.com/google/dagger)。

我們選用的是 Dagger2 ，這個框架藉由註解 (annotation)，在編譯的時候自動產生代碼來完成依賴注入。

把 Dagger2 加入 Gradle 中吧！
在使用Kotlin的環境中， [Kotlin 官方建議把註解解釋器 (annotation processor) 換成 kapt](https://blog.jetbrains.com/kotlin/2015/06/better-annotation-processing-supporting-stubs-in-kapt/)

```groovy
dependencies {
...
    implementation 'com.google.dagger:dagger:2.17'
    kapt 'com.google.dagger:dagger-compiler:2.17'
...
}
```

## 還是咖啡機的故事

### Inject

首先，我們改造一下需要依賴注入的幫浦以及咖啡機，把需要被注入的物件 (加熱器) 用 `@Inject` 標記起來。

```kotlin
class Thermosiphon @Inject constructor(val heater: Heater) : Pump {
    override fun pump() {
        if(heater.isHot()){
            println("=> => pumping => =>")
        }
    }
}
```

```kotlin
class CoffeeMaker @Inject constructor(private val heater: Heater, private val pump: Pump) {
    fun brew(){
        heater.on()
        pump.pump()
        println(" [_]P coffee! [_]P ")
        heater.off()
    }
}
```

在Dagger2中， `@Inject` 可以用來標記建構子、屬性、以及函式，這裡以標記建構子作為範例。

### Module

接下來建立 Module ，他的腳色相當於前一章 Injector 的腳色，負責產生零件實體。

```kotlin
@Module(includes = [PumpModule::class])
class CoffeeMakerModule {
   @Provides
   fun provideHeater():Heater{
       return ElectricHeater()
   }
}

@Module
abstract class PumpModule{
    @Binds
    abstract fun providePump(pump: Thermosiphon):Pump
}
```

由於 `Thermosiphon` 也依賴於 `Heater` ，因此幫浦的提供方法不使用 `@Provides` ，而是用 `@Binds` 將提供幫浦的方法以及我們選擇的幫浦實作類別 (Thermosiphon) 連結起來。
最後使用 `includes` 將兩個模組連結起來，Dagger會將同一個加熱器同時注入到咖啡機以及幫浦中。

### Component

最後則是產生咖啡機的一條龍產線 Component ，相當於前一章的 CoffeeMakerProvider 。

```kotlin
@Component(modules = [CoffeeMakerModule::class])
interface CoffeeComponent {
    fun provideCoffeeMaker() : CoffeeMaker
}
```

我們只要告訴 Dagger 要使用的模組以及要產生的咖啡機類別就可以了。

### 做咖啡囉

東西都準備好了之後， build 這個專案。 Dagger2 會根據這些註解以及介面，為我們產生相對應的程式碼實作。

讓我們試著做一台咖啡機，用這台咖啡機做一杯咖啡。

```kotlin
fun main(){
    val coffeeProvider = DaggerCoffeeComponent.builder().build()
    val coffeeMaker = coffeeProvider.provideCoffeeMaker()
    coffeeMaker.brew()
}
```

DaggerCoffeeComponent是程式碼產生器自動產生的類別。

執行結果如下：

```
~ ~ ~ heating ~ ~ ~
=> => pumping => =>
 [_]P coffee! [_]P
```

## Dagger2 in Android

咖啡的故事結束了，我們把場景拉回 Android App。

由於 Android 中很多組件的實體是由 Android 系統掌控，我們沒辦法請框架幫我們產生已經注入好的 Activity 並交給 Android 調用。

後來，以下的追加套件就出現了：

``` groovy
dependencies {
...
    implementation 'com.google.dagger:dagger-android:2.17'
    implementation 'com.google.dagger:dagger-android-support:2.17' // if you use the support libraries
    kapt 'com.google.dagger:dagger-android-processor:2.17'
...
}
```

這邊舉一個栗子。簡單的 Activity 注入範例 (我也還在學習中，也許改天會有個續篇吧...改天:no_mouth:)

現在有一個 Activity ，需要一個 ViewModel工廠實體。工廠是什麼東西就先不在此討論。

### Inject

```kotlin
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var viewModelFactory: ViewModelFactory
    
    ...
}
```

由於 Activity 沒有建構子，因此在創造成員物件的時候注入需要的物件。

### Module

跟前一章的方法沒什麼太大的不同，使用ViewModelFactory的建構子來創造實體。

```kotlin
@Module
class MainActivityModule{
    @Provides
    fun provideViewModelFactory() : ViewModelFactory{
        return ViewModelFactory()
    }
}
```

### Component

這裡必須要在定義 modules 時引入 dagger.android 中的組件，`AndroidInjectionModule` ：

而原本輸出已注入實體的方法變成了需要輸入 activity 的方法。

```kotlin
@Component(modules = [
    MainActivityModule::class,
    AndroidInjectionModule::class
])
interface MainActivityComponent{
    fun inject(activity:MainActivity)
}
```
### Run

設定完成，按下 Build 之後獲得 Dagger2 幫我們產生的類別 `DaggerMainActivityComponent` ，用它在 activity 的生命週期中 **自己注入自己** 。

```kotlin
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var viewModelFactory: ViewModelFactory
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // inject what you need
        DaggerMainActivityComponent.builder().build().inject(this)
    }
}
```

在這個生命週期之後，標註 `@Inject` 的物件就會被注入，也就是說有實體可以用了。

### 正確的注入姿勢

以上僅是做個簡單的注入處理。

官方提供的[教學](https://google.github.io/dagger/android)將注入端延伸到了 Application 類別來初始化。
有興趣的看官可以參考看看，看過以上的例子，應該會比較容易看得懂官方的作法。

<!-- # Dagger 以外 -->

# Outro

寫這篇文花了快一個星期的時間，過程中一直在懷疑流程到底對不對，這些的程式碼到底有沒有必要。

也許經驗不足就是會這個樣子。

Dagger2 還有很多其他的功能，像是傳遞生成實體所需要的參數、定 Scope 、設成單例、 MultiBinds 、blabla...，也許改天會有個續篇...改天:no_mouth:




