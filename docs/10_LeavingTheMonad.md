---
tags: Rx, C#
title: Rx介紹 Part 3 - Leaving the monad(譯ing...)
---
>嗯，正在學Rx，學了一輪後實際應用時發現還有不懂的地方，再重讀一次，順便簡單的翻譯下…*翻譯不出來*的或是*覺得不重要*的就以"…"符號替換，或顯示原文。
當然，辭不達義的地方也會有，請包含…
##[Leaving the monad](http://www.introtorx.com/Content/v1.0.10621.0/10_LeavingTheMonad.html#LeavingTheMonad)
可觀察序列是一個很有用的概念，特別是我們應用LINQ來組合複雜的查詢時。但即使我們認知可觀察序列的優點，有時會需要離開`IObservable<T>`的應用來使用其它的方式，也許是為了讓你能夠與現有的API(如event和`Task<T>`)合作，也許是你發現會更好測試，或更簡單，於是在你學習Rx時你需要在可觀察範式和原先熟悉的範式切換使用。

###What is a monad
我們在本書前面有稍微的提及了monad，它其實是一個外來的名詞。我會避免以過度複雜的方式來解釋monad，謹提供足夠的說明以幫助我們瞭解下一個要說明的函式群組。Monad的完整定義是非常抽象的。許多人試著用太空人到愛麗絲夢遊仙境等方式提供它的定義，而另一些monadic的教學則使用Haskell的範例程式，這更增加了它的混亂。對我們來說，monad其實是一個表示計算的程式結構。將此與其它編程架構來比較：

Data structure
- 純綷的狀態，例如List、Tree或Tuple
Contract
- 契約定義或抽象化函式，例如介面或抽像類別
Object-Orientated structure
- 狀態和行為一起

通常一個monadic架構讓你將運算子串接在一起以形成一個pipeline，就像我們在擴充函式中做的一樣。

>Monads是一種抽象資料型別建構器，它在領域模型中封裝了程式邏輯而不是資料。

這個簡潔的monad的定義來自於維基百科，它讓我們可將序列當為一個monad；這種狀況下的抽象資料型別就是`IObservable<T>`。當我們使用可觀察序列時，我們將函式組合進抽象資料型別(`IObservable<T>`)中以建立查詢，這個查詢本身變成了被封裝的程式邏輯。

使用monad來定義控制流程在處理典型的麻煩的程式領域(如IO、同步和異常)等非常有用，這恰好是Rx的優勢之一！

###Why leave the monad?
有很多的原因導致你想在不同的範式間使用可觀察序列。需要公開特定函式的函式庫可能要求以事件或是Task實體來呈現，在示範和範例中你可能偏好使用同步函式來限制非同步部份的程式碼數量，這會讓Rx的學習曲線不那麼陡峭。

在產品程式碼中，很少會建議你'break the monad'，特別是從可觀察序列移到阻塞式的函式。在切換非同步和同步範式時要很謹慎，因為這是例如死鎖和可伸縮性問題的主要的常見原因。

In this chapter, we will look at the methods in Rx which allow you to leave the IObservable<T> monad.

###ForEach
ForEach函式提供你一個在資料到達時處理它們的方式。ForEach和Subscribe主要的分別是ForEach會阻塞當前執行緒直到序列完成。
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1))
	.Take(5);
