---
tags: Rx, C#
title: Rx介紹 Part 3 - Advanced error handling(譯...)
---
>嗯，正在學Rx，學了一輪後實際應用時發現還有不懂的地方，再重讀一次，順便簡單的翻譯下…*翻譯不出來*的或是*覺得不重要*的就以"…"符號替換，或顯示原文。
當然，辭不達義的地方也會有，請包含…
話說，本章中的swallow我翻譯成"吃掉或吞掉"…或者有更文雅的名詞？
另Exception我基本上翻成例外。

##[Advanced error handling](http://www.introtorx.com/Content/v1.0.10621.0/09_SideEffects.html#SideEffects)

例外總是會發生。例外無所謂好或壞，不管我們如何喚起或是捕捉它們。一些例外是可預測的，且通常都是因為粗心的程式碼導致，例如 DivideByZeroException。而一些例外無法被防禦式編程捕獲，例如I/O例外（FileNotFoundException、TimeoutException）。在這些例子中，我們要小心的處理例外，提供使用者錯誤訊息、記錄錯誤或者是重試都是我們處理例外的方法。

The IObserver<T> interface and Subscribe extension methods provide the ability to cater for sequences that terminate in error, however they leave the sequence terminated. They also do not offer a composable way to cater for different Exception types. A functional approach that enables composition of error handlers, allowing us to remain in the monad, would be more useful. Again, Rx delivers.
`IObserver<T>`介面及訂閱的擴充函式提供我們處理序列因錯誤而中斷的方法，然而它們會導致序列中斷，也沒有提供一個可組合的方式以處理不同的例外類型。一個函數式的目標會更有用，它讓我們可以組合錯誤處理函式，讓我們保持在monad的狀態。再次的說，Rx提供了這類處理。

##Control flow constructs
我們將使用marble圖來研究以不同的方式處理不同的控制流程，與正常的.NET程式一樣，我們有如同 try/catch/finally 的流程控制結構；在本章中，我們會看到它們如何用於可觀察序列中。

###Catch
就像SEH(結構式例外處理)一樣，用Rx你可以選擇把例外吞掉、包成另一個例外或執行其它的處理。

我們已經知道可觀察序列可以用OnError架構來處理錯誤狀況。在Rx中，一個稱為 Catch 的擴充函式可以用來處理OnError的訊息推送，它讓你可以攔截一個特定的例外型別並接續其它的序列。

下面是catch覆載的定義：
```csharp
public static IObservable<TSource> Catch<TSource>(
	this IObservable<TSource> first, 
	IObservable<TSource> second)
{
	...
}
```
###Swallowing exceptions
With Rx, you can catch and swallow exceptions in a similar way to SEH. It is quite simple; we use the Catch extension method and provide an empty sequence as the second value.
用Rx，你可以像SEH一樣捕捉並吞掉例外，很簡單 – 我們使用 Catch 擴充函式並提供一個空序列當做第二個值。

我們用marble圖呈現一個被吃掉的例外：
```dos
S1--1--2--3--X
S2            -|
R --1--2--3----|
```
S1代表第一個序列，它以一個錯誤(X)結束，S2是一個空的接續序列，R是以S1開始，在S1中斷後接續S2的結果序列。
```csharp
var source = new Subject<int>();
var result = source.Catch(Observable.Empty<int>());
result.Dump("Catch");
source.OnNext(1);
source.OnNext(2);
source.OnError(new Exception("Fail!"));
```
輸出：
```dos
Catch-->1
Catch-->2
Catch completed
```
上述範例會捕捉並吃掉所有類型的例外，這跟下列SEH相同：
```csharp
try
{
	DoSomeWork();
}
catch
{
}
```
就如同一般在SEH中會做的，在Rx中你也會想指定要吃掉的例外型別。你也許想處理一個特定的型別，正好Catch有一個可讓你指定特定型別的覆載，如同下列範例中，你想捕捉一個TimeoutException：
```csharp
try
{
	//
}
catch (TimeoutException tx)
{
	//
}
```
Rx提供一個Catch的覆載以處理這種情況：
```csharp
public static IObservable<TSource> Catch<TSource, TException>(
	this IObservable<TSource> source, 
	Func<TException, IObservable<TSource>> handler) 
	where TException : Exception
{
	//...
}
```
以下Rx程式讓你可以捕捉TimeoutException例外。我們提供了一個接受例外並回傳序列的函式，讓你不用提供第二個序列當做參數，且可用工廠方法來建立你的continuation。下列範例中，我們在錯誤序列中加入一個-1的值並結束它。
```csharp
var source = new Subject<int>();
var result = source.Catch<int, TimeoutException>(tx=>Observable.Return(-1));
result.Dump("Catch");
source.OnNext(1);
source.OnNext(2);
source.OnError(new TimeoutException());
```
輸出：
```dos
Catch-->1
Catch-->2
Catch-->-1
Catch completed
```
如果序列以無法被轉換為TimeoutException的例外結束，則錯誤不會被捕捉，並且將被推送至訂閱者。
```csharp
var source = new Subject<int>();
var result = source.Catch<int, TimeoutException>(
	tx => Observable.Return(-1));
result.Dump("Catch");
source.OnNext(1);
source.OnNext(2);
source.OnError(new ArgumentException("Fail!"));
```
輸出：
```dos
Catch-->1
Catch-->2
Catch failed-->Fail!
Finally
```
類似SEH中的finally語法，Rx提供了在序列結束時執行程式碼的功能(不管它如何結束)。Finally擴充函式接受一個Action作為參數。不管序列是正常或錯誤地結束，或者訂閱被disposed，都會呼叫此Action。
```csharp
public static IObservable<TSource> Finally<TSource>(
	this IObservable<TSource> source, 
	Action finallyAction)
{
	...
}
```
在這個範例中，我們有一個正常完成的序列。我們提供一個Action，並看到它在我們的OnCompleted後被執行。
```csharp
var source = new Subject<int>();
var result = source.Finally(() => 
	Console.WriteLine("Finally action ran"));
result.Dump("Finally");
source.OnNext(1);
source.OnNext(2);
source.OnNext(3);
source.OnCompleted();
```
輸出：
```dos
Finally-->1
Finally-->2
Finally-->3
Finally completed
Finally action ran
```
相同地，來源序列可能被一個例外終止。這種情況下，例外會被推送至console，而我們提供的委託會被執行。

