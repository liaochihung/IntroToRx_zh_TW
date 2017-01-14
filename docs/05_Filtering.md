---
tags: Rx, C#
title: Rx介紹 Part 2 - Reducing a sequence(譯ing...)
---
>嗯，正在學Rx，學了一輪後實際應用時發現還有不懂的地方，再重讀一次，順便簡單的翻譯下…*翻譯不出來*的或是*覺得不重要*的就以"…"符號替換，或顯示原文。
當然，辭不達義的地方也會有，請包含…
## [Reducing a sequence](http://www.introtorx.com/Content/v1.0.10621.0/05_Filtering.html#Reduction)
我們活在一個充滿資訊的時代，各種資料以難以想象的速度被產生、儲存並分佈。Consuming this data can be overwhelming, like trying to drink directly from the fire hose. 我們要有擷取所要的資料的能力，選擇那些是相關或無關的，或找出資料中的相關性，使用者、客戶或主管需要你做到這些的同時，並保持高效及耗時更短。

我們已經知道如何建立可觀察序列，現在要準備來瞭解數種可縮減此序列的方式，有下列數種：

Filter and partition operators

- 減少來源序列至一最多保有相同元素數的新序列

Aggregation operators

- 減少來源序列至一僅擁有單一元素的新序列

Fold operators

- Reduce the source sequence to a single element as a scalar value
- 減少來源序列至一擁有單一元素且為純量值的新序列

我們討論過從一純量值建立的可觀察序列被定義為anamorphism或者說是unfold，我們可以想成當把一型別T轉成`IObservable<T>`的anamorphism是一個unfold的動作。這也可以引申為"entering the monad"，此時的monad為`IObservable<T>`。What we will now start looking at are methods that eventually get us to the inverse which is defined as catamorphism or a fold. Fold的其它較受歡迎的名稱是'reduce'、'accumulate'或'inject'。
###Where
在一個序列上套用filter可說是最常見的應用，而最常用的filter就是where子句了。在Rx中你使用Where擴充方法來應用where子句。Where方法的定義如下：
```csharp
IObservable<T> Where(this IObservable<T> source, Fun<T, bool> predicate)
```
注意來源參數和回傳值的型態是一樣的，這讓我們可以應用Fluent Interface方式來寫程式，這種方式在Rx和其它LINQ中很常使用。這範例中我們使用Where來過濾從一個Range序列產生的所有偶數值：
```csharp
var oddNumbers = Observable.Range(0, 10)
	.Where(i => i % 2 == 0)
	.Subscribe(
		Console.WriteLine, 
		() => Console.WriteLine("Completed"));
```
Output:
```dos
0
2
4
6
8
Completed
```
Where運算子是眾多LINQ運算子之一，和其它LINQ運算子一樣被廣泛的使用在不同查詢中，但最常見的是用在`IEnumerable<T>`上，大多情況下它的行為和用在`IEnumerable<T>`時是一樣的，除了例外狀況，後續會討論到。介由實作這些泛用運算子的同時，Rx也得到了C#語言層面上對查詢語法的支持，但本書中的範例為了一致性仍會使用擴充方法。

###Distinct and DistinctUntilChanged

我可以確定大多讀者熟悉在`IEnumerable<T>`上使用Where擴充方式操作，有些應也知道Distinct函式，在Rx中，Distinct也可應用在可觀察序列上。對不熟悉Distinct的人來說，Distinct函式旨在從來源序列中取得原先沒有的值。
```csharp
var subject = new Subject<int>();
var distinct = subject.Distinct();
subject.Subscribe(
	i => Console.WriteLine("{0}", i),
	() => Console.WriteLine("subject.OnCompleted()"));
distinct.Subscribe(
	i => Console.WriteLine("distinct.OnNext({0})", i),
	() => Console.WriteLine("distinct.OnCompleted()"));

subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnNext(1);
subject.OnNext(1);
subject.OnNext(4);
subject.OnCompleted();
```
Output:
```dos
1
distinct.OnNext(1)
2
distinct.OnNext(2)
3
distinct.OnNext(3)
1
1
4
distinct.OnNext(4)
subject.OnCompleted()
distinct.OnCompleted()
```
特別要注意1被推送了3次，但只有第一次可見。Distinct函式有其它覆載讓你可以決定何時要distinct，一是另提供函式以回傳不同值並用比較的方式決定，下列範例使用一自訂類別的屬性來決定一個值是否是distinct。
```csharp
public class Account
{
	public int AccountId { get; set; }
	//... etc
}

public void Distinct_with_KeySelector()
{
	var subject = new Subject<Account>();
	var distinct = subject.Distinct(acc => acc.AccountId);
}
```
In addition to the keySelector function that can be provided, there is an overload that takes an IEqualityComparer<T> instance. This is useful if you have a custom implementation that you can reuse to compare instances of your type T. Lastly there is an overload that takes a keySelector and an instance of IEqualityComparer<TKey>. Note that the equality comparer in this case is aimed at the selected key type (TKey), not the type T.

