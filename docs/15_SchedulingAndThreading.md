---
title: Scheduling and threading
---

#PART 4 - Concurrency

Rx主要是以非同步查詢動態資料的一個系統，為了有效的提供開發者所需的非同步級別，一定等級的並行控制是需要的。為了使用序列資料，我們也要能夠同步的產生序列資料。

在本書第四也是最後的部份，我們將討論在查詢動態資料時必須要的各種併發性的考慮。我們會看到如何在可能時避免使用併發，並在需要時正確使用，也會看到Rx提供的優秀抽象機制，讓併發成為宣告式語法且適合單元測試。在我看來，這兩個功能已經夠讓你決定在程式中使用Rx了。我們也將研究在sliding windows of time中查詢併發序列及分析資料。

#Scheduling and threading

到目前為止，我們已經設法顯式地使用多執行緒或併發。我們談論過的一些函式隱涵地使用了一定程度的併發以執行工作（如`Buffer`、`Delay`、`Sample`等，都需要一個獨立的thread/scheduler/timer去執行），然而，這些大都遠離我們了。本章將討論Rx API的優雅與美觀，以及它有效的消除了對`WaitHandle`型別的需求，及任何對`Thread`s、`ThreadPool`或`Task`s等的顯式呼叫的能力。

##Rx is single-threaded by default

更精確的說，Rx是一個自由執行緒的模型。

