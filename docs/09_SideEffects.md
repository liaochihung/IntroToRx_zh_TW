---
tags: Rx, C#
title: Rx介紹 Part 3 - Side efforts(譯ing...)
---

##[Side efforts](http://www.introtorx.com/Content/v1.0.10621.0/09_SideEffects.html#SideEffects)
生產系統的非功能需求通常需要高可用性，質量監測特性和低缺陷辨視率的前置時間。日誌記錄，除錯，儀器和日誌記錄是開發人員為生產就緒系統需要考慮的常見的非功能需求。這些成品可以被認為是主要業務工作流的邊際效應。邊際效應是一個現實生活中的問題，程式範例和使用導覽經常忽略，但Rx提供了工具來幫助。

在本章中，我們將討論在使用可觀察序列時引入邊際效應的後果。如果除了任何回傳值，它還具有一些其他observable effort，則函式被認為具有邊際效應。一般來說，“可觀察效應”是對狀態的修改。 這個可觀察的效果可以

* 修改具有比函式更大範圍的變數(即全域，靜態或參數)
* I/O，例如從文件或網路讀/寫
* 更新顯示

###邊際效應的問題
函數式編程通常盡量避免產生任何邊際效應。具有邊際效應的函式，特別是會修改狀態的函式，要求programmer不僅須了解函式的輸入和輸出。它們需要理解的地方現在需要擴展到被修改的狀態的歷史和context。這可能大大增加函式的複雜性，從而使得更難正確地理解和維護。

邊際效應並不一定是意外，也不總是故意的。減少意外的邊際效應的簡單方法是減少變更的表面積。我們可以採取的簡單動作是減少狀態的可視性或範圍，且讓其immutable。你可以通過將變數範圍限定為類似函式的程式碼區塊來降低變數的可視性。 您可以通過將類別成員設為private或protected來降低其可視性。通過定義不可變的資料不能被修改，所以不會有邊際效應。這些是明智的封裝規則，將顯著提高你的Rx代碼的可維護性。

為了提供具有邊際效用的查詢的範例，我們將試著通過更新變數(在select子句中)來輸出接收到的元素的索引和值。
```csharp
var letters = Observable.Range(0, 3)
	.Select(i => (char)(i + 65));
var index = -1;
var result = letters.Select(c =>
	{
		index++;
		return c;
	});
result.Subscribe(c => 
	Console.WriteLine("Received {0} at index {1}", c, index),
	() => Console.WriteLine("completed"));
```
輸出：
```dos
Received A at index 0
Received B at index 1
Received C at index 2
completed
```
雖然這似乎是無害的，想像一下若另一個人看了這段程式，且理解為這個團隊使用類似的模式來寫程式。他們也採用這種風格。為了示範，我們在前面的例子中再加上一個訂閱。
```csharp
var letters = Observable.Range(0, 3)
.Select(i => (char)(i + 65));
var index = -1;
var result = letters.Select(c =>
	{
		index++;
		return c;
	});
result.Subscribe(
	c => Console.WriteLine("Received {0} at index {1}", c, index),
	() => Console.WriteLine("completed"));
result.Subscribe(
	c => Console.WriteLine("Also received {0} at index {1}", c, index),
	() => Console.WriteLine("2nd completed"));
```
Output
```dos
Received A at index 0
Received B at index 1
Received C at index 2
completed
Also received A at index 3
Also received B at index 4
Also received C at index 5
2nd completed
```
現在第二個訂閱的輸出顯然是無效的。原其預期的索引值是0、1和2，但是卻得到3、4和5。我曾經在程式庫中見過更誇張的邊際效應。更不好的是修改了布林值的狀態，如hasValues、isStreaming等。我們在後續章節將學到在可觀察序列中比共享狀態來控制工作流程的更好的方式。

除了在現在軟體中建立潛在的無法預測的結果外，帶有邊際效應的程式更難以被測試和維護。且對於未來的重構、增強或是其它的維護性都更加不穩固，在非同步或是併行軟體中更是如此。

