---
tags: Rx, C#
title: Rx介紹 Part 2 - Transformation of sequences(譯ing...)
---

##[Transformation of sequences](http://www.introtorx.com/Content/v1.0.10621.0/08_Transformation.html#TransformationOfSequences)
我們消費的序列中的值並不總是我們需要的格式。有時在數據中有太多的雜訊，所以我們要將雜訊去掉。有時，每個值都需要擴展進更豐富的物件中或更多的值。通過組合運算子，Rx允許您控制所消耗的可觀察序列中的值的質量以及數量。

到目前為止，我們已經學習了序列的建立，將值轉移至序列，以及通過過濾，聚合或fold來縮減序列。在本章中，我們將討論序列的轉換。這讓我們得以介紹我們的第三種函數式方法，bind。Rx中的bind函數對序列中的每個元素應用一組轉換以產生新序列。回顧一下：
**Ana(morphism) T --> `IObservable<T>`**
**Cata(morphism) `IObservable<T>` --> T**
**Bind `IObservable<T1>` --> `IObservable<T2>`**

現在我們已經介紹了所有的三個higher order函數，你可能會發現你已經知道他們。Bind和Cata（morphism）由來自Google的MapReduce框架而著名。這裡Google通過他們或許更常見的別名來引用Bind和Cata；Map 及 Reduce。

記住higher order functions的**ABCs**代表的意思，它是很好的助憶碼。
**A** na 進入序列，T --> `IObservable<T>`
**B** ind 修改序列，`IObservable<T1>` --> `IObservable<T2>`
**C** ata 離開序列， `IObservable<T>` --> T

###Select
經典的轉換函式是Select。它允許你提供一個接受TSource的值並返回TResult的值的函式。Select的函式定義良好且簡單，並且建議其最常見的用法是從一種類型轉換到另一種類型，即`IObservable <TSource>`到`IObservable <TResult>`。
```csharp
IObservable<TResult> Select<TSource, TResult>(
	this IObservable<TSource> source, 
	Func<TSource, TResult> selector)
```
注意，並沒有任何限制說TSource和TResult不能相同。 所以對於我們的第一個範例，我們將用一個整數序列，並對每個值用加3的方式來轉換，以產生另一個整數序列。
```csharp
var source = Observable.Range(0, 5);
source.Select(i=>i+3)
	.Dump("+3")
```
輸出：
```dos
+3-->3
+3-->4
+3-->5
+3-->6
+3-->7
+3 completed
```
雖然這可能很有用，但更常見的用途是將值從一種型別轉換為另一種型別。在這個例子中，我們將整數值轉換為字元。
```csharp
Observable.Range(1, 5)
	.Select(i =>(char)(i + 64))
	.Dump("char");
```
輸出：
```dos
char-->A
char-->B
char-->C
char-->D
char-->E
char completed
```
如果我們真的想利用LINQ，我們可以將我們的整數序列轉換為一個匿名型別序列。
```csharp
Observable.Range(1, 5)
	.Select(i => 
		new 
		{ 
			Number = i, 
			Character = (char)(i + 64) 
		})
	.Dump("anon");
```
輸出：
```dos
anon-->{ Number = 1, Character = A }
anon-->{ Number = 2, Character = B }
anon-->{ Number = 3, Character = C }
anon-->{ Number = 4, Character = D }
anon-->{ Number = 5, Character = E }
anon completed
```
為了更進一步利用LINQ，我們可以使用query comprehension syntax編寫上述查詢。
```csharp
var query = 
		from i in Observable.Range(1, 5)
		select new 
		{
			Number = i, 
			Character = (char) (i + 64)
		};
query.Dump("anon");
```
在Rx中，Select有另一個覆載。第二個覆載為選擇器函數提供兩個參數。附加參數是序列中元素的索引。如果序列中的元素的索引對於selector function很重要，請使用此方法。

