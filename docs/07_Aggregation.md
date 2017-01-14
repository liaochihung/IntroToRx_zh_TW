---
tags: Rx, C#
title: Rx介紹 Part 2 - Aggregation(譯ing...)
---

##[Aggregation](http://www.introtorx.com/Content/v1.0.10621.0/07_Aggregation.html#Aggregation)
原始的資料可能看起來沒有價值，很多時候我們接收到的比山還多的資料，就需要我們透過加強、校驗、組合並濃縮等方式讓資料變的可口一點。考慮到諸如量測、金融、訊號處理及營運資訊等大資料量的領域，其每秒產生的資料可能超過10筆以上，人們可能直接瞭解嗎？或許對人們來說，資料經過求平均、最大最小後的聚合值會較易理解。

接續之前對序列的縮減，本章我們要看的是Rx中存在的聚合函式。第一組方法延續之前對序列處理並減少至單一值的續列輸出。

在開始介紹前，讓我們先建立一個擴充函式"Dump"，我們會用來幫忙建立後續範例。
```csharp
public static class SampleExtentions
{
	public static void Dump<T>(this IObservable<T> source, string name)
	{
		source.Subscribe(
			i=>Console.WriteLine("{0}-->{1}", name, i), 
			ex=>Console.WriteLine("{0} failed-->{1}", name, ex.Message),
			()=>Console.WriteLine("{0} completed", name));
	}
}
```
Those who use LINQPad will recognize that this is the source of inspiration. For those who have not used LINQPad, I highly recommend it. It is perfect for whipping up quick samples to validate a snippet of code. LINQPad also fully supports the IObservable<T> type.

###Count
Count對在`IEnumerable<T>`使用LINQ的人應該很熟悉。就像其它好函式的命名，它“做它所描述的”事，Rx版的Count不一樣在它會回傳一個可觀察序列，而不是一scalar值。回傳的序列僅包含一個記數來源序列元素數的值。很明顯的除非來源序列結束，我們將無法提供此值。

```csharp
var numbers = Observable.Range(0,3);
numbers.Dump("numbers");
numbers.Count().Dump("count");
```
Output:
```dos
numbers-->1
numbers-->2
numbers-->3
numbers Completed
count-->3
count Completed
```
如果你的來源序列元素數可能超過32位元整數的範圍，可以使用另一個LongCount函式，跟Count一樣，只是它回傳的是`IObservable<long>`。

###Min, Max, Sum and Average
另一些較常見的aggregations是Min, Max, Sum and Average。如同Count函式，這些都回傳單一值序列，一旦來源序列完成，它們會計算並產出結果序列。
```csharp
var numbers = new Subject<int>();
numbers.Dump("numbers");
numbers.Min().Dump("Min");
numbers.Average().Dump("Average");
numbers.OnNext(1);
numbers.OnNext(2);
numbers.OnNext(3);
numbers.OnCompleted();
```
Output:
```dos
numbers-->1
numbers-->2
numbers-->3
numbers Completed
min-->1
min Completed
avg-->2
avg Completed
```
Min跟Max另有覆載函式可讓你提供自訂的`ICompare<T>`實作以排序數值，Average擴充函式計算序列的平均值(非中位值)，對整數(int or long)序列來說，輸出會是`IObservable<double>`，如果來原序列是nullable整數，輸出會是`IObservable<double?>，所有其它的數值型別(float, double, decimal and their nullable equivalents)的輸出會跟來源型別相同。

###Functional folds
我們終於要談到在Rx中所謂的catamorphism/fold，這些方法需傳入`IObservable<T>`且會產出一個型別T的輸出。

要先提醒的一點是，所有這些fold函式都是阻塞式的。要注意這些阻塞式函式是因為這讓你從原先非同步範式轉到fold函式的同步式，一不小心的話可能會讓UI凍結或造成deadlocks。後續在我們談到concurrency時會深入瞭解。

>It is worth noting that in the soon to be released .NET 4.5 and Rx 2.0 will provide support for avoiding these concurrency problems. The new async/await keywords and related features in Rx 2.0 can help exit the monad in a safer way.

###First
First函式單純的回傳來源序列中的第一個值。
```csharp
var interval = Observable.Interval(TimeSpan.FromSeconds(3));
//Will block for 3s before returning
Console.WriteLine(interval.First());
```
如果來源序列沒有任何值(i.e. 空序列)，First函式會丟出例外，你可以用這三種方式處理：

- 使用try/catch包住First()函式呼叫
- 使用Take(1)替代，然而這是非同步式的，不是阻塞式。
- 使用FirstOrDefault()函式替代。

FirstOrDefault()函式仍然會阻塞到來源序列推送任一值後，如果推送的是OnError它會被再轉丟出，如果是OnNext，此值會回傳，而在OnCompleted時預設值會被回傳。如同我們前面看到的函式，我們也可以使用無參數版的函式，那預設值就會是default(T)(i.e. 參考型別為null，值型別為0)，或者是提供我們自訂的預設值來使用。

應特別提到BehaviorSubject和First()擴充方法具有的唯一關係。這背後的原因是，BehaviorSubject保證會有一個推送，無論是值，錯誤或結束。當與BehaviorSubject一起使用時，這有效地消除了第一個擴充函式的阻塞式特性。這可以用來使behavior subjects行為像屬性。

###Last
Last和LastOrDefault會阻塞直到來源序列完成，然後回傳最後一個值，如同First()函式OnError會被推送，而若序列為空，Last()會丟出InvalidOperationException，但你可以使用LastOrDefault去避免。
###Single
Single函式用在從序列中取單一值，跟First()及Last()不同的是它幫你確認序列只包含單一個值。這個函式會阻塞直到來源序列產生一個值然後結束。如果來源序列產生其它任何推送的組合，此函式也會轉發出。這個函式很適合和AsyncSubject一起使用，因AsyncSubject只產生一個值。

##建立自訂的Aggregations
如果現在的聚合函式不夠用，你可以自訂，Rx提供了兩個方式。

###Aggregate
Aggregate函式讓你在序列上用自訂的accumulator函式。最基本的覆載中，你要提供一個可代入當下累計值及目前推送值等兩個參數的函式，而回傳值會是新的累計值，函式的定義如下：
```csharp
IObservable<TSource> Aggregate<TSource>(
	this IObservable<TSource> source, 
	Func<TSource, TSource, TSource> accumulator)