source.ForEach(i => Console.WriteLine("received {0} @ 	{1}", i, DateTime.Now));
Console.WriteLine("completed @ {0}", DateTime.Now);
```
輸出：
```dos
received 0 @ 01/01/2012 12:00:01 a.m.
received 1 @ 01/01/2012 12:00:02 a.m.
received 2 @ 01/01/2012 12:00:03 a.m.
received 3 @ 01/01/2012 12:00:04 a.m.
received 4 @ 01/01/2012 12:00:05 a.m.
completed @ 01/01/2012 12:00:05 a.m.
```
注意，如你預期的，完成行是最後一行。更清楚的說，你可以用Subscribe擴充函式得到相似的結果，但Subscribe函式不會阻塞程式，所以如果你用Subscribe替代範例中的ForEach，我們會先看到完成行先顯示。
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1))
	.Take(5);
source.Subscribe(i => Console.WriteLine("received {0} @ {1}", i, DateTime.Now));
Console.WriteLine("completed @ {0}", DateTime.Now);
```
輸出：
```dos
completed @ 01/01/2012 12:00:00 a.m.
received 0 @ 01/01/2012 12:00:01 a.m.
received 1 @ 01/01/2012 12:00:02 a.m.
received 2 @ 01/01/2012 12:00:03 a.m.
received 3 @ 01/01/2012 12:00:04 a.m.
received 4 @ 01/01/2012 12:00:05 a.m.
```
不像Subscribe擴充函式，ForEach只有一個覆載；需要代入一個`Action<T>`參數。相較之下，之前(預覽)版本的Rx時，ForEach有和Subscribe大部份相同的覆載，但目前已被棄置，我也認為是正確的決定，因在非同步呼叫中不需要OnCompleted處理程序。你可以在ForEach完成後在處理就可，如同我們上述範例的方式。此外，OnError處理程序現構可以替換為標準的try/catch結構化異常處理區塊，就像你在其它的同步程式中所做的一樣。這也給出了在`List<T>`型別上的ForEach實例函式的對稱性。
```csharp
var source = Observable.Throw<int>(new Exception("Fail"));
try
{
	source.ForEach(Console.WriteLine);
}
catch (Exception ex)
{
	Console.WriteLine("error @ {0} with {1}", DateTime.Now, ex.Message);
}
finally
{
	Console.WriteLine("completed @ {0}", DateTime.Now);    
}
```
輸出：
```dos
error @ 01/01/2012 12:00:00 a.m. with Fail
completed @ 01/01/2012 12:00:00 a.m.
```
ForEach函式，如同其它阻塞式的函式(First或Last等)，應該謹慎使用。我謹呈現ForEach函式的測試及範例。在後續介紹concurrency時會和阻塞式呼叫的引入一起討論。

###ToEnumerable
另一個切換出`IObservable<T>`的方式是呼叫ToEnumerable擴充函式。一個簡單的範例：
```csharp
var period = TimeSpan.FromMilliseconds(200);
var source = Observable.Timer(TimeSpan.Zero, period) 
	.Take(5); 
var result = source.ToEnumerable();
foreach (var value in result) 
{ 
	Console.WriteLine(value); 
} 
Console.WriteLine("done");
```
輸出：
```dos
0
1
2
3
4
done
```
當你開始列舉序列時(i.e. lazily)，來源可觀察序列會被實際訂閱。和ForEach相反的是，使用ToEnumerable函式代表你只有在試著移到下一個元素或不存在時才會被阻塞。此外，如果來源序列的產生比你使用的速度還快，值會被快取起來。

為了處理錯誤，你可以對如同其它可列舉序列處理的一樣，將foreach包在一個try/catch中：
```csharp
try 
{ 
	foreach (var value in result)
	{ 
		Console.WriteLine(value); 
	} 
} 
catch (Exception e) 
{ 
	Console.WriteLine(e.Message);
}
```
As you are moving from a push to a pull model (non-blocking to blocking), the standard warning applies.
當你從push轉到pull模式(非阻塞式到阻塞式)時，標準警告要被加上。

###To a single collection
為了避免在push和pull之間反覆，可以使用以下四種方法之一以在單次通知中回傳整個串列。它們都有相同的語義，但產生不同格式的資料。它們類似於對應的`IEnumerable <T>`運算子，但回傳值不同，以保留非同步行為。

###ToArray and ToList
ToArray和ToList都採用可觀察序列，並將其分別打包進`List <T>`的陣列或實體中。一旦可觀察序列完成，陣列或串列將作為結果序列的單一值被推送。
```csharp
var period = TimeSpan.FromMilliseconds(200); 
var source = Observable.Timer(TimeSpan.Zero, period).Take(5); 
var result = source.ToArray(); 
result.Subscribe(arr => 
	{ 
		Console.WriteLine("Received array"); 
		foreach (var value in arr) 
		{ 
			Console.WriteLine(value); 
		} 
	}, 
	() => Console.WriteLine("Completed")); 
Console.WriteLine("Subscribed"); 
```
輸出：
```dos
Subscribed
Received array
0
1
2
3
4
Completed
```
因為這些函式仍然回傳可觀察序列，所以我們可以使用我們的OnError處理這些錯誤。注意來源序列被打包成單一的推送；你不是得到整個序列就是得到錯誤。如果來源序列產生值，然後產生錯誤，你僅能得到錯誤。這四個運算子(ToArray, ToList, ToDictionary and ToLookup)都以相同的方式處理。

###ToDictionary and ToLookup
做為陣列和串列的替代，Rx也可以用ToDictionary和ToLookup將可觀察序列打包進一個dictionary或lookup中。兩個函式都具有和ToArray及ToList函式相同的語義，他們也回傳僅有單一值的序列及相同的錯誤處理方式。