一個常見的誤解是Rx是預設多執行緒的。這可能是一個無效的假設而不是強烈的信念，同樣的有人認為.Net事件是多執行緒的直到他們挑戰這個概念。我們在[Appendix](19_DispellingMyths.html#DispellingEventMyths)會解開這個謎題，並很確定的說事件是單執行緒且同步的。

就像事件，Rx只是一種將callbacks鏈接在一起並給定通知的方法。雖然Rx是自由執行緒模式，這並不表示訂閱或呼叫`OnNext`會對你的序列引入多執行緒。自由執行緒表示你不會被你選擇的執行緒限制住以執行工作。舉例來說，你可以選擇喚起一個訂閱，觀察或產生通知以執行你的工作。相對於自由執行緒模型的是_Single Threaded Apartment_ (STA)模型，你僅能在給定的執行緒中和系統互動。STA模型常被用在使用者介面或是一些COM interop中。所以，做一個總結：如果你沒有用任何的scheduling，你的callbacks會在`OnNext`/`OnError`/`OnCompleted`被呼叫的同一個執行緒中被執行。

此範例中，我們建了一個subject，然後在不同執行緒中呼叫`OnNext`，並顯示執行緒的編號：
```csharp
	Console.WriteLine("Starting on threadId:{0}", Thread.CurrentThread.ManagedThreadId);
	var subject = new Subject<object>();

	subject.Subscribe(
		o => Console.WriteLine("Received {1} on threadId:{0}", 
			Thread.CurrentThread.ManagedThreadId, 
			o));

	ParameterizedThreadStart notify = obj =>
	{
		Console.WriteLine("OnNext({1}) on threadId:{0}",
							Thread.CurrentThread.ManagedThreadId, 
							obj);
		subject.OnNext(obj);
	};

	notify(1);
	new Thread(notify).Start(2);
	new Thread(notify).Start(3);
```
輸出：
```dos
Starting on threadId:9
OnNext(1) on threadId:9
Received 1 on threadId:9
OnNext(2) on threadId:10
Received 2 on threadId:10
OnNext(3) on threadId:11
Received 3 on threadId:11
```

注意每一個`OnNext`是在它收到通知的同一個執行緒當中被呼叫的，雖然我們不總是預期如此，因此Rx引入了一個非常方便的機制，用於在程式中導入併發和多執行緒：`Scheduling`。

##SubscribeOn and ObserveOn

在Rx的世界中，一般來說有兩種併發模型是你想控制的：

 * 訂閱的使用
 * 觀察通知

正如你可能猜測的，透過`IObservable<T>`公開了`SubscribeOn`及`ObserveOn`兩個擴充函式，兩個函式都有一個可代入`IScheduler`(或`SynchronizationContext`)並回傳一個`IObservable<T>`的覆載，所以讓你可以將函式鏈結在一起。
```csharp
	public static class Observable 
	{
		public static IObservable<TSource> ObserveOn<TSource>(
			this IObservable<TSource> source, 
			IScheduler scheduler)
		{...}
		public static IObservable<TSource> ObserveOn<TSource>(
			this IObservable<TSource> source, 
			SynchronizationContext context)
		{...}
		public static IObservable<TSource> SubscribeOn<TSource>(
			this IObservable<TSource> source, 
			IScheduler scheduler)
		{...}
		public static IObservable<TSource> SubscribeOn<TSource>(
			this IObservable<TSource> source, 
			SynchronizationContext context)
		{...}
	}
```
在這裡我想指出一個陷阱，在前幾次我使用這些覆載函式時，我對他們到底做了什麼很困惑。你應該用`SubscribeOn`函式來描述你計劃執行的任何warm-up及背景執行程式碼。例如，如果你使用`Subscribe`和`Observable.Create`，傳遞給`Create`函式的委託會在指定的scheduler中被執行。

在此範例中，我們有一個透過標準訂閱`Observable.Create`函式產生的序列：
```csharp
	Console.WriteLine("Starting on threadId:{0}", Thread.CurrentThread.ManagedThreadId);
	var source = Observable.Create<int>(
		o =>
		{
			Console.WriteLine("Invoked on threadId:{0}", Thread.CurrentThread.ManagedThreadId);
			o.OnNext(1);
			o.OnNext(2);
			o.OnNext(3);
			o.OnCompleted();
			Console.WriteLine("Finished on threadId:{0}",
			Thread.CurrentThread.ManagedThreadId);
			return Disposable.Empty;
		});

	source
		//.SubscribeOn(Scheduler.ThreadPool)
		.Subscribe(
			o => Console.WriteLine("Received {1} on threadId:{0}",
				Thread.CurrentThread.ManagedThreadId,
				o),
			() => Console.WriteLine("OnCompleted on threadId:{0}",
				Thread.CurrentThread.ManagedThreadId));
	Console.WriteLine("Subscribed on threadId:{0}", Thread.CurrentThread.ManagedThreadId);
```
輸出：
```dos
Starting on threadId:9
Invoked on threadId:9
Received 1 on threadId:9
Received 2 on threadId:9
Received 3 on threadId:9
OnCompleted on threadId:9
Finished on threadId:9
Subscribed on threadId:9
```
你會發現我們執行的所有動作都在同一個執行緒上，而且，要注意到的是每件事都是循序的。當訂閱完成，`Create`委託被呼叫，當`OnNext(1)`被呼叫，因此，`OnNext`處理器也被呼叫，後面的也是，這些都是同步的一直到`Create`委託完成，所以`Subscribe`在最後一行繼續，並顯示我們在同一個執行緒中。

而如果你在鏈結中加上`SubscribeOn`(取消註解)，執行的順序會很不一樣：
```dos
Starting on threadId:9
Subscribed on threadId:9
Invoked on threadId:10
Received 1 on threadId:10
Received 2 on threadId:10
Received 3 on threadId:10
OnCompleted on threadId:10
Finished on threadId:10
```
觀察到訂閱的呼叫是非阻塞式的，而`Create`委託在thread pool中被執行，所有我們的處理程序也是如此。

`ObserveOn`函式用於聲明你希望將通知排程至何處。當使用STA系統，最常見的UI應用程式時，我覺得`ObserveOn`函式最有用處。當撰寫UI應用程式時，`SubscribeOn`/`ObserveOn`有兩個很有用的原因：

 * 你不想讓UI介面的執行緒停止回應
 * 但你需要透過UI執行緒更新UI中的物件

避免阻塞UI執行緒是很重要的，否則使用者的體驗會覺得很差。Silverlight和WPF的一般說明是若會阻塞超過150-250ms的工作不應該在UI執行緒中被執行，這約略是使用者會注意到介面延遲的時間（如滑鼠位移遲頓、動畫顯示delay），而適用於windows 8的Metro風格的軟體中，允許的最大阻塞時間僅為50ms。這更嚴格的規則是為確保在不同的應用程序中有一致的快速與流暢的使用者體驗。而目前桌上型處理器的能力可以輕鬆的實現大量的50ms的處理。然而，隨著處理器更加多樣化（單/多/更多核，加上高功率的桌上型vs.低功率的ARM 平板/手機），你可以在50ms時間內執行的數量範圍很大。一般來說：任何I/O、大量計算或任何無關UI的工作不應該在UI執行緒中被執行。建立響應式UI應用程式的一般模式是：

 * 響應某種使用者操作
 * 在背景執行工作
 * 將結果傳回UI執行緒
 * 更新UI

這很適合Rx：對事件響應，隱涵的組合事件，並傳送資料至鏈結的函式呼叫。隨著scheduling的納入，在響應式軟體中我們有可隨使用者的需求隨時調整是否要在UI執行緒中執行。

考慮下在一個WPF應用程式中，我們用Rx來產生一個`ObservableCollection<T>`集合，幾乎可以確定的是你會用`SubscribeOn`以離開當前的`Dispatcher`，再來用`ObserveOn`確保你能收到來自原先Dispatcher的通知。如果無法使用`ObserveOn`函式，你的`OnNext`處理程式會在產生此推送的同一個執行緒被喚起。在Silverlight/WPF中，這可能會導致一些不支援的/跨越執行緒的例外。在下列範例中，我們對一個`Customers`的序列訂閱，並在新的執行緒中執行訂閱，及確保當我們收到`Customer`的通知時，我們把它加入至在`Dispatcher`中的`Customers`集合。
```csharp
	_customerService.GetCustomers()
		.SubscribeOn(Scheduler.NewThread)
		.ObserveOn(DispatcherScheduler.Instance) 
		//or .ObserveOnDispatcher() 
		.Subscribe(Customers.Add);
```
##Schedulers
`SubscribeOn`和`ObserveOn`需要我們代入一個`IScheduler`型別的參數，這裡我們會深入瞭解一下到底什麼是schedulers，以及有那些可用的實作。

有兩種我們可以使用的關於schedulers的主要型別：

* `IScheduler`介面
  * 所有schedulers的共通介面
* `Scheduler`靜態類別
  * `IScheduler`介面的實作，並提供一些針對`IScheduler`的幫助函式

`IScheduler`介面目前和實作此介面的型別比起來還不是那麼重要。要瞭解Rx中的`IScheduler`的一個關鍵的概念是它是用來排程某些動作馬上或在未來的某一時間被執行。`IScheduler`的實作定義了此動作會如何被喚起，例如透過執行緒池非同步的喚起，或在新的執行緒、或是一個message pump，或在當前執行緒同步執行。根據你的平台（Silverlight 4，Silverlight 5，.NET 3.5，.NET 4.0），你將通過一個靜態類別“Scheduler”得到你需要的大部分實作。

在我們更深入瞭解`IScheduler`前，讓我們先看一下最常用的擴充函式，然後再介紹一些常見的實作。

這是`IScheduler`中最常用的（擴充）函式，只要指定待執行的動作給他。
```csharp
	public static IDisposable Schedule(this IScheduler scheduler, Action action)
	{...}
```
可以像這樣使用：
```csharp
	IScheduler scheduler = ...;
	scheduler.Schedule(()=>{ Console.WriteLine("Work to be scheduled"); });
```
這些是`Scheduler`類別提供的靜態屬性，

* `Scheduler.Immediate` 確定這個動作馬上被執行而不會被排程進去。will ensure the action is not scheduled, but rather executed immediately.
* `Scheduler.CurrentThread`確保動作被當前執行緒執行。這和`Immediate`不一樣，`CurrentThread`會把動作放入佇列中等待執行，稍後會使用範例程式比較這兩個schedulers。
* `Scheduler.NewThread`會在新的執行緒將工作排程。

* `Scheduler.ThreadPool`會在執行緒池中排程所有的工作。
* `Scheduler.TaskPool`會在TaskPool排程所有的工作。Silverlight 4 或 .NET 3.5還沒有此屬性。

如果你使用WPF或Silverlight，你也可以存取`DispatcherScheduler.Instance`。這讓你不管是現在或以後都使用共用介面將工作排程至`Dispatcher`中。`IObservable<T>`另提供`SubscribeOnDispatcher()`和`ObserveOnDispatcher()`擴充函式，可幫助你存取Dispatcher。雖然它們看起來很有用，你應在產出的程式碼中避免使用，我們在[Testing Rx](16_TestingRx.md)中會說明。

上述所列的schedulers都擁有良好的命名，你應可以望文知義。後面會更深入的講解。併發應用程式可能會在除錯、測試和重構等領域面臨維護上的問題。

##Concurrency pitfalls

在你的軟體中引入併發性會增加複雜度。如果引入一層併發操作不能明顯的改進軟體，你應該避免引入。

引入併發後常見的問題是無法預測的時序。此無法預測的時序可能由系統上的不同負載及配置（例如，改變核心時脈速度或處理器的數量）的變化引起，最終可能導致race conditions。
Symptoms of race conditions include out-of-order execution, [deadlocks](http://en.wikipedia.org/wiki/Deadlock), [livelocks](http://en.wikipedia.org/wiki/Deadlock#Livelock) and corrupted state.

在我看來，偶然地在軟體中引入併發的最大危險是可能無聲地導入bug。這些缺陷可能會安然通過開發、品檢和UAT，並只在生產環境中出現！

然而，Rx做了很多工作以簡化對可觀察序列的併發處理，讓你減少對上述問題的疑慮。雖然你可能仍會出現問題，但若是你遵循指南，你會覺得較安全，應你有了可大大陂低不必要的race conditions的知識。

後續章節，[Testing Rx](16_TestingRx.md)，我們會看到Rx如何提高測試併發工作流程的能力。

###Lock-ups

當我在做第一個使用Rx的商用軟體時，團隊發現the hard way that Rx code can most certainly deadlock。當你認為一些函式（比如`First`，`Last`，`Single`和`ForEach`）是阻塞式的，而我們可以將工作排程以在未來呼叫，很顯然的race condition可能會發生。這個範例是我所能想到的最簡單的阻塞式程式，誠然，這是相當基本的，然而這可以展現我想表達的。
```csharp
	var sequence = new Subject<int>();
	Console.WriteLine("Next line should lock the system.");
	var value = sequence.First();
	sequence.OnNext(1);
	Console.WriteLine("I can never execute....");
```
希望我們不會寫出這樣的程式，如果我們寫了，我們的測試會快速的回饋，讓我們知道問題發生。更實際地，race condition常常會在整合的部位進入程式中。下一個範例可能有點難以察覺，但對比上一個不太現實的例子，這只進了幾步。這裡，我們故意在一個會在dispatcher中建立的UI元素的建構式中阻塞，此阻塞在等待僅會從dispatcher產生的事件，因此造成了死鎖。

```csharp
	public Window1()
	{
		InitializeComponent();
		DataContext = this;
		Value = "Default value";
		//Deadlock! 
		//We need the dispatcher to continue to allow me to click the button to produce a value
		Value = _subject.First();
		//This will give same result but will not be blocking (deadlocking). 
		_subject.Take(1).Subscribe(value => Value = value);
	}
	private void MyButton_Click(object sender, RoutedEventArgs e)
	{
		_subject.OnNext("New Value");
	}
	public string Value
	{
		get { return _value; }
		set
		{
			_value = value;
			var handler = PropertyChanged;
			if (handler != null) handler(this, new PropertyChangedEventArgs("Value"));
		}
	}
```

接下來，我們開始看到可能會更加危險的事。這個按鈕的事件處理器會試著從一個可觀察序列的介面取得第一個值：

```csharp
	public partial class Window1 : INotifyPropertyChanged
	{
		//Imagine DI here.
		private readonly IMyService _service = new MyService(); 
		private int _value2;

		public Window1()
		{
			InitializeComponent();
			DataContext = this;
		}
		public int Value2
		{
			get { return _value2; }
			set
			{
				_value2 = value;
				var handler = PropertyChanged;
				if (handler != null) handler(this, new PropertyChangedEventArgs("Value2"));
			}
		}
		#region INotifyPropertyChanged Members
		public event PropertyChangedEventHandler PropertyChanged;
		#endregion
		private void MyButton2_Click(object sender, RoutedEventArgs e)
		{
			Value2 = _service.GetTemperature().First();
		}
	}
```

這裡只有一個小問題，我們阻塞了`Dispatcher`執行緒（`First`是阻塞式呼叫），但若如下服務程式寫的不正確，就會產生死鎖。

```csharp
	class MyService : IMyService
	{
		public IObservable<int> GetTemperature()
		{
			return Observable.Create<int>(
				o =>
				{
					o.OnNext(27);
					o.OnNext(26);
					o.OnNext(24);
					return () => { };
				})
			   .SubscribeOnDispatcher();
		}
	}
```

這個奇怪的排程的實作，會導致三個`OnNext`的呼叫被排程直到`First()`結束；然而，`that`正在等待`OnNext`被呼叫：我們產生死鎖！

到目前為止，注意我們可能面對的問題，本章似乎可以說併發是所有錯誤的淵源，然而，這不是目的，我們不是想簡單地透過採用Rx來神奇的避免併發性問題，只要你遵循以下兩個原則，就可較簡單的透過Rx將事情做對：

 * 只有最後的訂閱者應設定排程
 * 避免使用阻塞式呼叫，如`First`、`Last`及`Single`等

The last example came unstuck with one simple problem; the service was dictating the scheduling paradigm when, really, it had no business doing so.在我們最終瞭解該在那裡執行排程時，我們在每一“層”都加了“有用的”排程程式。它最終創造的是一個執行緒的惡夢。當我們最終刪除所有的排程程式，然後將其限制在單一層上（至少在Silverlight client），大多數併發問題消失了。我建議你用同樣的方式。至少在WPF/Silverlight應用程式中，模式應很簡單："在背景執行緒訂閱，在Dispatcher中Observe"。

##Advanced features of schedulers

到目前為止，我們只談論到最簡單的排程使用：
  
 * 排程一個要馬上執行的動作
 * 排程一個可觀察序列的訂閱
 * Scheduling the observation of notifications coming from an observable sequence排程來自一個可觀察序列的訂閱的通知

排程器也有提供一些進階的功能以在不同問題上幫助你。

###Passing state

我們已談論的`IScheduler`的擴充函式中，你只能夠提供一個`Action`等待執行，此`Action`無法接受任何參數，如果你想傳一個狀態進`Action`，你可以用一個closure以共享資料，如下：
```csharp
	var myName = "Lee";
	Scheduler.NewThread.Schedule(
		() => Console.WriteLine("myName = {0}", myName));
```
這可能造成一個問題，因為你在兩個範圍中共享同一個資料，我可能因修改到`myName`變數，而得到未預期的結果。

這個範例中，我們使用如上所述的closure並傳入狀態，然後我馬上修改變數值，這就造成了race condition：我的修改會先執行，還是在排程中的先？
```csharp
	var myName = "Lee";
	scheduler.Schedule(
		() => Console.WriteLine("myName = {0}", myName));
	myName = "John";//What will get written to the console?
```
在我的測試中，如果`scheduler`是`NewThreadScheduler`的一個實體，那"John"會先被寫入至console中，如果我使用`ImmediateScheduler`排程，那"Lee"會被寫入。這問題是出在程式本身就無法預先斷定結果的狀態。

This example takes advantage of this overload, giving us certainty about our state.
較好的傳入狀態的方法是使用有接受參數的`Schedule`的覆載，我們在這個範例使用此覆載，讓我們能夠確定state。

```csharp
	var myName = "Lee";
	scheduler.Schedule(myName, 
		(_, state) =>
		{
			Console.WriteLine(state);
			return Disposable.Empty;
		});
	myName = "John";
```

這裡，我們將`myName`當成狀態傳入，同時也傳入一個委託，此委託需代入一個狀態參數，且會回傳一個disposable；這個disposable是用來做cancellation，稍後會深入瞭解。而這個委託也代入一個`IScheduler`參數，我們將其命名為"_"（底線），這表示我們會略過這一個參數。當我們傳入`myName`當做狀態，其內部會保存此狀態值，所以我們將`myName`設成"John"，在排程器內其保存的值仍為"Lee"。

注意我們上一個範例，我們修改`myName`變數，並指出會有一個新的字串實體。如果我們換用一個需修改的實體，仍然可能得到未預期的結果。下一個範例中，我們用一個list當做狀態變數，在排程一個action後，我們顯示list的計數，然後修改這一個list。

```csharp
	var list = new List<int>();
	scheduler.Schedule(list,
		(innerScheduler, state) =>
		{
			Console.WriteLine(state.Count);
			return Disposable.Empty;
		});
	list.Add(1);
```
現在我們修改了狀態，我們仍會得到不可預期的結果。在這個範例中，我們不知道scheduler的型別，所以我們也無法預測我們會面對的race conditions。所以在任何併發的軟體，都應該避免修改共用狀態（資料）。

###Future scheduling

如你所預期的，使用`IScheduler`型別，你可以將一個動作排程至未來才執行。你可以靠著指定一個時間或是週期性的時間讓動作被執行。這對buffering或是timers等功能很有幫助。

排程到未來執行可靠兩種可能的風格，一個可代入`TimeSpan`，另一個是`DateTimeOffset`。有兩種最簡單的覆載：

```csharp
	public static IDisposable Schedule(
		this IScheduler scheduler, 
		TimeSpan dueTime, 
		Action action)
	{...}
	public static IDisposable Schedule(
		this IScheduler scheduler, 
		DateTimeOffset dueTime, 
		Action action)
	{...}
```
你可以像這樣使用`TimeSpan`覆載：
```csharp
	var delay = TimeSpan.FromSeconds(1);
	Console.WriteLine("Before schedule at {0:o}", DateTime.Now);
	scheduler.Schedule(delay, 
		() => Console.WriteLine("Inside schedule at {0:o}", DateTime.Now));
	Console.WriteLine("After schedule at  {0:o}", DateTime.Now);
```
輸出：
```dos
Before schedule at 2012-01-01T12:00:00.000000+00:00
After schedule at 2012-01-01T12:00:00.058000+00:00
Inside schedule at 2012-01-01T12:00:01.044000+00:00
```
因為'before'和'after'呼叫的時間點非常接近，所以我們可以得知排程是非阻塞式的。也可以看到大概過了一秒後，排程的動作被呼叫執行。

你也可以用`DateTimeOffset`覆載指定一個特定的時間點以排程工作，如果因一些原因，你所指定的時間已過去了，你排程的動作會儘快被執行。

###Cancelation

這些對`Schedule`的覆載都會回傳一個`IDisposable`；用這個方式，使用者可以取消排程的工作。在上一個範例中，我們將工作排在一秒後執行，我們可以disposing這一個token去取消工作。

```csharp
	var delay = TimeSpan.FromSeconds(1);
	Console.WriteLine("Before schedule at {0:o}", DateTime.Now);
	var token = scheduler.Schedule(delay, 
		() => Console.WriteLine("Inside schedule at {0:o}", DateTime.Now));
	Console.WriteLine("After schedule at  {0:o}", DateTime.Now);
	token.Dispose();
```
輸出：
```dos
Before schedule at 2012-01-01T12:00:00.000000+00:00
After schedule at 2012-01-01T12:00:00.058000+00:00
```
Note that the scheduled action never occurs, as we have cancelled it almost immediately.
注意被排程的動作沒有執行，應因為我們幾乎馬上就把它取消。當使用者在排程工作執行前就將它取消，這個工作就從佇列中被移除。這是上面我們看到的行為。如果你想取消一個正在被執行的工作，你可以使用另一個`Schedule`的覆載，它需要代入一個`Func<IDisposable>`參數，這讓使用者可以取消一個執行中的工作，可能是一個I/O、大量計算或是一個`Task`中的工作。

現在這可能會造成一個問題；如果你想取消一個已經開始的工作，你要dispose一個`IDisposable`型別的實體，但如果你正在執行，你要如何回傳一它的disposable？你可能啟動另一個執行緒以同步執行工作，但執行緒是我們試著要避免的。

在這一個範例，我們有一個將被排程的委託函式，它只是用等待來假裝執行一些工作，並在`list`中加入值，這裡的關鍵是使用者可透過我們回傳的disposable去取消這一個`CancellationToken`。
```csharp
	public IDisposable Work(IScheduler scheduler, List<int> list)
	{
		var tokenSource = new CancellationTokenSource();
		var cancelToken = tokenSource.Token;
		var task = new Task(() =>
		{
			Console.WriteLine();
			for (int i = 0; i < 1000; i++)
			{
				var sw = new SpinWait();
				for (int j = 0; j < 3000; j++) sw.SpinOnce();
				Console.Write(".");
				list.Add(i);
				if (cancelToken.IsCancellationRequested)
				{
					Console.WriteLine("Cancelation requested");
					//cancelToken.ThrowIfCancellationRequested();
					return;
				}
			}
		}, cancelToken);
		task.Start();
		return Disposable.Create(tokenSource.Cancel);
	}
```
這個範例中我們將上述程式排程，並允許使用者按下Enter來取消工作。
```csharp
	var list = new List<int>();
	Console.WriteLine("Enter to quit:");
	var token = scheduler.Schedule(list, Work);
	Console.ReadLine();
	Console.WriteLine("Cancelling...");
	token.Dispose();
	Console.WriteLine("Cancelled");
```
輸出：
```dos
Enter to quit:
........
Cancelling...
Cancelled
Cancelation requested
```
這裡的問題是我們額外引入了`Task`的使用。如果我們使用Rx的recursive scheduler，可以避免併發模式的使用。

###Recursion

較進階的`Schedule`擴充函式的覆載需代入一些看起來怪怪的委託當做參數。要特別注意每個覆載的最後一個參數。

```csharp
	public static IDisposable Schedule(
		this IScheduler scheduler, 
		Action<Action> action)
	{...}
	public static IDisposable Schedule<TState>(
		this IScheduler scheduler, 
		TState state, 
		Action<TState, Action<TState>> action)
	{...}
	public static IDisposable Schedule(
		this IScheduler scheduler, 
		TimeSpan dueTime, 
		Action<Action<TimeSpan>> action)
	{...}
	public static IDisposable Schedule<TState>(
		this IScheduler scheduler, 
		TState state, 
		TimeSpan dueTime, 
		Action<TState, Action<TState, TimeSpan>> action)
	{...}
	public static IDisposable Schedule(
		this IScheduler scheduler, 
		DateTimeOffset dueTime, 
		Action<Action<DateTimeOffset>> action)
	{...}
	public static IDisposable Schedule<TState>(
		this IScheduler scheduler, 
		TState state, DateTimeOffset dueTime, 
		Action<TState, Action<TState, DateTimeOffset>> action)
	{...}   
```
這些覆載都需要一個委託"action"，以讓你可以遞迴地呼叫此"action"，這定義看起來很奇怪，但它是很有用的API。它允許你很有效率地建立遞迴呼叫，下面用範例來說明。
這個範例使用最簡單的遞迴覆載。我們有一個可被遞回呼叫的`Action`。
```csharp
	Action<Action> work = (Action self) 
		=>
		{
			Console.WriteLine("Running");
			self();
		};
	var token = s.Schedule(work);
		
	Console.ReadLine();
	Console.WriteLine("Cancelling");
	token.Dispose();
	Console.WriteLine("Cancelled");
```
Output:
```dos
Enter to quit:
Running
Running
Running
Running
Cancelling
Cancelled
Running
```
注意我們不需要在委託寫任何取消的程式碼，Rx幫我們處理掉遞迴及檢查是否取消的動作，聰明吧！不像C#中的簡單遞迴函式，我們也受到stack overflows的保護，因Rx有提供額外的抽象等級保護。是的，Rx讓我們代入遞迴函式並將它轉成loop結構。

####Creating your own iterator

Earlier in the book, we looked at how we can use [Rx with APM](04_CreatingObservableSequences.html#FromAPM). 
In our example, we just read the entire file into memory. 
We also referenced Jeffrey van Gogh's [blog post](http://blogs.msdn.com/b/jeffva/archive/2010/07/23/rx-on-the-server-part-1-of-n-asynchronous-system-io-stream-reading.aspx), which sadly is now out of date; however, his concepts are still sound.
Instead of the Iterator method from Jeffrey's post, we can use schedulers to achieve the same result.

The goal of the following sample is to open a file and stream it in chunks. 
This enables us to work with files that are larger than the memory available to us, as we would only ever read and cache a portion of the file at a time. 
In addition to this, we can leverage the compositional nature of Rx to apply multiple transformations to the file such as encryption and compression. 
By reading chunks at a time, we are able to start the other transformations before we have finished reading the file.

First, let us refresh our memory with how to get from the `FileStream`'s APM methods into Rx.

在本書的前面，我們瞭解如何使用[Rx with APM]（04_CreatingObservableSequences.md＃FromAPM）。在我們的範例中，我們只是將整個文件讀入記憶體。
我們還引用了Jeffrey van Gogh的Blog（http://blogs.msdn.com/b/jeffva/archive/2010/07/23/rx-on-the-server-part-1-of-n-asynchronous -system-io-stream-reading.aspx），遺憾的是已過時了;然而，他的概念仍然可行。只是我們是使用scheduler而不是Jeffrey的文章中的Iterator方法。

以下範例的目標是打開一個檔案並以chucks為單位進行串流傳輸。這使我們能夠處理比我們可用的記憶體大的檔案，因為我們每次只讀取並快取檔案的一部分。除此之外，我們可以利用Rx的合成本質來對檔案應用多種轉換，例如加密和壓縮。通過一次讀取一個chuck，我們能夠在我們完成讀取檔案之前啟動其他的轉換。

首先，讓我們回憶一下，如何用APM從`FileStream`進入Rx。
```csharp
	var source = new FileStream(@"C:\Somefile.txt", FileMode.Open, FileAccess.Read);
	var factory = Observable.FromAsyncPattern<byte[], int, int, int>(
		source.BeginRead, 
		source.EndRead);
	var buffer = new byte[source.Length];
	IObservable<int> reader = factory(buffer, 0, (int)source.Length);
	reader.Subscribe(
		bytesRead => 
			Console.WriteLine("Read {0} bytes from file into buffer", bytesRead));
```		
上述的範例使用`FromAsyncPattern`建立了factory，這個factory需要代入一個byte陣列(`buffer`)，一個偏移量(`0`)及長度(`source.Length`)；它回傳單一元素的序列的bytes計數值。當這個序列(`reader`)被訂閱，`BeginRead`會從偏移位址開始讀取數值進入buffer中，這個範例中，我們會讀取整個檔案。一旦檔案被讀進buffer中，序列(`reader`)會推送值(`bytesRead`)進入序列中。

這很好，但如果我們想一次讀取所有資料這就不夠了。我們需可以指定想使用的緩衝區大小。讓我們從4kb(4096 bytes)。
```csharp
	var bufferSize = 4096;
	var buffer = new byte[bufferSize];
	IObservable<int> reader = factory(buffer, 0, bufferSize);
	reader.Subscribe(
		bytesRead => 
			Console.WriteLine("Read {0} bytes from file", bytesRead));
```

這程式可以動作，但只能讀取最大4kb的檔案。如果檔案更大，我們會想整個讀取完。因為`FileStream`的`Position`會指向最後讀取的位置，我們可以重用`factory`去載入緩衝區，再來，我們想開始推送這些資料至一個可觀察序列，讓我們先定義我們的擴充函式。

```csharp
	public static IObservable<byte> ToObservable(
		this FileStream source, 
		int buffersize, 
		IScheduler scheduler)
	{...}
```

靠著使用`Observable.Create`我們可以確定我們的擴充函式是延遲估值的，而由於使用`Observable.Usign`運算子，我們也可以知道使用取dispose後`FileStream`會被關閉。

```csharp
	public static IObservable<byte> ToObservable(
		this FileStream source, 
		int buffersize, 
		IScheduler scheduler)
	{
		var bytes = Observable.Create<byte>(o =>
		{
			...
		});

		return Observable.Using(() => source, _ => bytes);
	}
```
下一步，我們要使用scheduler的recursive功能以在提供使用者可以dispose/cancel的同時將資料連續的讀出來。This creates a bit of a pickle；我們只可以傳入一個狀態參數，但是需要管理很多部份(buffers, factory, filestream)，為了達成目的，我們建立自己的協助類別：
```csharp
	private sealed class StreamReaderState
	{
		private readonly int _bufferSize;
		private readonly Func<byte[], int, int, IObservable<int>> _factory;

		public StreamReaderState(FileStream source, int bufferSize)
		{
			_bufferSize = bufferSize;
			_factory = Observable.FromAsyncPattern<byte[], int, int, int>(
				source.BeginRead, 
				source.EndRead);
			Buffer = new byte[bufferSize];
		}

		public IObservable<int> ReadNext()
		{
			return _factory(Buffer, 0, _bufferSize);
		}

		public byte[] Buffer { get; set; }
	}
```
這一個類別讓我們將資料讀入buffer，然後呼叫`ReadNext()`讀取下一個區塊。在我們的`Observable.Create`委託中，我們建立協助類別並使用它將資料推送至我們的可觀察序列。
```csharp
	public static IObservable<byte> ToObservable(
		this FileStream source, 
		int buffersize, 
		IScheduler scheduler)
	{
		var bytes = Observable.Create<byte>(o =>
		{
			var initialState = new StreamReaderState(source, buffersize);

			initialState
				.ReadNext()
				.Subscribe(bytesRead =>
				{
					for (int i = 0; i < bytesRead; i++)
					{
						o.OnNext(initialState.Buffer[i]);
					}
				});
			...
		});

		return Observable.Using(() => source, _ => bytes);
	}
```
So this gets us off the ground，但我們仍不支援大於緩衝區的檔案。現在，我們要加上遞迴排程。為此我們需要一個委託來滿足所需的定義。我們需要一個可以接受`StreamReaderState`，且可以遞迴呼叫`Action<StreamReaderState>`的委託。
```csharp
	public static IObservable<byte> ToObservable(
		this FileStream source, 
		int buffersize, 
		IScheduler scheduler)
	{
		var bytes = Observable.Create<byte>(o =>
		{
			var initialState = new StreamReaderState(source, buffersize);

			Action<StreamReaderState, Action<StreamReaderState>> iterator;
			iterator = (state, self) =>
			{
				state.ReadNext()
					 .Subscribe(bytesRead =>
							{
								for (int i = 0; i < bytesRead; i++)
								{
									o.OnNext(state.Buffer[i]);
								}
								self(state);
							});
			};
			return scheduler.Schedule(initialState, iterator);
		});

		return Observable.Using(() => source, _ => bytes);
	}
```
現在我們有一個`iterator`的action，它會：

 * 呼叫`ReadNext()`
 * 訂閱結果
 * 推送buffer至可觀察序列
 * 遞迴呼叫自己
 
我們也會在提供的排程器中排程待執行的遞迴工作，下一步，我們會在檔案結束時也結束序列，這很簡我們維持遞迴程序直到`bytesRead`為0。
```csharp
	public static IObservable<byte> ToObservable(
		this FileStream source, 
		int buffersize, 
		IScheduler scheduler)
	{
		var bytes = Observable.Create<byte>(o =>
		{
			var initialState = new StreamReaderState(source, buffersize);

			Action<StreamReaderState, Action<StreamReaderState>> iterator;
			iterator = (state, self) =>
			{
				state.ReadNext()
					 .Subscribe(bytesRead =>
							{
								for (int i = 0; i < bytesRead; i++)
								{
									o.OnNext(state.Buffer[i]);
								}
								if (bytesRead > 0)
									self(state);
								else
									o.OnCompleted();
							});
			};
			return scheduler.Schedule(initialState, iterator);
		});

		return Observable.Using(() => source, _ => bytes);
	}
```
現在，我們有一個擴充函式，它會迭代檔串流案中的bytes。最後，讓我們釐清一下，以正確的管理我們的資源和例外，函式最後看起來像：
```csharp
	public static IObservable<byte> ToObservable(
		this FileStream source, 
		int buffersize, 
		IScheduler scheduler)
	{
		var bytes = Observable.Create<byte>(o =>
		{
			var initialState = new StreamReaderState(source, buffersize);
			var currentStateSubscription = new SerialDisposable();
			Action<StreamReaderState, Action<StreamReaderState>> iterator =
			(state, self) =>
				currentStateSubscription.Disposable = state.ReadNext()
					 .Subscribe(
						bytesRead =>
						{
							for (int i = 0; i < bytesRead; i++)
							{
								o.OnNext(state.Buffer[i]);
							}

							if (bytesRead > 0)
								self(state);
							else
								o.OnCompleted();
						},
						o.OnError);

			var scheduledWork = scheduler.Schedule(initialState, iterator);
			return new CompositeDisposable(currentStateSubscription, scheduledWork);
		});

		return Observable.Using(() => source, _ => bytes);
	}
```
這是示範程式，而你的可能會有所不同。我發現增加緩衝區大小和回傳`IObservable<IList<byte>>`會更好，但上面的範例也很好。這裡的目標是提供一個iterator的示範，它提供了具有取消和資源高效緩衝的併發I/O存取。

<!--<a name="ScheduledExceptions"></a>
<h4>Exceptions from scheduled code</h4>
<p>
	TODO:
</p>-->

####Combinations of scheduler features

我們已經討論了很多`IScheduler`介面的相關特性，然而大多數的範例，實際上是使用擴充函式來呼叫我們尋找的功能，這個介面本身擁有更豐富的覆載。擴充函式本身只是做出權衡，透過減少豐富的覆載來增進可用性/可發現性。如果你想訪問傳送狀態、取消、排程和遞迴，都可以直接通過介面來使用。

```csharp
	namespace System.Reactive.Concurrency
	{
	  public interface IScheduler
	  {
		//Gets the scheduler's notion of current time.
		DateTimeOffset Now { get; }

		// Schedules an action to be executed with given state. 
		//  Returns a disposable object used to cancel the scheduled action (best effort).
		IDisposable Schedule<TState>(
			TState state, 
			Func<IScheduler, TState, IDisposable> action);

		// Schedules an action to be executed after dueTime with given state. 
		//  Returns a disposable object used to cancel the scheduled action (best effort).
		IDisposable Schedule<TState>(
			TState state, 
			TimeSpan dueTime, 
			Func<IScheduler, TState, IDisposable> action);

		//Schedules an action to be executed at dueTime with given state. 
		//  Returns a disposable object used to cancel the scheduled action (best effort).
		IDisposable Schedule<TState>(
			TState state, 
			DateTimeOffset dueTime, 
			Func<IScheduler, TState, IDisposable> action);
	  }
	}
```
##Schedulers in-depth

我們在很大的程度上關注的是排程器和`IScheduler`介面的抽象概念。這種抽象允許low-level plumbing不用知道併發模型。如同上述範例的檔案讀取器，程式不需要知道`IScheduler`的那一個實作被傳入，因為這是使用者程式的問題。
現在讓我們深入瞭解`IScheduler`的每一種實作，考慮它們個自的優點和權衡，以及是否適合使用。

###ImmediateScheduler

`ImmediateScheduler`是透過`Scheduler.Immediate`靜態屬性公開。這是最簡單的排程器，實際上它並沒有執行任何排程。如果你呼叫`Schedule(Action)`，它會直接喚起這一個action，如果你將action排程至未來執行，`ImmediateScheduler`會喚起一個`Thread.Sleep`並代入指定的時間，然後再執行動作。總之，`ImmediateScheduler`是同步的。

###CurrentThreadScheduler

就像`ImmediateScheduler`，`CurrentThreadScheduler`也是單執行緒。它透過`Scheduler.Current`靜態屬性公開。主要的不同是`CurrentThreadScheduler`的動作就像一個訊息佇列或一個_Trampoline_，如果你排程了一個工作，這工作本身也排程了另一工作，`CurrentThreadScheduler`會將其放入至內部的佇列中以待執行；在約定上，`ImmediateScheduler`會直接在內部的工作上執行。這最好透過範例來說明。

這個範例中，我們分析了`ImmediateScheduler`和`CurrentThreadScheduler`如何以不同的方式執行巢狀排程。
```csharp
	private static void ScheduleTasks(IScheduler scheduler)
	{
		Action leafAction = () => Console.WriteLine("----leafAction.");
		Action innerAction = () =>
		{
			Console.WriteLine("--innerAction start.");
			scheduler.Schedule(leafAction);
			Console.WriteLine("--innerAction end.");
		};
		Action outerAction = () =>
		{
			Console.WriteLine("outer start.");
			scheduler.Schedule(innerAction);
			Console.WriteLine("outer end.");
		};
		scheduler.Schedule(outerAction);
	}
	public void CurrentThreadExample()
	{
		ScheduleTasks(Scheduler.CurrentThread);
		/*Output: 
		outer start. 
		outer end. 
		--innerAction start. 
		--innerAction end. 
		----leafAction. 
		*/ 
	}
	public void ImmediateExample()
	{
		ScheduleTasks(Scheduler.Immediate);
		/*Output: 
		outer start. 
		--innerAction start. 
		----leafAction. 
		--innerAction end. 
		outer end. 
		*/ 
	}
```

注意`ImmediateScheduler`並不對工作排程，它只是馬上執行工作(同步地)。只要`Schedule`被委託呼叫，此委託同時被喚起。然而，會喚起第一個委託，然後，當巢狀委託被排程，會將其放入佇列以等待被執行。一旦起始委託完成，佇列會檢查是否還有委託(例：對`Schedule`的巢狀呼叫)，並且執行它們。這裡的區別是很重要的，因為你用錯的話，可能會以錯誤的順序執行、未預期的阻塞或甚至發生deadlock。

###DispatcherScheduler

`DispatcherScheduler`是在`System.Reactive.Window.Threading.dll`裡面(for WPF, Silverlight 4 and Silverlight 5)的。當使用`DispatcherScheduler`排程，它們被有效的整合進`Dispatcher`的`BeginInvoke`函式中，且會將工作加至dispatcher的優先權為_Normal_等級的佇列末端，這很像`CurrentThreadScheduler`面對巢狀`Schedule`呼叫時的處理方式。

當一個工作被排程以等待執行，一個`DispatcherTimer`會被建立，並指定匹配的時間間隔。這個timer的callback tick會停止timer，並重新將工作排程進`DispatcherScheduler`。如果`DispatcherScheduler`判定`dueTime`不是未來的時間，沒有timer會被建立，而此工作會被正常排程。

我想強調使用`DispatcherScheduler`的危險。你可以傳入一個`Dispatcher`的參考來建立自己的`DispatcherScheduler`實體，另一種方法是透過靜態屬性`DispatcherScheduler.Instance`。如果沒有正確使用，可能會引入難以理解的問題。
The static property does not return a reference to a static field, but creates a new instance each time, with the static property `Dispatcher.CurrentDispatcher` as the constructor argument. 
If you access `Dispatcher.CurrentDispatcher` from a thread that is not the UI thread, it will thus give you a new instance of a `Dispatcher`, but it will not be the instance you were hoping for.

例如，假設我們有一個WPF應用程序與`Observable.Create`方法。
在我們傳遞給`Observable.Create`的委託中，我們想要在dispatcher上將通知排程。我們認為這是一個好主意，因為序列的任何消費者都可以免費獲得dispatcher的通知。
```csharp
	var fileLines = Observable.Create<string>(
		o =>
		{
			var dScheduler = DispatcherScheduler.Instance;
			var lines = File.ReadAllLines(filePath);
			foreach (var line in lines)
			{
				var localLine = line;
				dScheduler.Schedule(
					() => o.OnNext(localLine));
			}
			return Disposable.Empty;
		});
```
這段程式直覺上看起來可能是對的，但實際上拿走了使用者對序列操作的能力。當我們訂閱這個序列，決定在UI執行緒當中讀取檔案是一個壞主意。所以我們在鏈接中加上`SubscribeOn(Scheduler.NewThread)`，如下：
```csharp
	fileLines
		.SubscribeOn(Scheduler.ThreadPool)
		.Subscribe(line => Lines.Add(line));
```

這會導致在新的執行緒上執行新建的委託。這個委託會讀取檔案，並取得一個`DispatcherScheduler`的實體。`DispatcherScheduler`會試著在當前執行緒中取得`Dispatcher`，但我們已經不在UI執行緒了，所以沒有`Dispatcher`存在。因此，它建立了一個用於`DispatcherScheduler`實體的新dispatcher。我們排程了一些工作(通知)，但，因為底層的`Dispatcher`沒有執行，所以沒有任何工作被執行，我們甚至不會得到例外通知！我曾在一個商業專案中看過這個狀況，讓一些人傷透了腦袋。

這讓我們有了一個關於排程的指南：**對`SubscribeOn`和`ObserveOn`的使用，應該在最後的subscriber才被喚起**。如果你在自己的擴充函式或服務函式中引入排程，你應讓使用者指定自己的排程器。我們在下一章會看到這個指南的更多理由。

###EventLoopScheduler

`EventLoopScheduler`讓你設計一個特定的執行緒以排程。就像巢狀排程中做起來像trampoline的`CurrentThreadScheduler`，提供了相同的trampoline機制。不同的是你會提供給`EventLoopScheduler`一個你想用它來排程的執行緒，而不是用當前執行緒。

`EventLoopScheduler`可以以空建構式建立，或傳入一個執行緒的factory委託。
```csharp
	// Creates an object that schedules units of work on a designated thread.
	public EventLoopScheduler()
	{...}

	// Creates an object that schedules units of work on a designated thread created by the 
	//  provided factory function.
	public EventLoopScheduler(Func&lt;ThreadStart, Thread> threadFactory)
	{...}
```
這個覆載允許你傳入一個工廠方法以讓你在指定給`EventLoopScheduler`前自訂執行緒。例如，你可以設定執行緒名稱、優先權、culture及最重要的是否是背景執行緒。記注如果你沒有設定執行緒屬性`IsBackground`為false，你的程式無法結束，除非這個執行緒也結束。`EventLoopScheduler`實作了`IDisposable`，所以呼叫Dispose會中斷這個執行緒。如同其它`IDisposable`的實作，你可以明確的管理你所建立的資源的生命週期。

如果你喜歡，這很適合和`Observable.Using`函式一起使用。這讓你可以綁定你的`EventLoopScheduler`的生命週期到一個可觀察序列上，例如：`GetPrices`函式代入一個`IScheduler`當作參數，並回傳一個可觀察序列。

```csharp
	private IObservable&lt;Price> GetPrices(IScheduler scheduler)
	{...}
```
這裡我們綁定`EventLoopScheduler`的生命週期到`GetPrices`函式的結果上。
```csharp
	Observable.Using(()=>new EventLoopScheduler(), els=> GetPrices(els))
			.Subscribe(...)
```
###New Thread

如果你不想管理執行緒或一個`EventLoopScheduler`的資源，那你也可以使用`NewThreadScheduler`。你可以建立自己的`NewThreadScheduler`的實體，或透過`Scheduler.NewThread`屬性存取它的靜態實體。就像`EventLoopScheduler`，你可以使用無參數的建構式或提供你自己的執行緒工廠函式。如果你提供自己的工廠函式，要小心`IsBackground`屬性值的設定。

當你在`NewThreadScheduler`上面呼叫`Schedule`，你實際上是建了一個`EventLoopScheduler`。這種方式，任何巢狀的排程會在同一個執行緒發生。子序列(非巢狀)對`Schedule`的呼叫會建立一個新的`EventLoopScheduler`，並且呼叫執行緒工廠函式建立新執行緒。

這個範例中，我們執行了一小段程式，並想起之前對`Immediate`和`Current`兩個schedulers的比較。然而這裡的不同是，我們追蹤執行的`ThreadId`屬性。我們使用`Schedule`覆載，這讓我們可以傳Scheduler實體進入我們的巢狀委託，以允許我們正確的使用巢狀呼叫。

```csharp
	private static IDisposable OuterAction(IScheduler scheduler, string state)
	{
		Console.WriteLine("{0} start. ThreadId:{1}", 
			state, 
			Thread.CurrentThread.ManagedThreadId);
		scheduler.Schedule(state + ".inner", InnerAction);
		Console.WriteLine("{0} end. ThreadId:{1}", 
			state, 
			Thread.CurrentThread.ManagedThreadId);
		return Disposable.Empty;
	}
	private static IDisposable InnerAction(IScheduler scheduler, string state)
	{
		Console.WriteLine("{0} start. ThreadId:{1}", 
			state, 
			Thread.CurrentThread.ManagedThreadId);
		scheduler.Schedule(state + ".Leaf", LeafAction);
		Console.WriteLine("{0} end. ThreadId:{1}", 
			state, 
			Thread.CurrentThread.ManagedThreadId);
		return Disposable.Empty;
	}
	private static IDisposable LeafAction(IScheduler scheduler, string state)
	{
		Console.WriteLine("{0}. ThreadId:{1}", 
			state, 
			Thread.CurrentThread.ManagedThreadId);
		return Disposable.Empty;
	}
```
我們像這樣和`NewThreadScheduler`執行：
```csharp
	Console.WriteLine("Starting on thread :{0}", 
		Thread.CurrentThread.ManagedThreadId);
	Scheduler.NewThread.Schedule("A", OuterAction);
```
輸出：
```dos
Starting on thread :9
A start. ThreadId:10
A end. ThreadId:10
A.inner start . ThreadId:10
A.inner end. ThreadId:10
A.inner.Leaf. ThreadId:10
```
如你所見，結果很像，除了trampoline發生在另一個執行緒中。如果我們使用一個`EventLoopScheduler`也會得到相同的結果。而使用`EventLoopScheduler`和`NewThreadScheduler`不同的地方會開始於當我們引入第二個(非巢狀)排程的task。
```csharp
	Console.WriteLine("Starting on thread :{0}", 
		Thread.CurrentThread.ManagedThreadId);
	Scheduler.NewThread.Schedule("A", OuterAction);
	Scheduler.NewThread.Schedule("B", OuterAction);
```
輸出：
```dos
Starting on thread :9
A start. ThreadId:10
A end. ThreadId:10
A.inner start . ThreadId:10
A.inner end. ThreadId:10
A.inner.Leaf. ThreadId:10
B start. ThreadId:11
B end. ThreadId:11
B.inner start . ThreadId:11
B.inner end. ThreadId:11
B.inner.Leaf. ThreadId:11
```
注意現在這裡有三個執行緒，9號是我們開始的地方，10號、11號是我們兩個排程呼叫執行的地方。

###Thread Pool

`ThreadPoolScheduler`只是一個將請求轉至的`ThreadPool`通道。對儘快執行的排程請求，這個工作會被傳送至`ThreadPool.QueueUserWorkItem`。對未來工作排程的請求，會使用`System.Threading.Timer`。

因為所有的工作被送至`ThreadPool`，所以可能不如你預期的執行順序。不像上一個我們看的scheduler，巢狀呼叫並不保證被順序處理。我們可以用和上述相同的範例但使用`ThreadPoolScheduler`來展示這個行為。

```csharp
	Console.WriteLine("Starting on thread :{0}", 
		Thread.CurrentThread.ManagedThreadId);
	Scheduler.ThreadPool.Schedule("A", OuterAction);
	Scheduler.ThreadPool.Schedule("B", OuterAction);
``
輸出：
```dos
Starting on thread :9
A start. ThreadId:10
A end. ThreadId:10
A.inner start . ThreadId:10
A.inner end. ThreadId:10
A.inner.Leaf. ThreadId:10
B start. ThreadId:11
B end. ThreadId:11
B.inner start . ThreadId:10
B.inner end. ThreadId:10
B.inner.Leaf. ThreadId:11
```
注意，每一個的測試，我們初始啟動於一執行緒中，但所有的排程發生在另兩個執行緒。The difference is that we can see that part of the second run "B" runs on thread 11 while another part of it runs on 10.

###TaskPool

`TaskPoolScheduler`非常類似於`ThreadPoolScheduler`，當可用時（取決於你的目標framework），你應該優先使用它。像`ThreadPoolScheduler`一樣，巢狀排程操作不能保證在同一個執行緒上執行。使用`TaskPoolScheduler`執行相同的測試顯示我們類似的結果。

```csharp
	Console.WriteLine("Starting on thread :{0}", 
		Thread.CurrentThread.ManagedThreadId);
	Scheduler.TaskPool.Schedule("A", OuterAction);
	Scheduler.TaskPool.Schedule("B", OuterAction);
```
輸出：
```dos
Starting on thread :9
A start. ThreadId:10
A end. ThreadId:10
B start. ThreadId:11
B end. ThreadId:11
A.inner start . ThreadId:10
A.inner end. ThreadId:10
A.inner.Leaf. ThreadId:10
B.inner start . ThreadId:11
B.inner end. ThreadId:11
B.inner.Leaf. ThreadId:10
```
###TestScheduler

值得注意的是還有一個`TestScheduler`型別，以及它的兩個父類別`VirtualTimeScheduler`和`VirtualTimeSchedulerBase`。後兩者際上不在Rx的介紹範圍內，但前一個是。我們會在下一章[Testing Rx](16_TestingRx.md)談論所有的測試，包含`TestScheduler`。

##Selecting an appropriate scheduler

由於有那麼多的選項可以選擇，常會讓人不知從何選起，這裡有一個簡單的檢查列表，以幫助你選取：

###UI Applications

 * 最後一個訂閱者通常會是presentation層，應負責控制排程
 * 在`DispatcherScheduler`上面觀察以允許對ViewModels的更新
 * 在背景執行緒上訂閱以避免UI無回應
   * 如果訂閱不會阻塞超過50ms，然後
     * 如果可以，用`TaskPoolScheduler`，或者
	 * 使用`ThreadPoolScheduler`
   * 會超過則應使用`NewThreadScheduler`
  
###Service layer

 * 如果你的服務是從一個佇列讀取資料(或類似行為)，考慮使用`EventLoopScheduler`，這個方式，你可以保持事件的順序
 * 如果處理一個項目很耗時(大於50ms或透過I/O)，考慮使用`NewThreadScheduler`
 * 如果只需要使用一個timer來排程，例如：`Observable.Interval`或`Observable.Timer`，那就用`TaskPool`
 
使用`ThreadPool`如果在你的平台上不存在`TaskPool`。

<p class="comment">
	The `ThreadPool` (and the `TaskPool` by proxy) have a time delay before	they will increase the number of threads that they use. 
	This delay is 500ms. 
	Let	us consider a PC with two cores that we will schedule four actions onto. 
	By default,	the thread pool size will be the number of cores (2). 
	If each action takes 1000ms, then two actions will be sitting in the queue for 500ms before the thread pool size is increased. 
	Instead of running all four actions in parallel, which would take one second in total, the work is not completed for 1.5 seconds as two of the actions sat in the queue for 500ms. 
	For this reason, you should only schedule work that	is very fast to execute (guideline 50ms) onto the ThreadPool or TaskPool. 
	Conversely,	creating a new thread is not free, but with the power of processors today the creation of a thread for work over 50ms is a small cost.
</p>

Rx解決了透過`ObserveOn`/`SubscribeOn`函式同步的產生及消費資料的問題。適當地使用Rx，我們可以讓我們的code base更簡單，增進響應性及減少我們對併發性的考慮。排程器提供很豐富的平台以協助處理併發性工作，而不需要直接面對執行緒的處理。它們也幫助處理常見的併發問題，如取消、傳遞狀態及遞迴等。通過減少併發的機率，Rx提供了一個(相對)簡單但功能強大的併發特性，鋪平了我們[pit of success](http://blogs.msdn.com/b/brada/archive/2003/10/02/50420.aspx)的道路。