In addition to the keySelector function that can be provided, 另有一個需要`IEqualityComparer<T>`的覆載的實例，如果你有自訂的實作，你可以有一個可重覆使用的針對型別T的比較類別，最後還有一個可代入KeySelector和`IEqualityComparer<TKey>`實體的覆載，注意這例子中的比較器的參數是TKey而不是型別T。

另有一個Rx特有的Distinct函式的變形，DistinctUntilChanged函式。這個函式只會讓和前一個值不同的值同過，延續上一個範例，但注意此次的輸出值：
```csharp
var subject = new Subject<int>();
var distinct = subject.DistinctUntilChanged();
subject.Subscribe(
	i => Console.WriteLine("{0}", i),
	() => Console.WriteLine("subject.OnCompleted()"));
distinct.Subscribe(
	i => Console.WriteLine("distinct.OnNext({0})", i),
	() => Console.WriteLine("distinct.OnCompleted()"));

subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnNext(1);
subject.OnNext(1);
subject.OnNext(4);
subject.OnCompleted();
```
Output:
```dos
1
distinct.OnNext(1)
2
distinct.OnNext(2)
3
distinct.OnNext(3)
1
distinct.OnNext(1)
1
4
distinct.OnNext(4)
subject.OnCompleted()
distinct.OnCompleted()
```
這兩個範例的不同處是1被推送了兩次，而在第二次的1被推送後，馬上又推送了第三次的1，但在這個範例中被略過了，目前我們團隊覺得這個功能對減少序列中的雜訊很有用。

###IgnoreElements
IgnoreElements擴充方法是個有點奇怪的小工具，它只讓你接收OnCompleted或OnError通知，我們也可以用一個代入總是回傳false的predicate的Where函式來達成相同效果。
```csharp
var subject = new Subject<int>();
//Could use subject.Where(_=>false);
var noElements = subject.IgnoreElements();
subject.Subscribe(
	i=>Console.WriteLine("subject.OnNext({0})", i),
	() => Console.WriteLine("subject.OnCompleted()"));
noElements.Subscribe(
	i=>Console.WriteLine("noElements.OnNext({0})", i),
	() => Console.WriteLine("noElements.OnCompleted()"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnCompleted();
```
Output:
```dos
subject.OnNext(1)
subject.OnNext(2)
subject.OnNext(3)
subject.OnCompleted()
noElements.OnCompleted()
```
如同前述我們可以使用Where來產生相同結果：
```csharp
subject.IgnoreElements();
//Equivalent to 
subject.Where(value=>false);
//Or functional style that implies that the value is ignored.
subject.Where(_=>false);
```
在我們結束Where跟Ignoreelements前，讓我們快速的看一下範例的最後一行，直到最近，我個人一直沒意注到'_'是一個有效的變數名稱；然而它很常被用在函數式編程中，代表一個忽略的參數，這在上述範例中很有用，我們忽略每個接收到的值且總是回傳false，這用法旨在以約定的方式增加可讀性。
###Skip and Take
另一種過濾的關鍵函式，由於很相似所以我們可以把它們當成同一類。讓我們先來看Skip跟Take，它們的行為跟在`IEnumerable<T>`實作中一樣，同時也是最簡單及最常被用的Skip/Take函式，也都只接受單一參數，要skip或是take的元素量。