```
如果你想建立自己的Sum函式，你可以提供一個僅加上當前值的累加器：
```csharp
var sum = source.Aggregate((acc, currentValue) => acc + currentValue);
```
這個Aggregate覆載有幾個問題。第一，它要求累加值要和原序列值的型別相同，我們在前面已經看到的Average就不一定要這樣。第二，這個覆載需要來源產生至少一個值，否則InvalidOperationException例外會被丟出，就算我們想對一個空序列使用自訂的Count或Sum等計算也不應該這樣，所以我們要使用其它的覆載。這個覆載須代入一額外的參數當做seed，此值可做為一個初始的累計值，它也允許累計值的型別和來源型別不一樣。
```csharp
IObservable<TAccumulate> Aggregate<TSource, TAccumulate>(
	this IObservable<TSource> source, 
	TAccumulate seed, 
	Func<TAccumulate, TSource, TAccumulate> accumulator)
```
用這個覆載更新我們的Sum函式很簡單，只要加上為0的seed值。如同我們預期的，當序列為空時0會被回傳，你也可以建立自己的Count函式。
```csharp
var sum = source.Aggregate(0, (acc, currentValue) => acc + currentValue);
var count = source.Aggregate(0, (acc, currentValue) => acc + 1);
//or using '_' to signify that the value is not used.
var count = source.Aggregate(0, (acc, _) => acc + 1);
```
使用Aggregate來建立自訂的Min和Max函式當做練習，你可能會發現'ICompare<T>'很有用，更確卻的說是`Compare<T>.Default'靜態屬性。完成練習後再看下列的實作…
使用Aggregate建立Min和Max：
```csharp
public static IObservable<T> MyMin<T>(this IObservable<T> source)
{
	return source.Aggregate(
		(min, current) => Comparer<T>
			.Default
			.Compare(min, current) > 0 
				? current 
				: min);
}
public static IObservable<T> MyMax<T>(this IObservable<T> source)
{
	var comparer = Comparer<T>.Default;
	Func<T, T, T> max = 
		(x, y) =>
		{
			if(comparer.Compare(x, y) < 0)
			{
				return y;
			}
			return x;
		};
	return source.Aggregate(max);
}
```
##Scan
雖然Aggregate讓我們在一個將會結束的序列中取得一值，但有時這不是我們需要的。考慮到若是我們想取得一個運作中的序列的總合值時，Aggregate就不適用了。當然它也不適合用在無窮序列中。然而，Scan擴充函式完全滿足這個要求。 Scan和Aggregate的函式定義是相同的；不同之處在於Scan將把每次調用的結果推送到累加器函數。 在這個例子中，我們產生一個running total。
```csharp
var numbers = new Subject<int>();
var scan = numbers.Scan(0, (acc, current) => acc + current);
numbers.Dump("numbers");
scan.Dump("scan");
numbers.OnNext(1);
numbers.OnNext(2);
numbers.OnNext(3);
numbers.OnCompleted();
```
Output:
```dos
numbers-->1
sum-->1
numbers-->2
sum-->3
numbers-->3
sum-->6
numbers completed
sum completed
```
可能值得指出的是，您使用Scan()和TakeLast()產生聚合值。
```csharp
source.Aggregate(0, (acc, current) => acc + current);
//is equivalent to 
source.Scan(0, (acc, current) => acc + current).TakeLast();
```
作為另一個練習，使用我們到目前為止所描述的方法來產生一序列以得到執行中的最小值和執行中的最大值。 這裡的關鍵是每次我們接收到一個小於(或大於一個Max運算子)的值時，我們應該推送該值並更新累加器值。 然而，我們不希望推送重複的值。 例如，給定一個序列[2，1，3，5，0]，我們應該看到執行中最小值序列為[2,1,0]，最大值序列為[2,3,5]。 我們不想看到[2，1，2，2，0]或[2，2，3，5，5]。 繼續看範例實作。
執行中最小值示範：
```csharp
var comparer = Comparer<T>.Default;
Func<T,T,T> minOf = (x, y) => comparer.Compare(x, y) < 0 ? x: y;
var min = source.Scan(minOf)
	.DistinctUntilChanged();
```
執行中最大值示範：
```csharp
public static IObservable<T> RunningMax<T>(this IObservable<T> source)
{
	return source.Scan(MaxOf)
		.Distinct();
}
private static T MaxOf<T>(T x, T y)
{
	var comparer = Comparer<T>.Default;
	if (comparer.Compare(x, y) < 0)
	{
		return y;
	}
	return x;
}
```
雖然兩個範例之間的唯一功能差異是檢查較大值而不是較小值，但範例顯示了兩種不同的樣式。 一些人喜歡第一個範例的簡潔，其他人習慣如第二個範例展現的方式。 這裡的關鍵是使用Distinct或DistinctUntilChanged函式來組成Scan函式。 最好使用DistinctUntilChanged，這樣我們內部不會保留所有值的快取。

