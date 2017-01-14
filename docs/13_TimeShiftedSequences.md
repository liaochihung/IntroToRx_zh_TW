---
tags: Rx, C#
title: Rx介紹 Part 3 - Time-shifted sequences(譯ing...)
---
>嗯，正在學Rx，學了一輪後實際應用時發現還有不懂的地方，再重讀一次，順便簡單的翻譯下…*翻譯不出來*的或是*覺得不重要*的就以"…"符號替換，或顯示原文。
當然，辭不達義的地方也會有，請包含…

##來自[Time-shifted sequences](http://www.introtorx.com/Content/v1.0.10621.0/13_TimeShiftedSequences.html#TimeShiftedSequences)

使用可觀察序列時，時間軸是未知量，我們不知道下一個通知何時到達？而使用IEnumerable序列時，非同步不是一個問題；當我們呼叫MoveNext()函式時，我們會一直被阻擋直到序列產生。本章討論了可以應用於可觀察序列的各種方法，當它和時間的關係是一個問題時。

###Buffer
我們的第一個主題是Buffer函式，一些條件下，你可能不想處理大量的個別通知；相反，你可能喜歡批次處理數據。可能的情況是，一次處理一筆資料太昂貴了，權衡之下，可能要接受一點延遲的代價來批次處理訊息。

Buffer運算子讓你儲存一定範圍的值，然後在緩衝區滿了後將其重新發佈成一個串列。你可以暫時保留定量的元素，避開給定時間範圍內的所有值，或者使用計數和時間的組合。Buffer也提供更進階的覆載，我們在後續章節會討論。
```csharp
public static IObservable<IList<TSource>> Buffer<TSource>(
	this IObservable<TSource> source, 
	int count)
{...}
public static IObservable<IList<TSource>> Buffer<TSource>(
	this IObservable<TSource> source, 
	TimeSpan timeSpan)
{...}
public static IObservable<IList<TSource>> Buffer<TSource>(
	this IObservable<TSource> source, 
	TimeSpan timeSpan, 
int count)
{...}
```
這兩個Buffer的覆載都很直覺，應可讓其它使用者很容易瞭解程式的意圖。
```csharp
IObservable<IList<T>> bufferedSequence;
bufferedSequence = mySequence.Buffer(4);
//or
bufferedSequence = mySequence.Buffer(TimeSpan.FromSeconds(1))
```
在一些狀況下，只能指定緩衝區大小和最大延遲時間可能不夠。一些系統可能有他們可以處理的批次數量的最佳大小，但仍需要時間來限制資料不是過時的，這種狀況對時間及數量的緩衝會是合適的。

下列範例中，我們建立一個一秒產生前十個值，之後每秒產生100個值的序列，並以最大三秒的時間區間及一批15個數值做為緩衝。
```csharp
var idealBatchSize = 15;
var maxTimeDelay = TimeSpan.FromSeconds(3);
var source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(10)
	.Concat(Observable.Interval(TimeSpan.FromSeconds(0.01)).Take(100));
source.Buffer(maxTimeDelay, idealBatchSize)
	.Subscribe(
		buffer => Console.WriteLine("Buffer of {1} @ {0}", DateTime.Now, buffer.Count), 
		() => Console.WriteLine("Completed"));
```
輸出：
```dos
Buffer of 3 @ 01/01/2012 12:00:03
Buffer of 3 @ 01/01/2012 12:00:06
Buffer of 3 @ 01/01/2012 12:00:09
Buffer of 15 @ 01/01/2012 12:00:10
Buffer of 15 @ 01/01/2012 12:00:10
Buffer of 15 @ 01/01/2012 12:00:10
Buffer of 15 @ 01/01/2012 12:00:11
Buffer of 15 @ 01/01/2012 12:00:11
Buffer of 15 @ 01/01/2012 12:00:11
Buffer of 11 @ 01/01/2012 12:00:11
```
注意時間和緩衝區大小的變化。我們絕不會取得一個大於15個元素的緩衝區大小，不會等待大於3秒。一個實際上的例子是，在一個WPF程式中你需要從一個外部的來源將資料載入至一個`ObservableCollection<T>`序列中，可能的情況是，一次添加一個項目對分配程序是一個不必要的負覆（特別是如果你要上百個項目）。你也許做了量測，舉例來說每處理15個項目花費100ms，你決定這是阻塞分配程序的最大時間，以保持程式的響應，這給了我們兩個合理的值來使用：