###Cast and OfType
如果你取得的是一個object序列，例如`IObservable <object>`，你可能會發現它不太有用。有一個專門針對`IObservable <object>`的方法，它將每個元素轉換為給定的型別，並在邏輯上將其稱為`Cast <T>()`。
```csharp
var objects = new Subject<object>();
objects.Cast<int>().Dump("cast");
objects.OnNext(1);
objects.OnNext(2);
objects.OnNext(3);
objects.OnCompleted();
```
輸出：
```dos
cast-->1
cast-->2
cast-->3
cast completed
```
然而，如果我們要添加一個不能被轉換成序列的值(型別不符)，那麼我們會得到錯誤。
```csharp
var objects = new Subject<object>();
objects.Cast<int>().Dump("cast");
objects.OnNext(1);
objects.OnNext(2);
objects.OnNext("3");//Fail
```
輸出：
```dos
cast-->1
cast-->2
cast failed -->Specified cast is not valid.
```
幸運的是，如果這不是我們想要的，我們可以使用替代的擴充函式`OfType <T>()`。
```csharp
var objects = new Subject<object>();
objects.OfType<int>().Dump("OfType");
objects.OnNext(1);
objects.OnNext(2);
objects.OnNext("3");//Ignored
objects.OnNext(4);
objects.OnCompleted();
```
輸出：
```dos
OfType-->1
OfType-->2
OfType-->4
OfType completed
```
公平地說，雖然這些函式很方便，但我們可以使用我們已知的運算子來建立它們。
```csharp
//source.Cast<int>(); is equivalent to
source.Select(i=>(int)i);
//source.OfType<int>();
source.Where(i=>i is int).Select(i=>(int)i);
```
###Timestamp and TimeInterval
由於可觀察序列是非同步的，因此可以很方便地知道接收到元素的時間。Timestamp擴充函式是一種方便的函式，它將序列的元素包裹在輕量Timestamp結構T中。 `Timestampe<T>`型別是一個結構，它公開了它所包裝的元素的值，以及使用DateTimeOffset建立的timestamp。

在這個例子中，我們每隔一秒建立一個包含三個值的序列，然後將其轉換為帶Timestamp的序列。 ToString()在`Timestampe<T>`上的實做給了我們一個易讀的輸出。
```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
	.Take(3)
	.Timestamp()
	.Dump("TimeStamp");
```
輸出：
```dos
TimeStamp-->0@01/01/2012 12:00:01 a.m. +00:00
TimeStamp-->1@01/01/2012 12:00:02 a.m. +00:00
TimeStamp-->2@01/01/2012 12:00:03 a.m. +00:00
TimeStamp completed
```
我們可以看到，值0、1和2每隔一秒產生。取得絕對timestamp的另一種方法是僅獲取自最後一個元素以來的間隔。TimeInterval擴充函式提供了這個。根據timestamp函式，元素被包裹在輕量結構中。 這個時候的結構是型別`TimeInterval <T>`。
```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
	.Take(3)
	.TimeInterval()
	.Dump("TimeInterval");
```
Output:
```dos
TimeInterval-->0@00:00:01.0180000
TimeInterval-->1@00:00:01.0010000
TimeInterval-->2@00:00:00.9980000
TimeInterval completed
```
正如你可以從輸出中看到的，間隔不是正好一秒鐘，但是非常接近。
###Materialize and Dematerialize
Timestamp和TimeInterval變換運算子可以證明對記錄和除錯序列有用，Materialize運算子也是如此。Materialize將序列轉換為序列的metadata 代表，將`IObservable<T>`轉換為`IObservable<Notification<T>>`。通知型別提供序列事件的metadata。

如果我們對一個序列做materialize，我們可以看到回傳的包裝的值。
```csharp
Observable.Range(1, 3)
	.Materialize()
	.Dump("Materialize");
```
輸出：
```dos
Materialize-->OnNext(1)
Materialize-->OnNext(2)
Materialize-->OnNext(3)
Materialize-->OnCompleted()
Materialize completed
```
注意，當來源序列完成時，materialized的序列產生一個“OnCompleted”推送值，然後完成。`Notification<T>`是一個具有三個實做的抽像類：