或者，我們可以取消我們的訂閱。在下一個範例中，我們看到即使序列未完成，Finally函式也會被呼叫。
```csharp
var source = new Subject<int>();
var result = source.Finally(() => 
	Console.WriteLine("Finally"));
var subscription = result.Subscribe(
	Console.WriteLine,
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
source.OnNext(1);
source.OnNext(2);
source.OnNext(3);
subscription.Dispose();
```
輸出：
```dos
1
2
3
Finally
```
注意，在當前的實作中有一個不正常的地方，如果沒有提供OnError處理程序，錯誤將被示為例外並拋出。這將在Finally函式被呼叫之前完成。我們可以通過從上面的示例中刪除OnError處理程序來輕鬆重現這種行為。
```csharp
var source = new Subject<int>();
var result = source.Finally(() => Console.WriteLine("Finally"));
result.Subscribe(
Console.WriteLine,
//Console.WriteLine,
() => Console.WriteLine("Completed"));
source.OnNext(1);
source.OnNext(2);
source.OnNext(3);
//Brings the app down. Finally action is not called.
source.OnError(new Exception("Fail"));
```
希望這被標識為一個bug，並在你閱讀下一個版本的Rx時被修復。出於學術興趣，這裡有一個Finally擴充函式的示範，它將按預期工作。(譯者注：此錯誤已被修正)
```csharp
public static IObservable<T> MyFinally<T>(
	this IObservable<T> source, 
	Action finallyAction)
{
	return Observable.Create<T>(o =>
	{
		var finallyOnce = Disposable.Create(finallyAction);
		var subscription = source.Subscribe(
			o.OnNext,
			ex =>
			{
				try { o.OnError(ex); }
				finally { finallyOnce.Dispose(); }
			},
			() =>
			{
				try { o.OnCompleted(); }
				finally { finallyOnce.Dispose(); }
			});
		return new CompositeDisposable(subscription, finallyOnce);
		});
	}
```
###Using
Using工廠方法允許你將資源的生命週期綁定到可觀察序列的生命週期。函式定義本身需要兩個工廠方法；一個提供資源，一個提供序列。這允許一切都被lazily evaluated。
```csharp
public static IObservable<TSource> Using<TSource, TResource>(
	Func<TResource> resourceFactory, 
	Func<TResource, IObservable<TSource>> observableFactory) 
	where TResource : IDisposable
{
	...
}
```
當你訂閱序列時，Using函式將呼叫這兩個工廠函式。當序列正常結束、被錯誤結束或訂閱被取消時，資源也會被disposed。

為了提供範例，我們要重新介紹第三章講過的TimeIt類別，可以使用這個方便的小類別去計算訂閱的經過時間。下個範例中，我們建立一個使用Using工廠函式的可觀察序列，並提供一個工廠函式給TimeIt資源及一個回傳序列的函式。
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1));
var result = Observable.Using(
	() => new TimeIt("Subscription Timer"),
	timeIt => source);