**source.Buffer(TimeSpan.FromMilliseconds(100), 50)**，這表示我們會最大阻塞UI在100ms以處理一批次50個數量的值，且我們在資料被處理前絕不會等待超過100ms。

###Overlapping buffers
Buffer also offers overloads to manipulate the overlapping of the buffers. The variants we have looked at so far do not overlap and have no gaps between buffers, i.e. all values from the source are propagated through.
Buffer函式也提供處理緩衝重疊的處理的覆載，我們到目前為止看到的都沒有重疊的現象，緩衝區之間也沒有間隙，例如：所有來源值都被傳輸通過。
```csharp
public static IObservable<IList<TSource>> Buffer<TSource>(
	this IObservable<TSource> source, 
	int count, 
	int skip)
{...}
public static IObservable<IList<TSource>> Buffer<TSource>(
	this IObservable<TSource> source, 
	TimeSpan timeSpan, 
	TimeSpan timeShift)
{...}
```
你可以對重疊的緩衝區做下列三種有趣的事：

- Overlapping behavior
確認目前的緩衝區包含一些或全部來自上一個緩衝區的數值
- Standard behavior
確認每個新緩衝區只有新的資料
- Skip behavior
確認每一個新的緩衝區不僅只有新資料，也略過一或多個前一緩衝區的資料

###Overlapping buffers by count
如果要將緩衝區大小當成計數量，需要使用這個覆載：
```csharp
public static IObservable<IList<TSource>> Buffer<TSource>(
	this IObservable<TSource> source, 
	int count, 
	int skip)
{...}
```
你可以應用上述方案在下列情況：

- Overlapping behavior
	* skip < count
- Standard behavior
	* skip = count
- Skip behavior
	* skip > count

skip參數不能小於或等於零，如果你想使用零值（例：每個緩衝區包含所有值），那可以考慮使用Scan函式，並代入`IList<T>`當做accumulator。

Let's see each of these in action. In this example, we have a source that produces values every second. We apply each of the variations of the buffer overload.

```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(10);
source.Buffer(3, 1)
	.Subscribe(
	buffer =>
	{
		Console.WriteLine("--Buffered values");
		foreach (var value in buffer)
		{
			Console.WriteLine(value);
		}
	}, () => Console.WriteLine("Completed"));
```
輸出：
```dos
--Buffered values
0
1
2
--Buffered values
1
2
3
--Buffered values
2
3
4
--Buffered values
3
4
5
etc....
```
注意在每個緩衝區中，一個值會從前一批中被略過，如果我們將skip的參數從1改為3（和緩衝區大小相同），可以看到標準的緩衝區行為。

```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(10);
source.Buffer(3, 3)
...
```
輸出：
```dos
--Buffered values
0
1
2
--Buffered values
3
4
5
--Buffered values
6
7
8
--Buffered values
9
Completed
```
Finally, if we change the skip parameter to 5 (a value greater than the count of 3), we can see that two values are lost between each buffer.
最後，如果我們把skip的參數變成5（比3大的計數），可以看到在每個buffer中間有兩個數值被略過。

```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(10);
source.Buffer(3, 5)
...
```
輸出：
```dos
--Buffered values
0
1
2
--Buffered values
5
6
7
Completed
```
###Overlapping buffers by time
當然，你也可以時間而不是計數來定義緩衝區的這三種行為。
```csharp
public static IObservable<IList<TSource>> Buffer<TSource>(
	this IObservable<TSource> source, 
	TimeSpan timeSpan, 
	TimeSpan timeShift)
{...}
```
為了正確的替換我們的計數重疊的緩衝區，我們需提供下列參數：
```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(10);
var overlapped = source.Buffer(TimeSpan.FromSeconds(3), TimeSpan.FromSeconds(1));
var standard = source.Buffer(TimeSpan.FromSeconds(3), TimeSpan.FromSeconds(3));
var skipped = source.Buffer(TimeSpan.FromSeconds(3), TimeSpan.FromSeconds(5));
```
由於我們的來源每秒固定的產生值，我們可以使用計算範例的相同值，只是以時間為變化。

###Delay
Delay擴充函式是一個對整個序列做純綷的時間位移的函式，你可以使用TimeSpan做為序列會被延遲的時間，或使用DateTimeOffset做為時間的絕對值以控制序列等待，值之間的時間間隔會被保留。