- OnNextNotification
- OnErrorNotification
- OnCompletedNotification

`Notification<T>`公開了四個公開屬性，以讓你知道它：Kind、HasValue、Value和Exception。顯然只有OnNextNotification會為HasValue返回true，並且有一個有用的Value實做。還應當明顯的是，OnErrorNotification是唯一的具有Exception值的實做。Kind屬性回傳一個列舉，應該夠你知道有哪些方法適合使用。
```csharp
public enum NotificationKind
{
	OnNext,
	OnError,
	OnCompleted,
}
```
在下一個例子中，我們產生一個有錯誤的序列。注意，序列的最終值是OnErrorNotification。此外，這一個materialized序列沒有錯誤，它成功完成。
```csharp
var source = new Subject<int>();
source.Materialize()
	.Dump("Materialize");
source.OnNext(1);
source.OnNext(2);
source.OnNext(3);
source.OnError(new Exception("Fail?"));
```
輸出：
```dos
Materialize-->OnNext(1)
Materialize-->OnNext(2)
Materialize-->OnNext(3)
Materialize-->OnError(System.Exception)
Materialize completed
```
對序列Materializing的實做對於執行序列的分析或記錄非常方便。你可以通過應用Dematerialize擴充函式以解開被materialized的序列。Dematerialize只在`IObservable<Notification<TSource>>`上有用。

###SelectMany
在上面的轉換運算子中，我們可以看到Select是最有用的。它在其變換輸出中非常的有彈性，甚至可以用於再現一些其他的變換運算子。然而SelectMany函式甚至更強大。在LINQ以及Rx中，bind函式是SelectMany。大多數其他轉換運算子可以使用SelectMany建立。考慮到這一點，可以認為SelectMany可能是LINQ中最被誤解的方法之一。

在我個人理解的Rx，我很艱難的學習SelectMany擴充函式。我的一個同事建議我把它想成**"from one, select many"**，這幫我更好的理解SelectMany。一個更好的定義是，**"From one, select zero or more"**。如果我們看SelectMany的函式定義，可看到它需要一個來源序列和一個函數作為其參數。
```csharp
IObservable<TResult> SelectMany<TSource, TResult>(
	this IObservable<TSource> source, 
	Func<TSource, IObservable<TResult>> selector)
```
選擇器參數是一個取單一T值並返回一個序列的函式。請注意，選擇器回傳的序列不必與來源型別相同。最後，SelectMany回傳型別與選擇器回傳型別相同。

如果你希望有效地使用Rx，瞭解這個函式是非常重要的，所以讓我們慢慢地來，同樣重要的是注意它與`IEnumerable<T>`的SelectMany運算子的些微差別，我們很快就會看到。

我們的第一個範例將採用單一值'3'的序列。我們提供的選擇器函數將產生另一個帶有數字的序列。該結果序列將是從1到所提供的值即3的範圍。因此，我們取序列[3]並從我們的選擇器函數返回序列[1,2,3]。
```csharp
Observable.Return(3)
	.SelectMany(i => Observable.Range(1, i))
	.Dump("SelectMany");
```
Output:
```dos
SelectMany-->1
SelectMany-->2
SelectMany-->3
SelectMany completed
```
如果我們將源碼修改為[1,2,3]的序列，就像這樣...
```csharp
Observable.Range(1,3)
	.SelectMany(i => Observable.Range(1, i))
	.Dump("SelectMany");
```
...我們現在得到一個輸出，每個序列（[1]，[1,2]和[1,2,3]）的結果被平展以產生[1,1,2,1,2,3] 。
```dos
SelectMany-->1
SelectMany-->1
SelectMany-->2
SelectMany-->1
SelectMany-->2
SelectMany-->3
SelectMany completed
```
最後一個範例更好地說明了SelectMany如何獲取單個值並將其擴展為多個值。當我們將其應用於值序列時，結果是每個子序列被組合以產生最終序列。 在這兩個範例中，我們返回了一個與來源類型相同的序列。這不是一個限制，所以在下一個範例中，我們回傳一個不同的類型。我們將重用將整數轉換為ASCII的Select範例。為此，選擇器函數僅回傳具有單一值的char序列。
```csharp
Func<int, char> letter = i => (char)(i + 64);
Observable.Return(1)
	.SelectMany(
		i => Observable.Return(letter(i)))
	.Dump("SelectMany");
```
因此，輸入[1]，我們回傳序列[A]。
```dos
SelectMany-->A
SelectMany completed
```
擴展來源序列以具有許多值，將給我們帶有許多值的結果。
```csharp
Func<int, char> letter = i => (char)(i + 64);
Observable.Range(1,3)
	.SelectMany(i => Observable.Return(letter(i)))
	.Dump("SelectMany");
```
現在，[1,2,3]的輸入產生[[A]，[B]，[C]]，其被平展為僅存[A，B，C]。
```dos
SelectMany-->A
SelectMany-->B
SelectMany-->C
```
注意，我們有效地重新建立了Select運算子。