先看到Skip，此範例中我們有一個內含10個元素的序列，我們用Skip(3)過濾它：
```csharp
Observable.Range(0, 10)
	.Skip(3)
	.Subscribe(
		Console.WriteLine, 
		() => Console.WriteLine("Completed"));
```
輸出：
```dos
3
4
5
6
7
8
9
Completed
```
注意前三個值(0, 1 & 2)皆從輸出序列中忽略，而如果我們用Take(3)，就會得到相反的結果；例如，我們只取前三個值，然後Take會將序列結束：
```csharp
Observable.Range(0, 10)
	.Take(3)
	.Subscribe(
		Console.WriteLine, 
		() => Console.WriteLine("Completed"));
```
輸出：
```dos
0
1
2
Completed
```
為免讓讀者忽略確實是**Take**運算子收到要的數量後結束序列，我們在一個無窮序列中使用以證明之。
```csharp
Observable.Interval(TimeSpan.FromMilliseconds(100))
	.Take(3)
	.Subscribe(
		Console.WriteLine, 
		() => Console.WriteLine("Completed"));
```
輸出：
```dos
0
1
2
Completed
```
###SkipWhile and TakeWhile
再來談到的方法是可以在一個序列中，以一個判斷式的結果來跳過或取得數值，對**SkipWhile**來說，它會濾掉所有值，直到判斷的結果為false後，再回傳後面的序列。
```csharp
var subject = new Subject<int>();
subject
	.SkipWhile(i => i < 4)
	.Subscribe(
		Console.WriteLine, 
		() => Console.WriteLine("Completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnNext(4);
subject.OnNext(3);
subject.OnNext(2);
subject.OnNext(1);
subject.OnNext(0);
subject.OnCompleted();
```
輸出：
```dos
4
3
2
1
0
Completed
```
**TakeWhile**則會在回傳所有符合判斷式的值，直到第一個false時，則結束序列。
```csharp
var subject = new Subject<int>();
subject
	.TakeWhile(i => i < 4)
	.Subscribe(
		Console.WriteLine,
		() => Console.WriteLine("Completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnNext(4);
subject.OnNext(3);
subject.OnNext(2);
subject.OnNext(1);
subject.OnNext(0);
subject.OnCompleted();
```
輸出：
```dos
1
2
3
Completed
```
###SkipLast and TakeLast
這些函式的行為都讓我們很容易從命名本身推論，且都讓我們可以在序列元素後端skip或take數值，然而**SkipLast**會快取所有值，等待來源序列結束，再replay除了最後一個元素值外的所有值。Rx團隊更聰明，實際上的實作會將特定數目的值推入佇列，直到超過最大值，再以先進先出的順序取出佇列值。
```csharp
var subject = new Subject<int>();
subject
	.SkipLast(2)
	.Subscribe(
		Console.WriteLine,
		() => Console.WriteLine("Completed"));
Console.WriteLine("Pushing 1");
subject.OnNext(1);
Console.WriteLine("Pushing 2");
subject.OnNext(2);
Console.WriteLine("Pushing 3");
subject.OnNext(3);
Console.WriteLine("Pushing 4");
subject.OnNext(4);
subject.OnCompleted();
```
輸出：
```dos
Pushing 1
Pushing 2
Pushing 3
1
Pushing 4
2
Completed
```
不像SkipLast，TakeLast需要等到序列結束後才推送它的結果，在範例中，每個步驟都會呼叫Console.WriteLine讓你知道它在那一步。
```csharp
var subject = new Subject<int>();
subject
	.SkipLast(2)
	.Subscribe(
		Console.WriteLine,
		() => Console.WriteLine("Completed"));
Console.WriteLine("Pushing 1");
subject.OnNext(1);
Console.WriteLine("Pushing 2");
subject.OnNext(2);
Console.WriteLine("Pushing 3");
subject.OnNext(3);
Console.WriteLine("Pushing 4");
subject.OnNext(4);
Console.WriteLine("Completing");
subject.OnCompleted();
```
輸出：
```dos
Pushing 1
Pushing 2
Pushing 3
Pushing 4
Completing
3
4
Completed
```
###SkipUntil and TakeUntil
最後兩個函式相較之前介紹的有很讓人興奮的改變，這將會是我們第一次遇到需要兩個可觀察序列互動的函式。

SkipUntil會跳過所有值，直到第二個可觀察序列產生任何值。
```csharp
var subject = new Subject<int>();
var otherSubject = new Subject<Unit>();
subject
	.SkipUntil(otherSubject)
	.Subscribe(
		Console.WriteLine,
		() => Console.WriteLine("Completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
otherSubject.OnNext(Unit.Default);
subject.OnNext(4);
subject.OnNext(5);
subject.OnNext(6);
subject.OnNext(7); // 原文中這裡有OnNext(8)，但沒有輸出
subject.OnCompleted();
```
輸出：
```dos
4
5
6
7
Completed
```
顯然地，對**TakeWhile**來說也是一樣。當第二個可觀察序列產生值，TakeWhile將會結束此輸出序列。
```csharp
var subject = new Subject<int>();
var otherSubject = new Subject<Unit>();
subject
	.TakeUntil(otherSubject)
	.Subscribe(
		Console.WriteLine,
		() => Console.WriteLine("Completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
otherSubject.OnNext(Unit.Default);
subject.OnNext(4);
subject.OnNext(5);
subject.OnNext(6);
subject.OnNext(7);
subject.OnNext(8);
subject.OnCompleted();
```
輸出：
```dos
1
2
3
Completed
```
這是我們對Rx中的過濾函式的快速導讀。它們很簡單，但Rx的Power來自對這些運算子的合成操作。

These operators provide a good introduction to the filtering in Rx. The filter operators are your first stop for managing the potential deluge of data we can face in the information age. We now know how to remove unmatched data, duplicate data or excess data. Next, we will move on to the other two sub classifications of the reduction operators, inspection and aggregation.

> Written with [StackEdit](https://stackedit.io/).