```csharp
// Time-shifts the observable sequence by a relative time.
public static IObservable<TSource> Delay<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime)
{...}
// Time-shifts the observable sequence by a relative time.
public static IObservable<TSource> Delay<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime, 
	IScheduler scheduler)
{...}
// Time-shifts the observable sequence by an absolute time.
public static IObservable<TSource> Delay<TSource>(
	this IObservable<TSource> source, 
	DateTimeOffset dueTime)
{...}
// Time-shifts the observable sequence by an absolute time.
public static IObservable<TSource> Delay<TSource>(
	this IObservable<TSource> source, 
	DateTimeOffset dueTime, 
	IScheduler scheduler)
{...}
```
為顯示動作時的Delay函式，我們建立了每秒產生一次值及timestamp的序列。這顯示不是訂閱被延遲，而是實際將通知轉發至我們的最後訂閱者。

```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1))
	.Take(5)
	.Timestamp();
var delay = source.Delay(TimeSpan.FromSeconds(2));
source.Subscribe(
	value => Console.WriteLine("source : {0}", value),
	() => Console.WriteLine("source Completed"));
delay.Subscribe(
	value => Console.WriteLine("delay : {0}", value),
	() => Console.WriteLine("delay Completed"));
```
輸出：
```dos
source : 0@01/01/2012 12:00:00 pm +00:00
source : 1@01/01/2012 12:00:01 pm +00:00
source : 2@01/01/2012 12:00:02 pm +00:00
delay : 0@01/01/2012 12:00:00 pm +00:00
source : 3@01/01/2012 12:00:03 pm +00:00
delay : 1@01/01/2012 12:00:01 pm +00:00
source : 4@01/01/2012 12:00:04 pm +00:00
source Completed
delay : 2@01/01/2012 12:00:02 pm +00:00
delay : 3@01/01/2012 12:00:03 pm +00:00
delay : 4@01/01/2012 12:00:04 pm +00:00
delay Completed
```
值得注意的是Delay函式不會延遲OnError推送，它會直接被推送。

###Sample
Sample函式單純的在每一個指定的TimeSpan時取得最後一個值。這很有用處，例如在來源序列推送的值太多時，我們僅以時間間隔來取得相關值。下列範得顯示此動作：

```csharp
var interval = Observable.Interval(TimeSpan.FromMilliseconds(150));
	interval.Sample(TimeSpan.FromSeconds(1))
	.Subscribe(Console.WriteLine);
```
輸出：
```dos
5
12
18
```
輸出很有趣，且這是為什麼我選擇用150ms的原因，如果我們根據它們產生的時間繪製值的序列，我們可以看到Sample函式會取得每個週期接收的最後一個值。

```dos
Relative time (ms)	Source value	Sampled value
0		
50		
100		
150	0	
200		
250		
300	1	
350		
400		
450	2	
500		
550		
600	3	
650		
700		
750	4	
800		
850		
900	5	
950		
1000		5
1050	6	
1100		
1150		
1200	7	
1250		
1300		
1350	8	
1400		
1450		
1500	9	
1550		
1600		
1650	10	
1700		
1750		
1800	11	
1850		
1900		
1950	12	
2000		12
2050		
2100	13	
2150		
2200		
2250	14	
2300		
2350		
2400	15	
2450		
2500		
2550	16	
2600		
2650		
2700	17	
2750		
2800		
2850	18	
2900		
2950		
3000	19	19
```
##Throttle
The Throttle extension method provides a sort of protection against sequences that produce values at variable rates and sometimes too quickly. Like the Sample method, Throttle will return the last sampled value for a period of time. Unlike Sample though, Throttle's period is a sliding window. Each time Throttle receives a value, the window is reset. Only once the period of time has elapsed will the last value be propagated. This means that the Throttle method is only useful for sequences that produce values at a variable rate. Sequences that produce values at a constant rate (like Interval or Timer) either would have all of their values suppressed if they produced values faster than the throttle period, or all of their values would be propagated if they produced values slower than the throttle period.
Throttle擴充函式提供了針對以可變速率（有時太快）產生值的序列的一種保護。像Sample函式一樣，Throttle將回傳一段時間內的最後一個取樣值。與範例不同，Throttle的週期是一個滑動的window。每次Throttle接收到一個值時，window被重置。只有一段時間過去才會傳送最後一個值。這意味著Throttle函式僅適用於以可變速率生成值的序列。以恆定速率產生值的序列（如間隔或定時器）或者如果它們產生比throttle週期更快的值，否則它們的所有值都將被抑制，或者如果它們產生的值比throttle週期慢，則它們的所有值都將被傳播。
```csharp
// Ignores values from an observable sequence which are followed by another value before
//  dueTime.
public static IObservable<TSource> Throttle<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime)
{...}
public static IObservable<TSource> Throttle<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime, 
	IScheduler scheduler)
{...}
```
Throttle函式的一個很好的應用是使用它於一個像“Google Suggest”的即時搜尋。當使用者仍在輸入時，我們可以阻止搜尋。一旦給定時間段有暫停，我們就可以使用他們輸入的內容執行搜索。Rx團隊在Rx Hands On Lab中有一個很好的例子

