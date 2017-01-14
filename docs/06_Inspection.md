---
tags: Rx, C#
title: Rx介紹 Part 2 - Inspection(譯...)
---

##[Inspection](http://www.introtorx.com/Content/v1.0.10621.0/06_Inspection.html#Inspection)
要知道我們並不能僅靠過濾冗餘的部份來取得所需的資料，有時候我們還需要從序列中取得相關或符合預期條件的部份，它是否符合特定條件？某個特定值是否存在？我想找到某個值！

上一章我們瞭解透過filter去縮減我們的來源可觀察序列，現在我們要看有關inspection的運算子，這些運算子大部份都會將來源序列縮減到單一值的序列，這些方法的回傳值雖然無法滿足我們對catamorphism的定義 - 非純量值(仍是`IObservable<T>`)，但仍符合我們要將序列縮減至單一值的期待。

這系列的方法對inspection一個現有的序列很有幫助，每一種都會回傳包含一單一結果值的可觀察序列，原生的非同步方式再次證明它們的用處，且易於使用，請看下列介紹。

###Any
我們先來看Any擴充函式的無參數覆載，如果來源序列沒有推送任何值就結束，它只回傳有一值為false的可觀察序列；而若來源序列的第一個值已被推送，則Any的可觀察序回傳true並結束。若第一個被推送的是錯誤，那它也會推送相同錯誤。
```csharp
var subject = new Subject<int>();
subject.Subscribe(
	Console.WriteLine, 
	() => Console.WriteLine("Subject completed"));
var any = subject.Any();
any.Subscribe(
	b => Console.WriteLine("The subject has any values? {0}", b));
subject.OnNext(1);
subject.OnCompleted();
```
輸出：
```dos
1
The subject has any values? True
subject completed
```
如果我們把OnNext(1)拿掉，輸出如下：
```dos
subject completed
The subject has any values? False
```
Any只關心來源序列的第一個值，若是推送的是一個錯誤，它直接送出錯誤，否則就推送true。
```csharp
var subject = new Subject<int>();
subject.Subscribe(Console.WriteLine,
	ex => Console.WriteLine("subject OnError : {0}", ex),
	() => Console.WriteLine("Subject completed"));
var any = subject.Any();
any.Subscribe(
	b => Console.WriteLine("The subject has any values? {0}", b),
	ex => Console.WriteLine(".Any() OnError : {0}", ex),
	() => Console.WriteLine(".Any() completed"));
subject.OnError(new Exception());
```
輸出：
```dos
subject OnError : System.Exception: Fail
.Any() OnError : System.Exception: Fail
```
Any另有一個需代入一個判斷式的覆載，就像Where一樣。
```csharp
subject.Any(i => i > 2);
//Functionally equivalent to 
subject.Where(i => i > 2).Any();
```
試著自己實作上述兩種Any擴充方法當做練習，或許答案不是那麼明顯，但我們之前學到的函式已夠你實現類似功能。

用Observable.Create來建立Any擴充函式：
```csharp
public static IObservable<bool> MyAny<T>(
this IObservable<T> source)
{
	return Observable.Create<bool>(
		o =>
		{
			var hasValues = false;
			return source
				.Take(1)
				.Subscribe(
					_ => hasValues = true,
					o.OnError,
					() =>
					{
						o.OnNext(hasValues);
						o.OnCompleted();
					});
		});
}
public static IObservable<bool> MyAny<T>(
this IObservable<T> source, 
Func<T, bool> predicate)
{
	return source
		.Where(predicate)
		.MyAny();
}
```
###All
擴充函式All()跟Any一樣，除了所有的值都需要符合判斷式，若是任一值的判斷結果為false，輸出序列會馬上結束，如果來源序列為空，All函式會傳回true，如同Any函式，錯誤也會一樣被送出。
```csharp
var subject = new Subject<int>();
subject.Subscribe(
	Console.WriteLine, () => Console.WriteLine("Subject completed"));
var all = subject.All(i => i < 5);
all.Subscribe(
	b => Console.WriteLine("All values less than 5? {0}", b));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(6);
subject.OnNext(2);
subject.OnNext(1);
subject.OnCompleted();
```
Output:
```dos
1
2
6
All values less than 5? False
all completed
2
1
subject completed
```
早期有用Rx的人可能會注意到IsEmpty擴充函式不見了，因為現在你可以用All()函式實作相同功能。
```csharp
//IsEmpty() is deprecated now.
//var isEmpty = subject.IsEmpty();
var isEmpty = subject.All(_ => false);
```
###Contains
Contains擴充函式可以很明顯的以Any函式來覆載之。Contains函式的行為就像Any，然而它的目標使用的是IComparable而不是判斷式，且是設計為尋找特定值而不是依據條件判斷結果而定。I believe that these are not overloads of Any for consistency with IEnumerable.
```csharp
var subject = new Subject<int>();
subject.Subscribe(
	Console.WriteLine, 
	() => Console.WriteLine("Subject completed"));
var contains = subject.Contains(2);
contains.Subscribe(
	b => Console.WriteLine("Contains the value 2? {0}", b),
	() => Console.WriteLine("contains completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnCompleted();
```
Output:
```dos
1
2
Contains the value 2? True
contains completed
3
Subject completed
```
Contains函式另有一個可讓你自行指定`IEqualityComparer<T>`型態參數的覆載，在你需要時會很有用。