最後一個範例將數字對映至字元。由於只有26個字母，忽略大於26的值是很好的且很容易做到。雖然我們必須為來源的每個元素返回一個序列，但沒有任何規則阻止它是一個空序列。在這種情況下，如果元素值是在範圍1-26之外的數字，我們返回一個空序列。
```csharp
Func<int, char> letter = i => (char)(i + 64);
Observable.Range(1, 30)
	.SelectMany(i =>
		{
			if (0 < i && i < 27)
			{
				return Observable.Return(letter(i));
			}
			else
			{
				return Observable.Empty<char>();
			}
		})
	.Dump("SelectMany");
```
輸出：
```dos
A
B
C
...
X
Y
Z
Completed
```
要清楚，對於來源序列[1..30]，值1產生序列[A]，值2產生序列[B]，依此類推，直到值26產生序列[Z]。當來源產生值27時，選擇器函數返回空序列[]。值28,29和30也產生空序列。一旦來自對選擇器的呼叫的所有序列已經被延展以產生最終結果，我們最終得到序列[A..Z]。

現在我們已經瞭解了我們三個higher order函數中的第三個函數，讓我們花時間來思考我們已經學習的一些函式。首先我們可以考慮Where擴充函式。我們首先在縮減序列的章節中看到這個方法。雖然這種方法確實縮減了一個序列，但它不適合functional fold，因為結果仍然是一個序列。考慮到這一點，我們發現Where實際上是bind的一個fit。作為練習，嘗試使用SelectMany運算符寫自己的擴充函式版本。查看最後一個範例以獲得一些幫助...

使用SelectMany寫的Where的擴充函式範例：
```csharp
public static IObservable<T> Where<T>(this IObservable<T> source, Func<T, bool> predicate)
{
return source.SelectMany(
	item =>
	{
		if (predicate(item))
		{
			return Observable.Return(item);
		}
		else
		{
			return Observable.Empty<T>();
		}
	});
}
```
現在我們知道我們可以使用SelectMany來建立Where函式，它應該是一個很自然的過程，我們可以擴充這個以重現其他過濾器，如Skip和Take。

作為另一個練習，嘗試使用SelectMany編寫您自己的Select擴充函式版本。如果你需要一些幫助，參考我們使用SelectMany將int值轉換為char值的範例...

使用SelectMany編寫的Select擴充函式的範例：
```csharp
public static IObservable<TResult> MySelect<TSource, TResult>(
	this IObservable<TSource> source, 
	Func<TSource, TResult> selector)
{
	return source.SelectMany(
		value => Observable.Return(selector(value)));
}
```
### IEnumerable<T> vs. IObservable<T> SelectMany
值得注意的是，`IEnumerable<T>` SelectMany和`IObservable <T>` SelectMany的實做之間的區別。考慮`IEnumerable <T>`序列是基於pull和blocking的。這意味著當使用SelectMany處理`IEnumerable <T>`時，它會一次向選擇器函數傳遞一個項目，並等待它在從來源請求（pull）下一個值之前處理來自選擇器的所有值 。