###Timeout
We have considered handling timeout exceptions previously in the chapter on Flow control. The Timeout extension method allows us terminate the sequence with an error if we do not receive any notifications for a given period. We can either specify the period as a sliding window with a TimeSpan, or as an absolute time that the sequence must complete by providing a DateTimeOffset.
我們已經考慮過在stream控制一章中處理逾時異常。 逾時擴充函式允許我們在錯誤的情況下終止序列，如果我們在給定的時間段內沒有收到任何通知。我們可以將時間段指定為具有TimeSpan的滑動window，或者指定序列必須通過提供DateTimeOffset完成的絕對時間。

```csharp
// Returns either the observable sequence or a TimeoutException if the maximum duration
//  between values elapses.
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime)
{...}
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime, 
	IScheduler scheduler)
{...}
// Returns either the observable sequence or a TimeoutException if dueTime elapses.
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	DateTimeOffset dueTime)
{...}
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	DateTimeOffset dueTime, 
	IScheduler scheduler)
{...}
```
如果我們提供了一個TimeSpan，若在此時間內沒有值產生，那這個序列會丟出一個TimeoutException例外。
```csharp
var source = Observable.Interval(TimeSpan.FromMilliseconds(100)).Take(10)
	.Concat(Observable.Interval(TimeSpan.FromSeconds(2)));
var timeout = source.Timeout(TimeSpan.FromSeconds(1));
	timeout.Subscribe(
		Console.WriteLine, 
		Console.WriteLine, 
		() => Console.WriteLine("Completed"));
```
輸出：
```dos
0
1
2
3
4
System.TimeoutException: The operation has timed out.
```
就像Throttle函式，這個覆載僅在序列會以未知頻率產生值的狀況有用。

另一個使用Timeout的方法是給定一個絕對時間；代表此序列須在此時間前完成。

```csharp
var dueDate = DateTimeOffset.UtcNow.AddSeconds(4);
var source = Observable.Interval(TimeSpan.FromSeconds(1));
var timeout = source.Timeout(dueDate);
timeout.Subscribe(
	Console.WriteLine, 
	Console.WriteLine, 
	() => Console.WriteLine("Completed"));
```
輸出：
```dos
0
1
2
System.TimeoutException: The operation has timed out.
```
也許Timeout函式的一個更有趣的用法是，當發生逾時時，在替代序列中進行替換。逾時函式另有覆載，如果發生逾時，則提供指定要使用的接續序列。此功能的行為非常類似於Catch運算子。很容易想像，覆載實際上只是簡單的呼叫這些覆載，並指定一個`Observable.Throw <TimeoutException>`作為接續序列。

```csharp
// Returns the source observable sequence or the other observable sequence if the maximum 
//  duration between values elapses.
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime, 
	IObservable<TSource> other)
{...}
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	TimeSpan dueTime, 
	IObservable<TSource> other, 
	IScheduler scheduler)
{...}
// Returns the source observable sequence or the other observable sequence if dueTime 
//  elapses.
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	DateTimeOffset dueTime, 
	IObservable<TSource> other)
{...}  
public static IObservable<TSource> Timeout<TSource>(
	this IObservable<TSource> source, 
	DateTimeOffset dueTime, 
	IObservable<TSource> other, 
	IScheduler scheduler)
{...}
```
Rx提供了在響應式模型中馴服時間的不可測元素的功能。可以對資料進行緩衝，限制，採樣或延遲，以滿足你的需求。整個序列可以隨時間移動具有延遲功能，並且資料的時間性可以使用逾時運算子確認。這些簡單而強大的功能進一步擴展了開發人員的工具箱，以用於查詢運作中的資料。

> Written with [StackEdit](https://stackedit.io/).