ToDictionary擴充函式的覆載：
```csharp
// Creates a dictionary from an observable sequence according to a specified key selector 
// function, a comparer, and an element selector function.
public static IObservable<IDictionary<TKey, TElement>> ToDictionary<TSource, TKey, TElement>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	Func<TSource, TElement> elementSelector, 
	IEqualityComparer<TKey> comparer) 
{...} 
// Creates a dictionary from an observable sequence according to a specified key selector 
// function, and an element selector function. 
public static IObservable<IDictionary<TKey, TElement>> ToDictionary<TSource, TKey, TElement>( 
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	Func<TSource, TElement> elementSelector) 
{...} 
// Creates a dictionary from an observable sequence according to a specified key selector 
// function, and a comparer. 
public static IObservable<IDictionary<TKey, TSource>> ToDictionary<TSource, TKey>( 
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector,
	IEqualityComparer<TKey> comparer) 
{...} 
// Creates a dictionary from an observable sequence according to a specified key selector 
// function. 
public static IObservable<IDictionary<TKey, TSource>> ToDictionary<TSource, TKey>( 
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector) 
{...} 
```
ToLookup擴充函式的覆載：
```csharp
// Creates a lookup from an observable sequence according to a specified key selector 
// function, a comparer, and an element selector function. 
public static IObservable<ILookup<TKey, TElement>> ToLookup<TSource, TKey, TElement>( 
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	Func<TSource, TElement> elementSelector,
	IEqualityComparer<TKey> comparer) 
{...} 
// Creates a lookup from an observable sequence according to a specified key selector 
// function, and a comparer. 
public static IObservable<ILookup<TKey, TSource>> ToLookup<TSource, TKey>(
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	IEqualityComparer<TKey> comparer) 
{...} 
// Creates a lookup from an observable sequence according to a specified key selector 
// function, and an element selector function. 
public static IObservable<ILookup<TKey, TElement>> ToLookup<TSource, TKey, TElement>( 
	this IObservable<TSource> source, 
	Func<TSource, TKey> keySelector, 
	Func<TSource, TElement> elementSelector)
{...} 
// Creates a lookup from an observable sequence according to a specified key selector 
// function. 
public static IObservable<ILookup<TKey, TSource>> ToLookup<TSource, TKey>( 
	this IObservable<TSource> source, 
	Func<TSource,
	TKey> keySelector) 
{...} 
```
ToDictionary和ToLookup都需要一個函式可以套用在每個元素中來獲取它的鍵值。此外，ToDictionary方法的覆載確認所有鍵應該是唯一的。如果找到重複的鍵值，它使用DuplicateKeyException終止序列。另一方面，`ILookup <TKey，TElement>`被設計為具有由鍵值分組的多個值。如果每個鍵值有多個值，則ToLookup可能是更好的選項。

###ToTask
我們已經將`AsyncSubject <T>`與`Task <T>`進行了比較，甚至展示了如何從一個Task轉換成一個可觀察序列。ToTask擴充函式將允許你將可觀察序列轉換為`Task<T>`。像`AsyncSubject <T>`，此函式將忽略多個值，只返回最後一個值。
```csharp
// Returns a task that contains the last value of the observable sequence. 
public static Task<TResult> ToTask<TResult>(
	this IObservable<TResult> observable) 
{...} 
// Returns a task that contains the last value of the observable sequence, with state to 
//  use as the underlying task's AsyncState. 
public static Task<TResult> ToTask<TResult>(
	this IObservable<TResult> observable,
	object state) 
{...} 
// Returns a task that contains the last value of the observable sequence. Requires a 
//  cancellation token that can be used to cancel the task, causing unsubscription from 
//  the observable sequence. 
public static Task<TResult> ToTask<TResult>(
	this IObservable<TResult> observable, 
	CancellationToken cancellationToken) 
{...} 
// Returns a task that contains the last value of the observable sequence, with state to 
//  use as the underlying task's AsyncState. Requires a cancellation token that can be used
//  to cancel the task, causing unsubscription from the observable sequence. 
public static Task<TResult> ToTask<TResult>(
	this IObservable<TResult> observable, 
	CancellationToken cancellationToken, 
	object state) 
{...}
```
這是一個簡單的範例，展示ToTask如何被使用。注意，ToTask屬於System.Reactive.Threading.Tasks的namespace。
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1)) 
	.Take(5);