###DefaultIfEmpty
DefaultIfEmpty函式在來源序列是空的時候會回傳預設值或是Default(T)，Default(T)的值若是結構則為0，類別則為null，如果來源不為空，則所有值會直接傳出。

這個範例中來源序列產生值，所以DefaultIfEmpty的結果就是來源的值。
```csharp
var subject = new Subject<int>();
subject.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Subject completed"));
var defaultIfEmpty = subject.DefaultIfEmpty();
defaultIfEmpty.Subscribe(
	b => Console.WriteLine("defaultIfEmpty value: {0}", b),
	() => Console.WriteLine("defaultIfEmpty completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnCompleted();
```
Output:
```dos
1
defaultIfEmpty value: 1
2
defaultIfEmpty value: 2
3
defaultIfEmpty value: 3
Subject completed
defaultIfEmpty completed
```
如果來源為空，我們可以使用型別的預設值(如在int則為0)或提供我們自己的值，如範例中的42：
```csharp
var subject = new Subject<int>();
subject.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Subject completed"));
var defaultIfEmpty = subject.DefaultIfEmpty();
defaultIfEmpty.Subscribe(
	b => Console.WriteLine("defaultIfEmpty value: {0}", b),
	() => Console.WriteLine("defaultIfEmpty completed"));
var default42IfEmpty = subject.DefaultIfEmpty(42);
default42IfEmpty.Subscribe(
	b => Console.WriteLine("default42IfEmpty value: {0}", b),
	() => Console.WriteLine("default42IfEmpty completed"));
subject.OnCompleted();
```
Output:
```dos
Subject completed
defaultIfEmpty value: 0
defaultIfEmpty completed
default42IfEmpty value: 42
default42IfEmpty completed
```
###ElementAt
ElementAt函式讓我們可以用索引挑選值，如同`IEnumerable<T>`是以0開始。
```csharp
var subject = new Subject<int>();
subject.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Subject completed"));
var elementAt1 = subject.ElementAt(1);
elementAt1.Subscribe(
	b => Console.WriteLine("elementAt1 value: {0}", b),
	() => Console.WriteLine("elementAt1 completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnCompleted();
```
Output
```dos
1
2
elementAt1 value: 2
elementAt1 completed
3
subject completed
```
由於我們無法確認可觀察序列到底會有多長，所以可以假定這個函式可能會造成問題。如果來源序列只有5個值，然後你想跟它要第六個值，ArgumentOutOfRangeException會在來源序列結束後被推送，因此我們有三個選擇：

- 優雅的處理OnError
- 使用Skip(5).Take(1)；這會跳過前5個值，並只取第六個值，如果序列的元素少於六個，我們只會得到一個空序列，而不會發生錯誤。
- 使用ElementAtOrDefault

ElementAtOrDefault擴充函式在這種索引值大於範圍的狀況下，會推送Default(T)值，而目前沒有提供你自設預設值的選項。
###SequenceEqual
Finally SequenceEqual extension method is perhaps a stretch to put in a chapter that starts off talking about catamorphism and fold, but it does serve well for the theme of inspection. 這個函式讓我們可以比較兩個可觀察序列。當任一序列產生值時，此值會和另一序列的快取值相互比較，確保兩個序列是值相同，長度也相同。這也表示若是來源序列一產生不一樣的值時，結果序列會為false，而在兩個來源序皆相同且結束後，結果序列會為true。
```csharp
var subject1 = new Subject<int>();
subject1.Subscribe(
	i=>Console.WriteLine("subject1.OnNext({0})", i),
	() => Console.WriteLine("subject1 completed"));
var subject2 = new Subject<int>();
subject2.Subscribe(
	i=>Console.WriteLine("subject2.OnNext({0})", i),
	() => Console.WriteLine("subject2 completed"));
var areEqual = subject1.SequenceEqual(subject2);
areEqual.Subscribe(
	i => Console.WriteLine("areEqual.OnNext({0})", i),
	() => Console.WriteLine("areEqual completed"));
subject1.OnNext(1);
subject1.OnNext(2);
subject2.OnNext(1);
subject2.OnNext(2);
subject2.OnNext(3);
subject1.OnNext(3);
subject1.OnCompleted();
subject2.OnCompleted();
```
Output:
```dos
subject1.OnNext(1)
subject1.OnNext(2)
subject2.OnNext(1)
subject2.OnNext(2)
subject2.OnNext(3)
subject1.OnNext(3)
subject1 completed
subject2 completed
areEqual.OnNext(True)
areEqual completed
```
本章討論了一組可讓我們檢查可觀察序列的函式，且一般都會回傳一單值的序列。我們將繼續研究可減少我們的序列的函式，直到我們碰到不好理解的functional fold功能。