###Composing data in a pipeline
取得狀態的較佳方式是將其引入管道中，理想上，我們想讓管道上的每一個部份都是獨立且可確認的。也就是說，組成管道中的每個功能都有自己的輸入和輸出做為它的唯一狀態。為了修正範例，我們增強管道中的資料以讓其不再共享狀態，這將是一個很好的示範以select覆載輸出索引的方式。
```csharp
var source = Observable.Range(0, 3);
var result = source.Select(
	(idx, value) => new
	{
		Index = idx,
		Letter = (char) (value + 65)
	});
result.Subscribe(
	x => Console.WriteLine("Received {0} at index {1}", x.Letter, x.Index),
	() => Console.WriteLine("completed"));
result.Subscribe(
	x => Console.WriteLine("Also received {0} at index {1}", x.Letter, x.Index),
	() => Console.WriteLine("2nd completed"));
```
輸出：
```dos
Received A at index 0
Received B at index 1
Received C at index 2
completed
Also received A at index 0
Also received B at index 1
Also received C at index 2
2nd completed
```
跳脫一下思考的範圍，我們也可以使用其它例如Scan函式來建構相似的結果，下面是範例：
```csharp
var result = source.Scan(
	new
	{
		Index = -1,
		Letter = new char()
	},
	(acc, value) => new
	{
		Index = acc.Index + 1,
		Letter = (char)(value + 65)
	});
```
這裡的關鍵是隔離狀態，並減少或移掉任何會修改狀態的邊際效應。

###Do
We should aim to avoid side effects, but in some cases it is unavoidable. The Do extension method allows you to inject side effect behavior. The signature of the Do extension method looks very much like the Select method;
我們應該努力避免邊際效應，但在某些狀況下無法避免的。而Do擴充函式可讓你注入邊際效應的行為。它的函式定義很像Select函式：

他們都有各種覆載，以適應OnNext，OnError和OnCompleted處理程序的組合：
```csharp
They both return and take an observable sequence
// Invokes an action with side effecting behavior for each element in the observable 
//  sequence.
public static IObservable<TSource> Do<TSource>(
this IObservable<TSource> source, 
Action<TSource> onNext)
{...}
// Invokes an action with side effecting behavior for each element in the observable 
//  sequence and invokes an action with side effecting behavior upon graceful termination
//  of the observable sequence.
public static IObservable<TSource> Do<TSource>(
this IObservable<TSource> source, 
Action<TSource> onNext, 
Action onCompleted)
{...}
// Invokes an action with side effecting behavior for each element in the observable
//  sequence and invokes an action with side effecting behavior upon exceptional 
//  termination of the observable sequence.
public static IObservable<TSource> Do<TSource>(
this IObservable<TSource> source, 
Action<TSource> onNext, 
Action<Exception> onError)
{...}
// Invokes an action with side effecting behavior for each element in the observable
//  sequence and invokes an action with side effecting behavior upon graceful or
//  exceptional termination of the observable sequence.
public static IObservable<TSource> Do<TSource>(
this IObservable<TSource> source, 
Action<TSource> onNext, 
Action<Exception> onError, 
Action onCompleted)
{...}
// Invokes the observer's methods for their side effects.
public static IObservable<TSource> Do<TSource>(
this IObservable<TSource> source, 
IObserver<TSource> observer)
{...}
```
Select覆載為它們的OnNext處理程序帶來Func參數，並且還提供回傳與來源不同類型的可觀察序列的能力。相反，Do方法僅對OnNext處理程序採用`Action <T>`，因此只能回傳與來源類型相同的序列。因為可以傳遞給Do覆載的每個參數都是actions，所以它們隱含地引起邊際效應。