var result = source.ToTask(); //Will arrive in 5 seconds. 
Console.WriteLine(result.Result);
```
輸出：
```dos
4
```
如果來源序列要顯示錯誤，task會延續它本來的錯誤處理語義。
```csharp
var source = Observable.Throw<long>(new Exception("Fail!")); 
var result = source.ToTask(); 
try 
{ 
	Console.WriteLine(result.Result);
} 
catch (AggregateException e) 
{ 
	Console.WriteLine(e.InnerException.Message); 
}
```
輸出：
```dos
Fail!
```
一旦你有了task，理所當然的你可以使用所有TPL的功能，例如continuations等。
###`ToEvent<T>`
就如同你可以用FromEventPattern來將事件轉為可觀察序列的來源，你也可以用ToEvent擴充函式讓你的可觀察序列看起來就像標準的.Net事件一樣。
```csharp
// Exposes an observable sequence as an object with a .NET event. 
public static IEventSource<unit> ToEvent(this 		IObservable<Unit> source)
{...} 
// Exposes an observable sequence as an object with a .NET event. 
public static IEventSource<TSource> ToEvent<TSource>(
	this IObservable<TSource> source) 
{...} 
// Exposes an observable sequence as an object with a .NET event. 
public static IEventPatternSource<TEventArgs> ToEventPattern<TEventArgs>(
	this IObservable<EventPattern<TEventArgs>> source) 
	where TEventArgs : EventArgs 
{...} 
```
ToEvent函式回傳一個`IEventSource<T>`型別，它有一個事件成員：OnNext。
```csharp
public interface IEventSource<T> 
{ 
	event Action<T> OnNext; 
} 
```
當我們將可觀察序列用ToEvent函式轉換成事件，我們可以提供一個`Action<T>`來訂閱它，這邊我們用lambda表達式：
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1))
	.Take(5); 
var result = source.ToEvent(); 
result.OnNext += val => Console.WriteLine(val);
```
輸出：
```dos
0
1
2
3
4
```
###ToEventPattern
注意這和標準的事件模式並不一樣，正常來說，當你訂閱了事件，你需要處理sender和EventArgs參數。而上面的範例，我們僅僅取值。如果你想讓你的序列轉成標準的事件模式，你要用ToEventPattern。

ToEventPattern將接受一個`IObservable <EventPattern <TEventArgs >>`並將其轉換為`IEventPatternSource <TEventArgs>`。這些型別的公開介面非常簡單。
```csharp
public class EventPattern<TEventArgs> : 	IEquatable<EventPattern<TEventArgs>>
	where TEventArgs : EventArgs 
{ 
	public EventPattern(object sender, TEventArgs e)
	{ 
		this.Sender = sender; 
		this.EventArgs = e; 
	} 
	public object Sender { get; private set; } 
	public TEventArgs EventArgs { get; private set; } 
	//...equality overloads
} 
public interface IEventPatternSource<TEventArgs> where TEventArgs : EventArgs
{ 
	event EventHandler<TEventArgs> OnNext; 
} 
```
這些看起來很容易應用。因此，如果我們建立一個EventArgs型別，然後用Select來做一些簡單的轉換，我們可以讓一個標準的序列適用於此模式。

EventArgs 型別：
```csharp
public class MyEventArgs : EventArgs 
{ 
	private readonly long _value; 
	public MyEventArgs(long value) 
	{ 
		_value = value; 
	} 
	public long Value 
	{ 
		get { return _value; } 
	} 
} 
```
轉換：
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1))
	.Select(i => new EventPattern<MyEventArgs>(this, new MyEventArgs(i)));
```
Now that we have a sequence that is compatible, we can use the ToEventPattern, and in turn, a standard event handler.
現在我們有一個相容的序列，我們可以使用ToEventPattern，且為標準的事件處理函式。
```csharp
var result = source.ToEventPattern(); 
result.OnNext += (sender, eventArgs) => Console.WriteLine(eventArgs.Value);
```
現在我們知道如何轉成.NET的事件了，讓我們暫停一下，且記住為什麼Rx在這裡會是比較好的模式。

- 在C#中，事件有一個很奇怪的介面，一些人覺得+=跟-=是很不直覺的註測回呼函式的方法。
- 事件很難被組合
- 事件並沒有提供在時間上可被查詢的方法
- 事件常常導致記憶體leak
- 事件並沒有一個標準的通知完成的模式
- 事件在concurrency或多執行緒應用上幾乎幫不到什麼忙，例如，在另一個執行緒上喚起一個事件需要你做一堆事。

---
The set of methods we have looked at in this chapter complete the circle started in the Creating a Sequence chapter. We now have the means to enter and leave the observable sequence monad. Take care when opting in and out of the IObservable<T> monad. Doing so excessively can quickly make a mess of your code base, and may indicate a design flaw.

> Written with [StackEdit](https://stackedit.io/).