考慮一個[1,2,3]的`IEnumerable <T>`來源序列。如果我們使用返回[x * 10，（x * 10）+1，（x * 10）+2]序列的SelectMany運算子來處理，我們將得到[[10,11,12]，[20 ，21,22]，[30,31,32]]。

```csharp
private IEnumerable<int> GetSubValues(int offset)
{
	yield return offset * 10;
	yield return (offset * 10) + 1;
	yield return (offset * 10) + 2;
}
```
然後我們使用以下程式應用GetSubValues函式：
```csharp
var enumerableSource = new [] {1, 2, 3};
var enumerableResult =
	enumerableSource.SelectMany(GetSubValues);
foreach (var value in enumerableResult)
{
	Console.WriteLine(value);
}
```
所得到的子序列被展開[10,11,12,20,21,22,30,31,32]：
```dos
10
11
12
20
21
22
30
31
32
```
與`IObservable <T>`序列的區別在於，對SelectMany的選擇器函數的呼叫不會阻塞，結果序列可以隨時間產生值。這意味著後續的“子”序列可以重疊。 讓我們再次考慮一個[1,2,3]的序列，但是這個時間值是相隔三秒產生的。 選擇器函數也將按照上面的例子產生[x * 10，（x * 10）+1，（x * 10）+2]的序列，然而這些值將相隔4秒。

為了可視化這種非同步數據，我們需要空間和時間的表示。

###Visualizing sequences
Let's divert quickly and talk about a technique we will use to help communicate the concepts relating to sequences. Marble diagrams are a way of visualizing sequences. Marble diagrams are great for sharing Rx concepts and describing composition of sequences. When using marble diagrams there are only a few things you need to know

a sequence is represented by a horizontal line
time moves to the right (i.e. things on the left happened before things on the right)
notifications are represented by symbols:
讓我們快速轉向並談談我們用來幫助傳達與序列相關的概念的技術。Marble 圖是一種可視化序列的方法。Marble 圖是共享Rx概念和描述序列的組成的好方法。當使用Marble 圖時，只有幾個事情你需要知道：

1. 序列由水平線表示
1. 時間向右移動（即左邊的事情發生在右邊的事情之前）
1. 通知由符號表示：
	1. '0' for OnNext
	1. 'X' for an OnError
	1. '|' for OnCompleted
4. 許多同步序列可以通過建立序列行來可視化

