---
tags: Rx, C#
title: Rx介紹 Part 1 - Key types(譯)
---

##[關鍵型別](http://www.introtorx.com/Content/v1.0.10621.0/02_KeyTypes.html#KeyTypes)
使用Rx時你需要瞭解兩個關鍵型別，而另一些子型別可以幫助你較有效的瞭解Rx。*`IObserver<T>`* and *`IObservable<T>`*是Rx的基礎，*`ISubject<TSource, TResult>`*的實作可以幫忙降低入門Rx的學習曲線。

很多人熟悉LINQ以及LINQ to Objects, LINQ to SQL & LINQ to XML，上述每一種方式都允許你對以查詢靜態的資料，但Rx提供你查詢動態資料的能力，更明確的說Rx是使用觀察者模式建立的，.Net雖已提供其它實作觀察者模式的實作，例如multicast delegates或events(實際上是繼承至multicast delegates)，然而multicast delegate並不完美，至少有下列不那麼令人滿意的特性；
>* 在C#中，事件有一個很奇怪的介面，一些人覺得"+="和"-="運算子在註冊回呼函式時是一個很不自然的方式
* 事件很難合成
* 事件並沒有提供在時間序列上易於查詢的功能
* 事件是常導致memory leaks的原因之一
* 事件並沒有一個標準的通知完成的pattern
* 在同步或多執行緒應用程式中事件基本上幫不上什麼忙，例如為了在另一個執行緒中喚起一個事件，你要做很多事情

Rx是要為了解決這些問題的。我將在這裡介紹你建構Rx的方式。
##`IObservable<T>`
`IObservable<T>`是Rx中兩個核心介面之一，它是一個很簡單的僅包含一個Subscribe方法的介面，微軟很有信心這介面會常被使用，所以它把這介面放在.Net 4.0 的BCL中，你可以把任何實作了`IObservable<T>`的物件當成是一個串列(流)式的序列物件，所以如果一個函式回傳了一個`IObservable<Price>`，可以當作它是一個價錢的串流。
```CSharp
//Defines a provider for push-based notification.
public interface IObservable<out T>
{
	//Notifies the provider that an observer is to receive notifications.
	IDisposable Subscribe(IObserver<T> observer);
}
```
<sub>
.NET 在System.IO.Stream中的型別及其子型別已經有了串流的概念，System.IO.Stream的實作通常是用在存取檔案、網路或記憶體區塊的串流資料(一般是bytes)，System.IO.Stream的實作通常會提供讀、寫的功能，而有些會有Seek的功能(例如在一個串流中前進後退定位)。前面提到把`IObservable<T>`的物件當成是一個串列(流)式的序列物件，但它並沒有提供串流的seek或寫入的函式，這是一個基礎上的差異，目的是為了防止Rx不會被用在System.IO.Stream的paradigm上。但Rx仍有forward streaming (push), disposing (closing) and completing (eof)這樣的概念，Rx同時也擴充了它的功能，如引進了同步架構及transformation, merging, aggregating and expanding等操作子，當然這些功能也不適用在System.IO.Stream型別上。有些人說可以把`IObservable<T>`當成是Observable Collections，我覺得這很難理解，observable是ok的，但我發現它一點也不像是一個collection，你可以對一個collection執行搜尋、插入及從中移除item等操作，但Rx不行。集合一般來說會有它自己的內部陣列當做它的backing store，而來自`IObservable<T>`的資料並不像集合中已經存在的數值(可能還沒產生)。在WPF/Silverlight中有一個`ObservableCollection<T>`集合也是一樣的，實際上`IObservable<T>`很適合和`ObservableCollection<T>`一起使用，所以為了不讓大家混肴，我們說`IObservable<T>`是一種序列，但`IEnumerable<T>`也是，但它是一種靜態的序列資料，而`IObservable<T>`是動態的序列資料。
</sub>