##Partitioning
Rx還使您能夠使用標準LINQ運算子GroupBy等功能來分割序列。 這對於取得單一序列及推送給眾多訂閱者或可能在各區中取得其aggregates是有用的。

###MinBy and MaxBy
MinBy和MaxBy運算子允許你基於key selector函式來對序列進行分區。Key selector函式在其他LINQ運算子中很常見，如`IEnumerable<T>`、ToDictionary或GroupBy和Distinct函式。 每個函式都將返回來自鍵值的最小值或最大值。
```csharp
// Returns an observable sequence containing a list of zero or more elements that have a 
//  minimum key value.
public static IObservable<IList<TSource>> MinBy<TSource, TKey>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector)
{...}
public static IObservable<IList<TSource>> MinBy<TSource, TKey>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	IComparer<TKey> comparer)
{...}
// Returns an observable sequence containing a list of zero or more elements that have a
//  maximum key value.
public static IObservable<IList<TSource>> MaxBy<TSource, TKey>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector)
{...}
public static IObservable<IList<TSource>> MaxBy<TSource, TKey>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	IComparer<TKey> comparer)
{...}
```
請注意，每個Min和Max運算子都有一個覆載，該覆載需要一個比較器。 這允許自定型別的比較或對標準型別的自定排序。

考慮從0到10的序列。如果我們用一個key selector，根據它們的modulus of 3將值分組，我們將有3組值。 值和它們的key將如下所示：
```csharp
Func<int, int> keySelector = i => i % 3;
```
```dos
0, key: 0
1, key: 1
2, key: 2
3, key: 0
4, key: 1
5, key: 2
6, key: 0
7, key: 1
8, key: 2
9, key: 0
```
我們可以看到，最小key是0，最大key是2。因此，如果我們用MinBy運算子，我們從序列取得的單一值將是串列[0,3,6,9]。 應用MaxBy運算子將產生串列[2,5,8]。MinBy和MaxBy運算子只會生成一個值（如AsyncSubject），該值將是一個具有零個或多個值的`IList <T>`。

如果你不是想要最小/最大key的值，而是想獲得每個key的最小值，那麼您需要看看GroupBy。