這是完成的三個值的序列的樣本：
```dos
--0--0--0-|
```
這是一個四個值序列的樣本，然後是一個錯誤：
```dos
--0--0--0--0--X
```
現在回到我們的SelectMany範例，我們可以通過使用值而不是0標記來可視化我們的輸入序列。這是間隔三秒的序列[1,2,3]的Marble 圖表示（注意每個字元表示一秒鐘）。
```dos
--1--2--3|
```
現在我們可以通過引入時間和空間的概念來利用Marble 圖的效用。這裡我們看到由第一個值1產生的序列的可視化，它給出了序列[10,11,12] (請由上往下看)。這些值間隔四秒，但是初始值立即產生。
```dos
1---1---1|
0   1   2|
```
由於值是兩位數字，它們占用兩行，因此值10不會與值1緊接著的值0混淆。我們為選擇器函數產生的每個序列添加一行。
```dos
--1--2--3|
  1---1---1|
  0   1   2|
     2---2---2|
     0   1   2|
        3---3---3|
        0   1   2|
```
現在我們可以可視化來源序列及其子序列，我們應該能夠推導出SelectMany運算子的預期輸出。為了為Marble 圖創建一個結果的行，我們簡單的允許每個子序列的值“落入”新的結果行。
```dos
--1--2--3|
  1---1---1|
  0   1   2|
     2---2---2|
     0   1   2|
        3---3---3|
        0   1   2|
--1--21-321-32--3|
  0  01 012 12  2|
```
如果我們進行這個練習，現在將其應用於程式中，我們可以驗證我們的Marble 圖。首先我們的函式將產生我們的子序列：
```csharp
private IObservable<long> GetSubValues(long offset)
{
//Produce values [x*10, (x*10)+1, (x*10)+2] 4 seconds apart, but starting immediately.
	return Observable.Timer(
		TimeSpan.Zero, 
		TimeSpan.FromSeconds(4))
		.Select(x => (offset*10) + x)
		.Take(3);
}
```
這是接收來源序列以產生最終輸出的程式：
```csharp
// Values [1,2,3] 3 seconds apart.
Observable.Interval(TimeSpan.FromSeconds(3))
	.Select(i => i + 1) //Values start at 0, so add 1.
	.Take(3)            //We only want 3 values
	.SelectMany(GetSubValues) //project into child sequences
	.Dump("SelectMany");
```
產生的輸出符合我們對Marble 圖的期望。
```dos
SelectMany-->10
SelectMany-->20
SelectMany-->11
SelectMany-->30
SelectMany-->21
SelectMany-->12
SelectMany-->31
SelectMany-->22
SelectMany-->32
SelectMany completed
```
我們之前已經看過在查詢語法中使用Select運算子，因而值得注意的是如何使用SelectMany運算子。Select擴充函式很明顯地映射到query comprehension syntax，而SelectMany不是那麼明顯。正如我們在前面的例子中所看到的，只使用Select的簡單實做如下：
```csharp
var query = from i in Observable.Range(1, 5)
select i;
```
如果我們想添加一個簡單的where子句，我們可以這樣做：
```csharp
var query = from i in Observable.Range(1, 5)
	where i%2==0
	select i;
```
要在查詢中增加SelectMany，我們實際上加了一個額外的from子句。
```csharp
var query = from i in Observable.Range(1, 5)
	where i%2==0
	from j in GetSubValues(i)
	select j;
//Equivalent to 
var query = Observable.Range(1, 5)
	.Where(i=>i%2==0)
	.SelectMany(GetSubValues);
```
使用query comprehension syntax的優點是，您可以輕鬆地存取查詢範圍內的其他變數。在這個例子中，我們將來源的值及子值轉換至一匿名型別中。
```csharp
var query = from i in Observable.Range(1, 5)
	where i%2==0
	from j in GetSubValues(i)
	select new {i, j};
query.Dump("SelectMany");
```
輸出：
```dos
SelectMany-->{ i = 2, j = 20 }
SelectMany-->{ i = 4, j = 40 }
SelectMany-->{ i = 2, j = 21 }
SelectMany-->{ i = 4, j = 41 }
SelectMany-->{ i = 2, j = 22 }
SelectMany-->{ i = 4, j = 42 }
SelectMany completed
```
我們準備結束第2章了，這裡的關鍵是讓讀者能夠理解Rx的一個關鍵原則：函數式合成。當我們學會本章後，範例會變得越來越複雜。我們利用LINQ的力量將擴充函式鏈接在一起組成複雜的查詢。

我們沒有一次嘗試學習所有的運算子，我們將它們分組以學習。

* Creation
* Reduction
* Inspection
* Aggregation
* Transformation

對運算子的更深入分析，我們發現大多數運算子實際上是higher order function 概念的特殊化。我們將它們命名為所謂ABC的函數式編程：

* Anamorphism, aka:
	* Ana
	* Unfold
	* Generate
* Bind, aka:
	* Map
	* SelectMany
	* Projection
	* Transform
* Catamorphism, aka:
	* Cata
	* Fold
	* Reduce
	* Accumulate
	* Inject

現在你應該覺得你對如何操作序列有很深的理解。然而，到目前為止我們所學到的東西也都大部分可以應用在IEnumerable序列上。Rx可能比許多人在IEnumerable世界中處理的複雜得多，正如我們在SelectMany函式中看到的。在本書的下一部分中，我們將瞭解Rx天然非同步的特性。憑著我們已經建立的基礎，我們應該能夠解決Rx中更具挑戰性和有趣的功能。