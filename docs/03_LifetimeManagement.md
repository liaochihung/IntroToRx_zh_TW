---
tags: Rx, C#
title: Rx介紹 Part 1 - Lifetime management(譯)
---

##[生命週期管理](http://www.introtorx.com/Content/v1.0.10621.0/03_LifetimeManagement.html#LifetimeManagement)
Rx程式中，身為消費者的你無法知道何時會從一個序列接收到值或是完成訊號是件很自然的事，然而這種不確定性並不能防止你提供某種程度上的確定性。你可以控制何時要開始或停止接收數值。你仍然要管理你的領域資料，瞭解管理Rx資源的基本方式，能讓你的程式有效率，更少bug，更可預期結果。

Rx提供你最合適訂閱查詢的控制粒度，當你使用這些熟悉的介面時，你可以自己決定相關查詢資源的釋放，這也讓你更有效的管理你的資源，並讓生命週期越緊湊越好。

上一章我們介紹了關鍵型別和一些範例，為了讓範例精簡，我們跳過了`IObservable<T>`介面的一些重要部份，Subscribe函式需要一個`IObserver<T>`做為參數，但我們使用它的以`Action<T>`為參數的擴充方法時就不用提供。我們忽略的重要部份是這兩種方式都有一個回傳值，它們都回傳`IDisposable`型別，這一章我們將帶你瞭解這個回傳值如何幫我們管理我們的訂閱的生命週期。