###GroupBy
GroupBy運算子允許你像`IEnumerable <T>`的GroupBy運算子那樣分割你的序列。 以類似的方式，IEnumerable <T>運算子回傳`IEnumerable<IGrouping<TKey,T >>`，而`IObservable<T>`GroupBy運算子回傳`IObservable <IGroupedObservable <TKey,T >>`。
```csharp
// Transforms a sequence into a sequence of observable groups, 
//  each of which corresponds to a unique key value, 
//  containing all elements that share that same key value.
public static IObservable<IGroupedObservable<TKey, TSource>> GroupBy<TSource, TKey>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector)
{...}
public static IObservable<IGroupedObservable<TKey, TSource>> GroupBy<TSource, TKey>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	IEqualityComparer<TKey> comparer)
{...}
	public static IObservable<IGroupedObservable<TKey, TElement>> GroupBy<TSource, TKey, TElement>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	Func<TSource, TElement> elementSelector)
{...}
	public static IObservable<IGroupedObservable<TKey, TElement>> GroupBy<TSource, TKey, TElement>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	Func<TSource, TElement> elementSelector, 
	IEqualityComparer<TKey> comparer)
{...}
```
我發現最後兩個重載有點冗餘，因為我們可以輕鬆地組合一個Select運算子至查詢以獲得相同的功能。

以類似的方式，`IGrouping<TKey,T>`類型擴展了`IEnumerable<T>`，`IGroupedObservable<T>`通過添加Key屬性來擴展`IObservable <T>`。 使用GroupBy有效地給我們一個巢狀的可觀察序列。

要使用GroupBy運算子獲取每個鍵的最小/最大值，我們可以先分割序列，然後對已分割的序列執行Min/Max。
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(0.1)).Take(10);
var group = source.GroupBy(i => i % 3);
group.Subscribe(
	grp => 
		grp.Min()
		.Subscribe(minValue => 
			Console.WriteLine("{0} min value = {1}", grp.Key, minValue)),
			() => Console.WriteLine("Completed"));
```
上面的程式可以工作，但是擁有這些巢狀訂閱呼叫並不是一個好習慣。 我們失去了對巢狀訂閱的控制且很難閱讀。 當您發現自己創建巢狀訂閱時，應考慮如何應用更好的模式。 在這種情況下，我們可以使用SelectMany，我們將在下一章中查看。
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(0.1)).Take(10);
var group = source.GroupBy(i => i % 3);
group.SelectMany(
	grp =>
		grp.Max()
		.Select(value => new { grp.Key, value }))
		.Dump("group");
```
###巢狀可觀察序列
序列的序列的概念在某種程度上可能有點過分，特別是如果兩個序列類型都是可觀察序列。 雖然這是一個進階主題，我們將在這裡討論它，因為它是Rx的常見事件。 我覺得如果我可以conceptualize一個狀況或範例會讓您更容易地了解此概念。

序列的序列的範例：
##資料分區
您可以從單個來源將資料分區，以讓它可以輕鬆地過濾和共享到許多來源。 我們已經看到，資料分區也可能對聚合有用。這通常使用GroupBy運算子完成。

###線上遊戲伺服器
考慮下一系列的伺服器。 新值表示伺服器上線，值本身是一系列延遲值，允許消費者看到可用伺服器的數量和質量的即時資料。如果伺服器關閉，則內部序列可以通過序列completed來表示。

###財務資料流
新市場或工具可能在每天都開和關，打開時可將價格資訊串流，關閉時將串流序列結束。

###聊天室
用戶可以加入聊天(外部序列)，留言(內部序列)和離開聊天(完成內部序列)。

###檔案監視器
當文件被添加到目錄時，他們的修改會被監看(外層序列)。內部序列可以代表對文件的改變，並且內部序列的結束可以代表刪除文件。

考慮這些例子，你可以看到巢狀可觀察序列是多麼有用。有一組運算子與巢狀的可觀察序列(例如SelectMany，Merge和Switch)非常適合，我們將在以後的章節中討論。

當使用巢狀可觀察序列時，可以方便地採用新序列表示新建的約定(例如，新分區被建立，新遊戲主機上線，市場開放，用戶加入聊天，在被監看目錄中建立新檔案)。然後，您可以採用一個約定，表達當一個內部序列完成時代表的意義，(例如，遊戲主機離線，市場關閉，用戶離開聊天，正在觀看的文件被刪除)。巢狀可觀察序列的好處在於，通過建立新的內部序列可以有效地重新啟動已完成的內部序列。

在本章中，我們揭示了LINQ的功能以及它如何適用於Rx。我們將方法鏈接在一起以重現其他方法已經提供的效果。雖然這在學術上是有益的，它也允許我們開始思考功能組成的方式。 我們還看到一些方法在某些類型上很好地工作：First() + `BehaviorSubject <T>`，Single() + `AsyncSubject <T>`，Single() + Aggregate等。接下來，我們將發現更多的方法可添加到我們的函數式工具，並且找到Rx如何處理我們的第三個函數式概念，綁定。

將數據合併到群組和聚合中實現了大量數據的合理消耗。快速產生的數據對於批次處理系統或人類而言太多了。Rx提供了即時聚合和分區的能力，實現即時報告，而不需要昂貴的CEP或OLAP產品。