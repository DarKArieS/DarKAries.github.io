---
title: 執行緒安全 Double-checked Locking
date: 2019-04-18 10:51:10
tags: 
- Java
- Kotlin
categories: Design pattern
---

有一天，在Google提供的[範例](https://github.com/googlesamples/android-Notifications)中讀到一段程式碼，當下不是很理解為什麼要這樣寫。

這是一段關於懶人~~丹利~~單例生成實例的程式碼：

<!-- more -->

```Java
public static class InboxStyleEmailAppData extends MockNotificationData {
    private static InboxStyleEmailAppData sInstance = null;
    
    ...
    
    public static InboxStyleEmailAppData getInstance() {
        if (sInstance == null) {
            sInstance = getSync();
        }
        return sInstance;
    }

    private static synchronized InboxStyleEmailAppData getSync() {
        if (sInstance == null) {
            sInstance = new InboxStyleEmailAppData();
        }

        return sInstance;
    }
    
    ...
}
```

看完後覺得奇怪，產生實體只要靠getSync()就可以完成了，為什麼還要再加上一層getInstance()並且再次確認靜態成員是否為null ?

經過高人提示，關鍵字為雙重鎖，立刻餵狗。

## Double-checked Locking

然後發現自己其實以前有讀過[這一篇](https://blog.csdn.net/fly910905/article/details/79286680)，顯然沒有記到腦子裡。

根據[Wiki](https://en.wikipedia.org/wiki/Double-checked_locking)，確保線程安全的單例的確只需要getSync()來生成或取得實體就可以了。
synchronized關鍵字確保當兩個執行緒同時在實體尚未生成時使用該方法，不會同時生成兩個實體，使得其中一個執行緒拿不到正確的sInstance成員。

但是如果每次要獲取單例實體時都使用帶有synchronized關鍵字的方法，系統必須要重複執行上鎖/解鎖的程序。當單例實體已經被生成後，這個鎖就顯得很不必要，且會降低系統效能。因此在外層又再加上一層不帶synchronized的null check函式，當單例實體存在時就不會再經過上鎖/解鎖的程序了。

## 一點點的缺陷

看起來萬無一失，但故事還沒完。

有一天，執行緒A呼叫了單例方法，他注意到了沒有單例實體，於是繼續呼叫getSync()來生成實體。
在執行緒A還在忙的時候，執行緒B來了。B發現A已經來過了，於是很放心的把單例拿出去用了。

糟糕的是，A其實這時候還沒有忙完，B拿到的是不成熟的，還沒做好的實體(partially constructed object)，於是B並沒辦法真正的使用它，只好躺在地上死給你看(Crash)。

以上的故事，根據編譯器、快取機制等等各種因素，並不一定每次都會發生。

## volatile 關鍵字

據說[^clipAnimation]在某個java版本以後，擴展了語意的volatile關鍵字加入了happens-before relationship的機制，可以解決這個問題。

volatile的原意為當每次要使用該變數時都會讀取當下的值，而不是使用快取的值。而happens-before relationship更進一步確立每次讀寫的順序，類似mutex的機制。

那要如何使用呢?
只要將sInstance宣告加入volatile就可以了
```Java
private volatile static InboxStyleEmailAppData sInstance = null;
```

另外還可以在函式中加入Local variable，來降低存取volatile物件的次數，據說可以提升25%的效能。

像是Wiki中舉的例子:
```Java
// Works with acquire/release semantics for volatile in Java 1.5 and later
// Broken under Java 1.4 and earlier semantics for volatile
class Foo {
    private volatile Helper helper;
    public Helper getHelper() {
        Helper localRef = helper;
        if (localRef == null) {
            synchronized (this) {
                localRef = helper;
                if (localRef == null) {
                    helper = localRef = new Helper();
                }
            }
        }
        return localRef;
    }
    // other functions and members...
}
```

存取volatile成員只在第一次localRef賦值的時候發生。

volatile關鍵字的作用似乎在每個不同的語言有不同的功能，像是C語言就不同於Java。老實說，我讀了半天，除了C語言的volatile會強調移除編譯器最佳化的功能外，我還是沒辦法強烈的感受到java與c兩者之間的不同，也許要懂底層的實作才有辦法了解，不過先在此打住好了。

以上純屬看著維基嘴砲，並無實驗佐證。
有任何錯誤歡迎提出:smiley:

[^clipAnimation]: 據說是J2SE 5.0，但要確切的版本得去翻oracle的Doc，先放棄好了。參見wiki: [Double-checked_locking](https://en.wikipedia.org/wiki/Double-checked_locking)、[Volatile](https://en.wikipedia.org/wiki/Volatile_(computer_programming))

# 補充

在Google提供的[範例](https://github.com/googlesamples/android-architecture-components/blob/master/BasicRxJavaSampleKotlin/app/src/main/java/com/example/android/observability/persistence/UsersDatabase.kt)中看到使用Kotlin的寫法，可以跟上面比較一下。

```kotlin

@Database(entities = arrayOf(User::class), version = 1)
abstract class UsersDatabase : RoomDatabase() {

    abstract fun userDao(): UserDao

    companion object {

        @Volatile private var INSTANCE: UsersDatabase? = null

        fun getInstance(context: Context): UsersDatabase =
                INSTANCE ?: synchronized(this) {
                    INSTANCE ?: buildDatabase(context).also { INSTANCE = it }
                }

        private fun buildDatabase(context: Context) =
                Room.databaseBuilder(context.applicationContext,
                        UsersDatabase::class.java, "Sample.db")
                        .build()
    }
}

```