對於下一個範例，我們首先定義以下記錄：
```csharp
private static void Log(object onNextValue)
{
	Console.WriteLine("Logging OnNext({0}) @ {1}", onNextValue, DateTime.Now);
}
private static void Log(Exception onErrorValue)
{
	Console.WriteLine("Logging OnError({0}) @ {1}", onErrorValue, DateTime.Now);
}
private static void Log()
{
	Console.WriteLine("Logging OnCompleted()@ {0}", DateTime.Now);
}
```
這段程式可以讓Do使用上面的方法來介紹一些日誌記錄。
```csharp
var source = Observable
	.Interval(TimeSpan.FromSeconds(1))
	.Take(3);
var result = source.Do(
	i => Log(i),
	ex => Log(ex),
	() => Log());
result.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("completed"));
```
輸出：
```dos
Logging OnNext(0) @ 01/01/2012 12:00:00
0
Logging OnNext(1) @ 01/01/2012 12:00:01
1
Logging OnNext(2) @ 01/01/2012 12:00:02
2
Logging OnCompleted() @ 01/01/2012 12:00:02
completed
```
注意範例中因為Do在查詢中比Subscribe早執行，所以它將先接收到值，因此先寫入Console中。我喜歡把Do方法看作是連到一個序列的接線，它讓你接收序列的值，而無法修改它。

在Rx中看到的最常見的可接受的邊際效應是記錄的需求。Do的定義允許你將它注入到查詢鏈中。這允許我們將日誌記錄添加到我們的序列中並保持封裝性。當存儲庫，服務代理或提供者公開可觀察序列時，它們可以在序列被公開前先加入想要的邊際效應（例如日誌記錄）。然後，消費者可以在查詢中增加運算子（例如，Where、SelectMany），這不會影響來源程序的日誌記錄。

考慮下面的方法。它產生數字，但也記錄它產生的（簡化起見輸出至Console中）。對於使用者的程式來說，日誌記錄是看不見的。

```csharp
private static IObservable<long> GetNumbers()
{
	return Observable.Interval(TimeSpan.FromMilliseconds(250))
		.Do(i => Console.WriteLine("pushing {0} from GetNumbers", i));
}
```
使用如下程式呼叫，
```csharp
var source = GetNumbers();
var result = source.Where(i => i%3 == 0)
	.Take(3)
	.Select(i => (char) (i + 65));
result.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("completed"));
```
輸出：
```dos
pushing 0 from GetNumbers
A
pushing 1 from GetNumbers
pushing 2 from GetNumbers
pushing 3 from GetNumbers
D
pushing 4 from GetNumbers
pushing 5 from GetNumbers
pushing 6 from GetNumbers
G
completed
```
這個範例顯示生產者或中介者可以在序列中增加Log，無論最終使用者要做什麼。

另一個Do的覆載允許你傳入一個`IObserver<T>`做為參數。在這個覆載中，每一個OnNext, OnError and OnCompleted函式會被傳至另一個Do的覆載做為要執行的動作。

使用邊際效應會增加查詢的複雜度。如果邊際效應是必要的惡，那明確的展示可幫助你的同事瞭解你的意圖。使用Do函式是最受歡迎的方式。這也許沒什麼，但考慮到當商業領域在非同步和併行下的固有複雜性，開發者不需要隱藏在Subscribe或Select後的額外增加的邊際效應。