##Subscribing訂閱
在我們繼續前，先簡略的介紹各Subscribe擴充方法的覆載方法。在前一章使用的[Overload to Subscribe](http://msdn.microsoft.com/en-us/library/ff626574(v=VS.92).aspx "Subscribe Extension method overloads on MSDN")讓我們可以直接傳入`Action<T>`當做訂閱時傳入的參數，以在被喚起時執行OnNext，這些覆載都讓你避免掉建立並傳入一個`IObserver<T>`的需要。
```csharp
//Just subscribes to the Observable for its side effects. 
// All OnNext and OnCompleted notifications are ignored.
// OnError notifications are re-thrown as Exceptions.
IDisposable Subscribe<TSource>(this IObservable<TSource> source);

//The onNext Action provided is invoked for each value.
//OnError notifications are re-thrown as Exceptions.
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
Action<TSource> onNext);

//The onNext Action is invoked for each value.
//The onError Action is invoked for errors
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
Action<TSource> onNext, 
Action<Exception> onError);

//The onNext Action is invoked for each value.
//The onCompleted Action is invoked when the source completes.
//OnError notifications are re-thrown as Exceptions.
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
Action<TSource> onNext, 
Action onCompleted);

//The complete implementation
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
Action<TSource> onNext, 
Action<Exception> onError, 
Action onCompleted);
```
這些覆載方法都允許你傳入不同組合的委託方法以在收到通知時執行，關鍵的是若你傳入的委託沒有指定`OnError`的處理時，此通知會被轉丟出來，考慮到這個錯誤可能在任何時間發生，這可能會讓除錯變的很困難，所以正常來說最好使用有處理`OnError`的覆載。

這個範例中我們使用.Net的結構式例外處理：
```csharp
var values = new Subject<int>();
try
{
	values.Subscribe(value => Console.WriteLine("1st subscription received {0}", value));
}
catch (Exception ex)
{
	Console.WriteLine("Won't catch anything here!");
}
values.OnNext(0);
//Exception will be thrown here causing the app to fail.
values.OnError(new Exception("Dummy exception"));
```
正確的處理例外的方式是提供一個給`OnError`呼叫的委託，如下範例所示：
```csharp
var values = new Subject<int>();
values.Subscribe(
	value => Console.WriteLine("1st subscription received {0}", value),
	ex => Console.WriteLine("Caught an exception : {0}", ex));
values.OnNext(0);
values.OnError(new Exception("Dummy exception"));
```
在本書的後續章結中我們會介紹其它有趣的方式讓你處理序列中的錯誤。

##Unsubscribing取消訂閱
我們還沒介紹如何取消訂閱，如果你想在Rx API中找到相關的方式，你會找不到的，因為與其提供一個方法來執行，Rx在你訂閱時回傳一個`IDisposable`型別物件，可以把這個物件當成是訂閱本身，或是一個訂閱的代表。注銷它會同時有效率的取消訂閱。謹記當你Dispose你訂閱的查詢時，並不有有其它的影響，它謹把自己從Observable的內部訂閱記錄移除，因此這可以讓我們對同一個`IObservable<T>`訂閱多次，且不會對其它的訂閱者有任何影響。下列這個範例中我們初始了兩個訂閱者，然後取消一個，而另一個仍能繼續收到序列的通知。
```csharp
var values = new Subject<int>();
var firstSubscription = values.Subscribe(value => 
	Console.WriteLine("1st subscription received {0}", value));
var secondSubscription = values.Subscribe(value => 
	Console.WriteLine("2nd subscription received {0}", value));
values.OnNext(0);
values.OnNext(1);
values.OnNext(2);
values.OnNext(3);
firstSubscription.Dispose();
Console.WriteLine("Disposed of 1st subscription");
values.OnNext(4);
values.OnNext(5);
```
輸出：
```bash
1st subscription received 0
2nd subscription received 0
1st subscription received 1
2nd subscription received 1
1st subscription received 2
2nd subscription received 2
1st subscription received 3
2nd subscription received 3
Disposed of 1st subscription
2nd subscription received 4
2nd subscription received 5
```
Rx團隊可以建立新的取消訂閱的介面，如_ISubscription_或是_IUnsubscribe_，也可以直接在`IObservable<T>`中增加取消訂閱的函式，但利用`IDisposable`型別反而可以免費的得到下列好處：

>* 型別已經存在
* 人們已知道有這個型別
* `IDisposable`有標準的使用方式和模式
* 透過`using`關鍵字可直接使用
* FxCop等靜態分析工具可以協助你分析你的使用
* 讓`IObservable<T>`保持簡單

如同每個`IDisposable`的教學，你可以呼叫`Dispose`任意次數，第一次呼叫會取消訂閱，而之後的任何呼叫將不會做任何事應為已被取消了。

##OnError and OnCompleted
這兩種函式都代表了一個序列的結束，如果你的序列發出了任一個通知，那一定是最後的通知了，不會再有`OnNext`被喚起，下列範例我們試著在完成後呼叫`OnNext`，其理所當然的被乎略了：
```csharp
var subject = new Subject<int>();
subject.Subscribe(
	Console.WriteLine, 
	() => Console.WriteLine("Completed"));
subject.OnCompleted();
subject.OnNext(2);
```
當然，你可以透過實作自己的`IObservable<T>`去允許在`OnCompleted`或`OnError`後還可以推送訊息，然而這就跟現在的Subject型別實作概念不相容且非標準操作，這可能會為你的軟體帶來無法預期的行為。

一件你應該要注意是是就算在序列完成或發生錯誤後，你仍然要dispose你的訂閱。

##IDisposable
`IDisposable`介面也包含在Rx的一部份，我喜歡把實作`IDisposable`的介面的型別想成它們擁有明確的生命週期管理，我可以明確的呼叫`Dispose()`來表達"我已經完成了某事"。

用這樣子的想法以及對C# 	`using`述序的使用，你可以控制建立的範圍，另外提醒，使用`using`本身是`try`/`finally`區塊的簡單方式，保證當程式離開scope後`Dispose`會被呼叫。

考慮到使用`IDisposable`介面去建立一個scope，你可以建立一些小的類別去實現，如下展示一個記錄執行時間的小類別：
```csharp
public class TimeIt : IDisposable
{
	private readonly string _name;
	private readonly Stopwatch _watch;
	public TimeIt(string name)
	{
		_name = name;
		_watch = Stopwatch.StartNew();
	}
	public void Dispose()
	{
		_watch.Stop();
		Console.WriteLine("{0} took {1}", _name, _watch.Elapsed);
	}
}
```
這個手做的小類別讓你可以量測在其中執行的程式碼所花費的時間，可以用下列的方式來實作：
```csharp
using (new TimeIt("Outer scope"))
{
	using (new TimeIt("Inner scope A"))
	{
		DoSomeWork("A");
	}
	using (new TimeIt("Inner scope B"))
	{
		DoSomeWork("B");
	}
	Cleanup();
}
```
輸出：
```bash
Inner scope A took 00:00:01.0000000
Inner scope B took 00:00:01.5000000
Outer scope took 00:00:02.8000000
```
你也可以用這個方法去設定console程式的文字顏色：
```csahrp
//Creates a scope for a console foreground color. When disposed, will return to 
//  the previous Console.ForegroundColor
public class ConsoleColor : IDisposable
{
	private readonly System.ConsoleColor _previousColor;
	public ConsoleColor(System.ConsoleColor color)
	{
		_previousColor = Console.ForegroundColor;
		Console.ForegroundColor = color;
	}
	public void Dispose()
	{
		Console.ForegroundColor = _previousColor;
	}
}
```
我覺得這可以讓你很容易在console程式中切換字體的顏色，如：
```csharp
Console.WriteLine("Normal color");
using (new ConsoleColor(System.ConsoleColor.Red))
{
	Console.WriteLine("Now I am Red");
	using (new ConsoleColor(System.ConsoleColor.Green))
	{
		Console.WriteLine("Now I am Green");
	}
	Console.WriteLine("and back to Red");
}
```
輸出：
```bash
Normal color
Now I am Red
Now I am Green
and back to Red
```
我們可以看到對`IDisposable`的使用不只能明確的釋放未託管的資源，也是個有用的管理生命週期範圍的工具；從stopwatch timer到console文字顏色，到序列事件的訂閱等。
Rx函式庫不僅採用`IDisposable`和加上了很多自訂的實作：
* Disposable
* BooleanDisposable
* CancellationDisposable
* CompositeDisposable
* ContextDisposable
* MultipleAssignmentDisposable
* RefCountDisposable
* ScheduledDisposable
* SerialDisposable
* SingleAssignmentDisposable

完整的說明請參考附件的Disposables說明，現在我們先看一些很簡單並有用的`Disposable`靜態類別：
```csharp
namespace System.Reactive.Disposables
{
	public static class Disposable
	{
		// Gets the disposable that does nothing when disposed.
		public static IDisposable Empty { get {...} }
		
		// Creates the disposable that invokes the specified action when disposed.
		public static IDisposable Create(Action dispose)
		{...}
	}
}
```
可以看到它有兩個成員方法：`Empty`和`Create`，`Empty`讓你建立空的實例，在`Dispose()`被呼叫時不做任何事，這在你需要一個回傳`IDisposable`的物件但並沒有實作時是很有用的。

而`Create`工廠方法讓你可以傳入一個`Action`委託，讓它在被disposed時可以呼叫執行，`Create`函式會確認標準的Dispose動作，就算你執行多次Dispose，實際上你的委託只會被喚起一次：
```csharp
var disposable = Disposable.Create(() => Console.WriteLine("Being disposed."));
Console.WriteLine("Calling dispose...");
disposable.Dispose();
Console.WriteLine("Calling again...");
disposable.Dispose();
```
輸出：
```bash
Calling dispose...
Being disposed.
Calling again...
```
注意"Being disposed."只被印出一次，後續章節我們會談到其它的管理生命週期的方法，如Observable.Using函式。

##資源管理 vs. 記憶體管理
看起來很多.NET開發者對.NET執行時的垃圾回收機制僅有很概略的了解，特別是它和Finalizer和IDisposable的關係，如同[Framework Design Guidelines](http://msdn.microsoft.com/en-us/library/ms229042.aspx)的作者所說，也這許是因對"資源管理"和"記憶體管理"的困惑。

> 很多人第一次聽到Dispose模式時，會抱怨GC沒有做好它的職責。他們認為它也應該回收資源，就像它在管理未托管資源時一樣，然而GC從來就不是要管理資源的，它是被設計為管理記憶體，且做得很好！- <a href="http://blogs.msdn.com/b/kcwalina/">
		Krzysztof Cwalina</a> from <a href="http://www.bluebytesoftware.com/blog/2005/04/08/DGUpdateDisposeFinalizationAndResourceManagement.aspx">
			Joe Duffy's blog</a>

這一方面證明了微軟讓.NET易於使用的同時，也造成了人們對runtime的背後機制的誤解問題。考慮到這個，我認為你應該謹記你的訂閱基本上不會自動被disposed，你應該假設在訂閱時使用的`IDisposable`物件沒有它自己的finalizer而且在離開它的執行範圍時不會被GC回收，如果你訂閱了但卻乎略它，你會遺失僅有的可取消訂閱的handle，這個訂閱會一直存在，但你無法再存取它的資源，這可能會導致memory leak及多餘的執行中的動作。

然而當你使用`Subscribe`擴充函式時有個例外。這些方法會在你的序列完成時，自動的將訂閱者取消掉，雖然這會自動被執行，但實作時你仍然要假設此序列有可能不會被結束，仍需要一個`IDisposable`去明確的中斷對這個無窮序列的訂閱。

你會發現本書中很多的範例並沒有處理`IDisposable`回傳值，這是為了讓範例簡單而清楚，在附錄中你可以找到相關的資訊。

靠著使用`IDisposable`這個介面，Rx提供了讓你自己決定訂閱的生命週期的功能。每個訂閱都是獨立的，所以取消任一個不會影響到其它，即時有的擴充方法會自動移除訂閱者，但最好還是自己來做，因為你可能在過程中使用到其它的需被dispose的資源。後續章節可看到，訂閱本身可能會引起其它的資源的消耗，如event handles、快取及執行緒等，且記得要在訂閱時提供`OnError`處理去避免例外丟出。

With the knowledge of subscription lifetime management, you are able to keep a tight leash on subscriptions and their underlying resources. With judicious application of standard disposal patterns to your Rx code, you can keep your applications predictable, easier to maintain, easier to extend and hopefully bug free.