result.Take(5).Dump("Using");
```
輸出：
```dos
Using-->0
Using-->1
Using-->2
Using-->3
Using-->4
Using completed
Subscription Timer took 00:00:05.0138199
```
由於Take(5) decorator，序列在5個元素後完成，訂閱因此被取消；與此同時，TimeIt資源會被disposed，因此記錄經過時間的函式也被喚起。

這種機制可以在有想像力的開發者手中找到各種實際應用。資源做為IDisposable型別是很方便的；事實上，它使得許多類型的資源可以被綁定，例如其他訂閱、串流讀取器/寫入器、資料庫連接及使用者控制等，且再與Disposable.Create(Action)合用幾乎可以做到任何事。

###OnErrorResumeNext
Just the title of this section will send a shudder down the spines of old VB developers! In Rx, there is an extension method called OnErrorResumeNext that has similar semantics to the VB keywords/statement that share the same name. This extension method allows the continuation of a sequence with another sequence regardless of whether the first sequence completes gracefully or due to an error. Under normal use, the two sequences would merge as below:
```dos
S1--0--0--|
S2        --0--|
R --0--0----0--|
```
即使第一個序列中發生錯誤，結果序列也會被合成：
```dos
S1--0--0--X
S2        --0--|
R --0--0----0--|
```
OnErrorResumeNext的覆載如下：
```csharp
public static IObservable<TSource> OnErrorResumeNext<TSource>(
	this IObservable<TSource> first, 
	IObservable<TSource> second)
{
	...
}
public static IObservable<TSource> OnErrorResumeNext<TSource>(
	params IObservable<TSource>[] sources)
{
	...
}
public static IObservable<TSource> OnErrorResumeNext<TSource>(
	this IEnumerable<IObservable<TSource>> sources)
{
	...
}
```
使用上很簡單；你可以用這些覆載傳入你想要的任何數量的序列，但當然要有個數限制，如同在VB中對OnErrorResumeNext這個關鍵字的警示說明，所以在Rx中要謹慎使用它。它悄悄地將例外吃掉，這可能會導致你的軟體處在未知的狀態中。一般來說這會讓你的軟體更難維護及除錯。

###Retry
如果你知道你的序列將遇到已知的可能問題時，你可能只想讓動作重試。這樣的例子就如同在執行一個I/O(例如web request或磁碟存取)動作時，你會想再試一次，因I/O常可能會有這樣的間歇性故障發生。Retry擴充函式讓你可以在錯誤發生時以指定的次數重試或重試到成功為止。
```csharp
//Repeats the source observable sequence until it successfully terminates.
public static IObservable<TSource> Retry<TSource>(
	this IObservable<TSource> source)
{
	...
}
// Repeats the source observable sequence the specified number of times or until it 
//  successfully terminates.
public static IObservable<TSource> Retry<TSource>(
	this IObservable<TSource> source, int retryCount)
{
	...
}
```
下列marble圖中，來源序列(S)產生值然後錯誤，再產生再出錯；總共兩次，結果序列(R)是對來源序列(S)訂閱後所有成功的值的組合。
```dos
S --1--2--X
            --1--2--3--X
                         --1
R --1--2------1--2--3------1
```
下個範例中，我們使用當任何例外發生時永遠重試的覆載。
```csharp
public static void RetrySample<T>(IObservable<T> source)
{
	source.Retry().Subscribe(t=>Console.WriteLine(t)); //Will always retry
	Console.ReadKey();
}
```
給予來源[0,1,2,X]，輸出為：
```dos
0
1
2
0
1
2
0
1
2
```
This output would continue forever, as we throw away the token from the subscribe method. As a marble diagram it would look like this:
```dos
S--0--1--2--x
             --0--1--2--x
                         --0--
R--0--1--2-----0--1--2-----0--
```
另外，我們可以指定最大重試次數。在這個範例中，我們只重試一次，然後第二個訂閱推送的錯誤會被傳至最後的訂閱中。注意雖然只重試一次但你傳入的值是2，也許這個函式應該被命名為"Try"？
```csharp
source.Retry(2).Dump("Retry(2)"); 
```
輸出：
```dos
Retry(2)-->0
Retry(2)-->1
Retry(2)-->2
Retry(2)-->0
Retry(2)-->1
Retry(2)-->2
Retry(2) failed-->Test Exception
```
Marble圖看起來會像：
```dos
S--0--1--2--x
             --0--1--2--x
R--0--1--2-----0--1--2--x
```
在使用永遠重覆的覆載時要小心。很顯然地如果在你的序列中有一個一直存在的錯誤，你會發現自己卡在無窮迴圈中。另外，注意目前沒有一個可以讓你指定特定例外型別的Retry覆載函式。

一個有用的可讓你加入自己的函式庫的擴充函式可能是"Back off and Retry"函式。我的團隊發現這樣的功能在執行I/O，特別是網路請求時很有用。概念是執行，並且在失敗後等待一段時間，然後再執行。你自己的版本可能要考慮需重試的例外類型及可重試的最大次數，你可能甚至想在每次重試時延長等待時間。

例外管理的需求比一般的OnError處理程序還多是司空見慣的。Rx提供了基本的例外處理運算子，你可以使用這些來組合更複雜且強健的查詢。本章中我也介紹了Rx中進階的例外處理，及一些資源管理功能。我們瞭解了Catch、Finally和Using等函式，及其它如OnErrorResumeNext和Retry函式，這些讓你可以玩一下'fast and loose'。我們還重新使用marble圖來幫助我們對多個序列的組合的視覺化。這將幫助我們學習在下一章要看的組合和聚合可觀察序列的其它函式。
> Written with [StackEdit](https://stackedit.io/).