###Encapsulating with AsObservable
Poor encapsulation is a way developers can leave the door open for unintended side effects. Here is a handful of scenarios where carelessness leads to leaky abstractions. Our first example may seem harmless at a glance, but has numerous problems.
不完善的封裝性，會讓開發者留下加入非預期邊際效應的空間。下列是一個因粗心導致抽象上設計不佳而引起的狀況，第一眼看到可能覺得沒什麼，但有很多問題。
```csharp
public class UltraLeakyLetterRepo
{
	public ReplaySubject<string> Letters { get; set; }
	public UltraLeakyLetterRepo()
	{
		Letters = new ReplaySubject<string>();
		Letters.OnNext("A");
		Letters.OnNext("B");
		Letters.OnNext("C");
	}
}
```
此範例中我們公開了一個可觀察序列的屬性。第一個問題就是它的值可以被外界設定，使用者可能在他們想要的狀況下變更。對於此類別的其它使用者是個很不好的體驗。我們加上一些簡單的改變讓它看起來較安全。
```csharp
public class LeakyLetterRepo
{
	private readonly ReplaySubject<string> _letters;
	public LeakyLetterRepo()
	{
		_letters = new ReplaySubject<string>();
		_letters.OnNext("A");
		_letters.OnNext("B");
		_letters.OnNext("C");
	}
	public ReplaySubject<string> Letters
	{
		get { return _letters; }
	}
}
```
現在，Letters屬性只有一個getter並且由一個read only支持。這好多了，敏銳的讀者會注意到Letters屬性返回一個`ReplaySubject <string>`。這在封裝上不好，使用者可以呼叫OnNext / OnError / OnCompleted。要關閉這個漏洞，我們可以簡單地讓回傳型別為`IObservable <string>`。
```csharp
public IObservable<string> Letters
{
	get { return _letters; }
}
```
這個類別看起來好多了，然而，改進的只是美觀上。沒有東西可以防止使用者將結果轉型為`ISubject<string>`，然後呼叫想要的任何函式。此範例中，我們看到外部的程式推送它們的數值進入序列中。
```csharp
var repo = new ObscuredLeakinessLetterRepo();
var good = repo.GetLetters();
var evil = repo.GetLetters();
good.Subscribe(
	Console.WriteLine);
//Be naughty
var asSubject = evil as ISubject<string>;
if (asSubject != null)
{
	//So naughty, 1 is not a letter!
	asSubject.OnNext("1");
}
else
{
	Console.WriteLine("could not sabotage");
}
```
輸出：
```dos
A
B
C
1
```
這問題的解法相當簡單，使用AsObservable擴充函式，_letters欄位會被包裝成僅實作`IObservable<T>`的型別。
```csharp
public IObservable<string> GetLetters()
{
	return _letters.AsObservable();
}
```
輸出：
```dos
A
B
C
could not sabotage
```
雖然我在這些例子中使用了像“邪惡”和“破壞”這樣的字眼，但它往往是一種疏忽，而不是固意導致問題的發生。首先敗在設計類別封裝的設計師。設計介面是很困難的，但我們應該盡最大努力幫助我們的程式碼的使用者成功，給予他們易於發現和強固的類別。如果我們減少它們外觀，只公開我們想要讓使用者使用的功能，類別變得更容易被使用。在這個例子中，我們減少了類別的介面屬性。通過移除屬性setter並透過AsObservable函式返回一個更簡單的型別。

###Mutable elements cannot be protected
雖然AsObservable方法可以封裝你的序列，但你仍然應該意識到它沒有給mutable元素提供任何保護。考慮這個類別序列的使用者可以做什麼：
```csharp
public class Account
{
	public int Id { get; set; }
	public string Name { get; set; }
}
```
這是個簡短的範例，以展示如果我們選擇在序列中修改它的元素，可能造成的混亂：
```csharp
var source = new Subject<Account>();
//Evil code. It modifies the Account object.
source.Subscribe(account => account.Name = "Garbage");
//unassuming well behaved code
source.Subscribe(
	account=>Console.WriteLine("{0} {1}", account.Id, account.Name),
	()=>Console.WriteLine("completed"));
source.OnNext(new Account {Id = 1, Name = "Microsoft"});
source.OnNext(new Account {Id = 2, Name = "Google"});
source.OnNext(new Account {Id = 3, Name = "IBM"});
source.OnCompleted();
```
輸出：
```dos
1 Garbage
2 Garbage
3 Garbage
completed
```
第二個消費者期待的是得到是"Microsoft"、"Google"及"IBM"，但只得到"Garbage"。

可觀察序列將被認為是一系列已解決的事件：事件作為發生了什麼的事實的陳述。這意味著兩件事：首先，每個元素表示在發佈時的狀態的快照，其次，信息從可靠的來源發出。我們想要消除篡改的可能性。理想情況下，類別T將是不可變的，解決這兩個問題。 這樣，序列的消費者可以確信他們獲得的資料是來源產生的資料。不能改變元素似乎是對消費者的限制，但是這些需求最好通過提供更好封裝性的轉換運算子來滿足。

儘可能的避免邊際效應。任何併行和共享狀態的組合通常需要復雜的鎖定，對CPU架構的深刻瞭解，以及他們如何用你的語言和鎖定及最佳化功能合作。較簡單及較佳的方法是避免資料共用，偏好使用不可變資料型別，並利用查詢的合成及轉換。在Where和Select中隱藏邊際效應可能造成程式碼的混亂。如果真的會產生邊際效應，那使用Do函式以明確的表式可能會造成邊際效應。