##`IObserver<T>`
[`IObserver<T>`](http://msdn.microsoft.com/en-us/library/dd783449.aspx "IObserver(Of T) interface - MSDN")是Rx的另一個核心介面，當然它也含在.Net 4.0 的BCL中。還沒使用到.Net4.0也不用擔心，Rx團隊在.Net3.5提供了額外的assembly供人使用。`IObservable<T>`是做為函數式上的`IEnumerable<T>`的二元性存在的，如果你想瞭解這句話的意義，你可以看一下在[Channel9](http://channel9.msdn.com/tags/Rx/)，他們有討論到這個型別在數學上的定義。對所有人來說`IEnumerable<T>`型別可以很有效的回傳三種事物(下個值、例外或序列結束)，所以`IObservable<T>`也可以透過`IObserver<T>`的三個函式`OnNext(T)`、`OnError(Exception)`及`OnCompleted()`做到。
```csharp
//Provides a mechanism for receiving push-based notifications.
public interface IObserver<in T>
{
	//Provides the observer with new data.
	void OnNext(T value);
	//Notifies the observer that the provider has experienced an error condition.
	void OnError(Exception error);
	//Notifies the observer that the provider has finished sending push-based notifications.
	void OnCompleted();
}
```
Rx有一個要遵守的原則，一個`IObserver<T>`的實作可能包含零或多個對`OnNext(T)`的呼叫，接續則可能是`OnEerror(Exception)`或`OnCompleted()`，這個協議確保當一個序列中斷，總是因為一個`OnEerror(Exception)`或`OnCompleted()`發生，且並不要求以上三個函式會被呼叫，如空序列或無限序限這兩個狀況，稍後會談到這個。

有趣的是，當你常在工作上用Rx的`IObservable<T>`的時候，一般你不太會用到`IObserver<T>`，因為Rx提供了如`Subscribe`函式中的匿名實作。

###實現`IObserver<T>`及`IObservable<T>`
這兩種介面很容易實作，如果我們想建立一個可以在console中列印數值的observer，可以如下程式碼：
```csharp
public class MyConsoleObserver<T> : IObserver<T>
{
	public void OnNext(T value)
	{
		Console.WriteLine("Received value {0}", value);
	}
	public void OnError(Exception error)
	{
		Console.WriteLine("Sequence faulted with {0}", error);
	}
	public void OnCompleted()
	{
		Console.WriteLine("Sequence terminated");
	}
}
```
實作observable序列稍稍難一些，回傳一串數值的最簡實作可以如下程式碼所示：
```csharp
public class MySequenceOfNumbers : IObservable<int>
{
	public IDisposable Subscribe(IObserver<int> observer)
	{
		observer.OnNext(1);
		observer.OnNext(2);
		observer.OnNext(3);
		observer.OnCompleted();
		return Disposable.Empty;
	}
}
```
把以上兩種實作程式結合
```csharp
var numbers = new MySequenceOfNumbers();
var observer = new MyConsoleObserver<int>();
numbers.Subscribe(observer);
```
輸出：
```bash
Received value 1
Received value 2
Received value 3
Sequence terminated
```
在這邊的問題是它一點也不像是響應式設計，它是阻塞的，所以也許用`IEnumerable<T>`的實作如`List<T>`或陣列在這邊會較好。
實做這些介面不應該成為困擾我們的問題，你會發現在你用Rx的時候，其實並不需要實作這些介面，Rx已提供你所有需要的東西了，看一下下面這個：
##`Subject<T>`
我喜歡把`IObserver<T>`和`IObservable<T>`當成讀者及寫作者，或是消費者和生產者的介面。如果你建立過`IObservable<T>`的實作，你也許會發現你需要實作推送資料到Subscribers，丟出例外錯誤，以及告之序列結束，而這不就是`IObserver<T>`的定義！也許在一個型別中同時定義了以上兩種介面看起來有點怪，但是它讓你很容易去實作。而[subjects](http://msdn.microsoft.com/en-us/library/hh242969(v=VS.103).aspx "Using Rx Subjects - MSDN")幫你做到了這點，[`Subject<T>`](http://msdn.microsoft.com/en-us/library/hh229173(v=VS.103).aspx "Subject(Of T) - MSDN")是最基本的subjects，實際上你可以在一個回傳`IObservable<T>`的函式中使用`Subject<T>`實作，所以你就可以用OnNext、OnError及OnCompleted去控制序列。

下列這一個基礎的範例中，我建立了一個subject，訂閱這個subject，然後產生數值到序列中(籍由呼叫`subject.OnNext(T)`)。
```csharp
static void Main(string[] args)
{
	var subject = new Subject<string>();
	WriteSequenceToConsole(subject);
	subject.OnNext("a");
	subject.OnNext("b");
	subject.OnNext("c");
	Console.ReadKey();
}
//Takes an IObservable<string> as its parameter. 
//Subject<string> implements this interface.
static void WriteSequenceToConsole(IObservable<string> sequence)
{
	//The next two lines are equivalent.
	//sequence.Subscribe(value=>Console.WriteLine(value));
	sequence.Subscribe(Console.WriteLine);
}
```
注意`WriteSequenceToConsole`這個函式，它帶入一個`IObservable<String>`參數，因它只想要執行`Subscribe`函式，等一下，`Subscribe`函式不是需要一個`IObservable<String>`當作參數嗎？很明顯的`Console.WriteLine`並不符合這個介面，然而不是，其實Rx團隊提供了一個擴充方法給`IObservable<T>`，讓它可以接受一個[`Action<T>`](http://msdn.microsoft.com/en-us/library/018hxwa8.aspx "Action(Of T) Delegate - MSDN")。這個action會在每個item被推送時執行，而還有[更多的擴充方法](http://msdn.microsoft.com/en-us/library/system.observableextensions(v=VS.103).aspx "ObservableExtensions class - MSDN")可以接受委托的組合，以被`OnNext`、`OnCompleted`及`OnError`等方法喚起，這實際上表示我不用實作`IObserver<T>`，酷吧。

你可以看到`Subject<T>`對你開始學Rx很有用，然而，它只是一個最基礎的實作，還有其它三個同類的實作，提供了不同的功能，可以徹底的改變你程式執行的方式。

##`ReplaySubject<T>`
[`ReplaySubject<T>`](http://msdn.microsoft.com/en-us/library/hh211810(v=VS.103).aspx "ReplaySubject(Of T) - MSDN")提供了對數值的快取，以讓後面的訂閱者可以得到舊的數值，考慮下列這個範例中我們將訂閱的動作放在了第一次發佈之後的狀況：
```csharp
static void Main(string[] args)
{
	var subject = new Subject<string>();
	subject.OnNext("a");
	WriteSequenceToConsole(subject);
	subject.OnNext("b");
	subject.OnNext("c");
	Console.ReadKey();
}
```
被寫入至Console的結果值會是'b'和'c'，但'a'被略過了，如果我們將subject變成`ReplaySubject<T>`，我們可以得到所有的數值：
```csharp
var subject = new ReplaySubject<string>();
subject.OnNext("a");
WriteSequenceToConsole(subject);
subject.OnNext("b");
subject.OnNext("c");
```
這在排除同步競爭時很有用處，但是要注意，`ReplaySubject<T>`的預設建構子會建立一個實體以快取所有推送的數值。在很多狀況下這會在程式中建立一些額外的記憶體開銷，所以`ReplaySubject<T>`允許你指定簡單的快取逾期設定值已管理記憶體使用，其中一個選擇是可以指定快取的大小，下列這個範例中我們建立了一個快取大小為2的空間，所以我們的訂閱者只會取得最後2個推送的數值：
```csharp
public void ReplaySubjectBufferExample()
{
	var bufferSize = 2;
	var subject = new ReplaySubject<string>(bufferSize);
	subject.OnNext("a");
	subject.OnNext("b");
	subject.OnNext("c");
	subject.Subscribe(Console.WriteLine);
	subject.OnNext("d");
}
```
這個輸出顯示'a'已經從快取中被丟棄了，但'b'及'c'仍然有效，而'd'是在訂閱後才推送的，所以也會被顯示。
輸出:
```bash
b
c
d
```
另一個可選的預防快取無窮的值的方式是提供一個'window'給這個快取，下列範例中，不再使用指定快取大小的方式，而是指定一個時間的'window'告訴那些快取值是有效的：
```csharp
public void ReplaySubjectWindowExample()
{
	var window = TimeSpan.FromMilliseconds(150);
	var subject = new ReplaySubject<string>(window);
	subject.OnNext("w");
	Thread.Sleep(TimeSpan.FromMilliseconds(100));
	subject.OnNext("x");
	Thread.Sleep(TimeSpan.FromMilliseconds(100));
	subject.OnNext("y");
	subject.Subscribe(Console.WriteLine);
	subject.OnNext("z");
}
```
上面這個範例中，'window'值設為150ms，而每個值推送的間隔是100ms，我們的值被訂閱者訂閱時，第一個值已經過了200ms，所以它已過期且從快取中丟棄了。
```bash
Output:
x
y
z
```

##`BehaviorSubject<T>`
[`BehaviorSubject<T>`](http://msdn.microsoft.com/en-us/library/hh211949(v=VS.103).aspx "BehaviorSubject(Of T) - MSDN")很像`ReplaySubject<T>`，除了它只記得最後一個推送的值。`BehaviorSubject<T>`建立時需要你提供一個預設值`T`，這代表所有的訂閱者將馬上收到一個值(除非它已經完成)。

這個範例中，'a'會被輸出至console中：
```csharp
public void BehaviorSubjectExample()
{
	//Need to provide a default value.
	var subject = new BehaviorSubject<string>("a");
	subject.Subscribe(Console.WriteLine);
}
```
這個範例中，'b'被輸出而不是'a'：
```csharp
public void BehaviorSubjectExample2()
{
	var subject = new BehaviorSubject<string>("a");
	subject.OnNext("b");
	subject.Subscribe(Console.WriteLine);
}
```
這個範例中，'b', 'c' 及'd'都被輸出，'a'沒有：
```csharp
public void BehaviorSubjectExample3()
{
	var subject = new BehaviorSubject<string>("a");
	subject.OnNext("b");
	subject.Subscribe(Console.WriteLine);
	subject.OnNext("c");
	subject.OnNext("d");
}
```
而這個範例中，沒有任何值會被輸出至console，因為此序列在訂閱前已結束：
```c#
public void BehaviorSubjectCompletedExample()
{
	var subject = new BehaviorSubject<string>("a");
	subject.OnNext("b");
	subject.OnNext("c");
	subject.OnCompleted();
	subject.Subscribe(Console.WriteLine);
}
```
要注意在一個快取大小為1的`ReplaySubject<T>`(可以稱呼為replay one subject)和`BehaviorSubject<T>`是不一樣的，`BehaviorSubject<T>`需要一個初始值，假設這兩種subject都已經發出completed訊息，可以確定的是`BehaviorSubject<T>`仍會有一個值，但`ReplaySubject<T>`是無法確定的，謹記此點，所以通常我們不會讓`BehaviorSubject<T>`發出完成訊息。另一個不同點是一個replay-one-subject在完成後仍然會快取一個值，向一個已完成的`BehaviorSubject<T>`訂閱時，我們可以確定不會再收到任何值，`ReplaySubject<T>`則有可能。

`BehaviorSubject<T>s`通常會關聯至類別中的[properties](http://msdn.microsoft.com/en-us/library/65zdfbdt(v=vs.71).aspx)，它們會保持一個值且可提供變更的通知，是以可以當為屬性值的備份欄位的候選。

##`AsyncSubject<T>`
[`AsyncSubject<T>`](http://msdn.microsoft.com/en-us/library/hh229363(v=VS.103).aspx "AsyncSubject(Of T) - MSDN")跟Replay及Behavior像的地方在於它也快取數值，但僅儲存最後一個，且僅在它完成時推送出去，一般用法是僅推送一個值然後馬上結束，這表示它是可以跟`Task<T>`相提並論的。

這個範例中，沒有任何值會被推送，因為此序列永不會結束，也就沒有任何輸出：
```csharp
static void Main(string[] args)
{
	var subject = new AsyncSubject<string>();
	subject.OnNext("a");
	WriteSequenceToConsole(subject);
	subject.OnNext("b");
	subject.OnNext("c");
	Console.ReadKey();
}
```
這個範例中，我們執行了`OnCompleted`函式，所以最後一個值'c'被輸出至console中：
```csharp
static void Main(string[] args)
{
	var subject = new AsyncSubject<string>();
	subject.OnNext("a");
	WriteSequenceToConsole(subject);
	subject.OnNext("b");
	subject.OnNext("c");
	subject.OnCompleted();
	Console.ReadKey();
}
```
##隱性契約
當你使用上述Rx函式時，有些隱性契約需要被支持，關鍵的是一旦一個序列已經完成，不要對它有其餘的操作。一個序列有兩種完成的方式，一是呼叫`OnCompleted()`，一是呼叫`OnError(Exception)`。
本章提及的這四種subjects都隱含的遵守這個契約，一旦序列已經完成，它會略過所有再次推送的數值、錯誤或完成訊息。
本範例中我們試著在序列完成後推送'c'，但只有'a'及'b'被輸出至console中：
```csharp
public void SubjectInvalidUsageExample()
{
	var subject = new Subject<string>();
	subject.Subscribe(Console.WriteLine);
	subject.OnNext("a");
	subject.OnNext("b");
	subject.OnCompleted();
	subject.OnNext("c");
}
```

## ISubject介面
上述這四種subjects透過另一個介面的集合實作了`IObservable<T>`及`IObserver<T>`：
```csharp
//Represents an object that is both an observable sequence as well as an observer.
public interface ISubject<in TSource, out TResult> 
: IObserver<TSource>, IObservable<TResult>
{
}
```
以上所有subjects都有TSource及TResult，它們實現下列這個所有介面的superset：
```csharp
//Represents an object that is both an observable sequence as well as an observer.
public interface ISubject<T> : ISubject<T, T>, IObserver<T>, IObservable<T>
{
}
```
這些介面不沒有被廣泛的使用，但證實了當subjects沒有共用一個基礎類別時也很有用，在談論到[Hot and cold observables](14_HotAndColdObservables.html)時我們會再說明subject介面的使用。

##Subject 工廠
最後讓你知道也可透過工廠方法來建立subject，考慮到subject實作了`IObservable<T>`及`IObserver<T>`兩種介面，很合理的也應該有種工廠方法可以讓你使用，`Subject.Create(IObserver<TSource>, IObservable<TResult>)`提供了這個功能：
```csharp
//Creates a subject from the specified observer used to publish messages to the subject
//  and observable used to subscribe to messages sent from the subject
public static ISubject<TSource, TResult> Create<TSource, TResult>(
IObserver<TSource> observer, 
IObservable<TResult> observable)
{...}
```
Subjects提供了一個很方便的方式去瞭解Rx，然而它們並不建議在日常中使用，[Usage Guidelines](http://www.introtorx.com/Content/v1.0.10621.0/18_UsageGuidelines.html)中的附錄說明了原因。與其直接使用，你應該偏用工廠方法，我們在下一部份[Part 2](04_CreatingObservableSequences.html)會說明。

這`IObservable<T>`及`IObserver<T>`這兩個基礎型別及subject型別應是你學習Rx的基本知識，重要的是瞭解這些型別及它們隱含的約定，你會發現在產品中的程式碼中很少使用`IObserver<T>`介面及subject型別，但是瞭解他們如何應用在系統中仍然是需要的。

The IObservable<T> interface is the dominant type that you will be exposed to for representing a sequence of data in motion, and therefore will comprise the core concern for most of your work with Rx and most of this book.
