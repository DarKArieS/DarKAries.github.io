---
title: C++ Lambda Function
date: 2019-03-04 11:10:21
tags: C++
categories: C++
---


今天打開了久違的[LeetCode](https://leetcode.com)，用C++寫了題Easy。

寫完看了看別人的解，突然發現原來C++ 在C++11 也有了lambda function的寫法，寫一篇文以玆紀念。

[Reference](https://zh.wikipedia.org/wiki/C%2B%2B11)

<!-- more -->

## 基礎用法

最簡單的結構長得像這樣：

```C++
[](int x, int y) { return x + y; }
```

在這裡三種括號都用上了，來看看它們都做了些什麼。

- (): Function傳入值
- {}: Function主要功能
- []: Closure的傳值方式


如果要規定function的return類型(例如返回int)，改寫成：
```C++
[](int x, int y)-> int { int z = x+y; return z; }
```

稍微測試了一下，似乎還沒遇到非得指定return類型的情況。
有經驗的高人可以指點一下:stuck_out_tongue:

把它當物件使用時要使用auto關鍵字宣告

```C++
class A {
	public:
	void runLambda(int (*f)(int,int), int x, int y){cout<<f(x,y)<<endl;};
};

void main(){
	std::vector<int> someList(5,1);
	int total = 0;
	auto y = [](int x, int y) { int z = x+y; return z; };
	A a;
	a.runLambda(y,10,5);
}
```

## 閉包 (Closure)

當Lambda function的功能會需要用到外部的變數時，必須將傳入function的方法事先在`[]`中設定好。
有以下幾種組合：
- []：不使用閉包，如果內部有使用其他scope外的變數會報錯
- [=]：全部使用傳值的方式
- [&]：全部使用傳址的方式
- [x, &y]：x變數用傳值，y變數用傳址，如果內部有使用其他未定義的scope外變數會報錯
- [&,x]：除了x使用傳值之外，其他使用傳址
- [=,&x]：除了x使用傳址之外，其他使用傳值

看個範例就可以了解了，範例來自[Reference](https://zh.wikipedia.org/wiki/C%2B%2B11)。

定義特定的值傳址：
```C++
std::vector<int> someList;
int total = 0;
std::for_each(someList.begin(), someList.end(), [&total](int x) {
     total += x;
});
std::cout << total;
```

不定義特定的，全部都傳址：
```C++
std::vector<int> someList;
int total = 0;
std::for_each(someList.begin(), someList.end(), [&](int x) {
    total += x;
});
std::cout << total;
```

如果在物件裡使用，且function內容需要使用物件成員的內部成員，需要傳入this pointer
```C++
class A {
	public:
	int x=0;
	int y=0;
	virtual void runOp(){
		auto f = [this]() { int z = this->x+this->y; return z; };
		cout<<f()<<endl;
	};
};

void main(){
	A a;
	a.x=5;
	a.y=10;
	a.runOp();	
}
```





