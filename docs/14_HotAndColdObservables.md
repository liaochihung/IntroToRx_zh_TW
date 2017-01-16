---
title : Hot and Cold observables
---

<!--TODO: Observable.Synchronize (vNext? Say it is not intro material?)-->
<!--TODO: Observable.Defer - Make cold (vNext? Say it is not intro material?)-->

#Hot and Cold observables			{#HotAndCold}

這一章我們要瞭解兩種可觀察序列：

1. 僅在請求後產生推送值的被動式序列
2. 訂閱後隨即產生推送值的主動式序列 

在這個意義上，被動序列是“泠“的，主動序列是”熱“的。你可以在`IObservable<T>`介面和`IEnumerable<T>`介面的實作描繪相對的冷和熱。使用`IEnumerable<T>`，你可以透過yield return語法按需求產生集合，或透過回傳`List<T>`得到一個及早估值的集合。我們可以通過嘗試從序列中讀取第一個值來比較這兩種樣式，可使用如下函式：

```csharp
	public void ReadFirstValue(IEnumerable<int> list)
	{
		foreach (var i in list)
		{
			Console.WriteLine("Read out first value of {0}", i);
			break;
		}
	}
```
而與其用`break`指令，我們也可以在`list`上用`Take(1)`，如果再使用及早估值的序列，我們可以看到序列一建立後就回傳。
```csharp
	public static void Main()
	{
		ReadFirstValue(EagerEvaluation());
	}
	public IEnumerable<int> EagerEvaluation()
	{
		var result = new List<int>();
		Console.WriteLine("About to return 1");
		result.Add(1);
		//code below is executed but not used.
		Console.WriteLine("About to return 2");
		result.Add(2);
		return result;
	}
```
輸出：

<div class="output">
	<div class="line">About to return 1</div>
	<div class="line">About to return 2</div>
	<div class="line">Read out first value of 1</div>
</div>

現在在延後取值的序列使用同樣的程式碼：
```csharp
	public IEnumerable<int> LazyEvaluation()
	{
		Console.WriteLine("About to return 1");
		yield return 1;
		//Execution stops here in this example
		Console.WriteLine("About to return 2");
		yield return 2;
	}
```
輸出：

<div class="output">
	<div class="line">About to return 1</div>
	<div class="line">Read out first value of 1</div>
</div>

延後求值序列不會回傳比需求還多的數值。延後求值適合於on-demand的查詢，而及早估值適合於共享序列以避免重覆估值多次。而`IObservable<T>`的實作在風格上有類似的變化。

以下是不管有任何訂閱者都會推送資訊的"熱"的可觀察序列的例子：

 * mouse movements 
 * timer events 
 * broadcasts like ESB channels or UDP network packets. 
 * price ticks from a trading exchange 
 * 滑鼠移動
 * 時間的事件
 * 廣播，如ESB頻道或UDP網路封包 
 * 交易所的交易價格

"冷"的可觀察序列的範例：

 * 非同步的請求(例：當使用`Observable.FromAsyncPattern`時)
 * 使用`Observable.Create`時
 * 從佇列中訂閱時
 * `on-demand`序列

##Cold observables					{#ColdObservables}

這個範例中，我們從資料庫中取得一串商品資料。這個實作中，我們選擇回傳一個`IObservable<string>`，取得結果後，會發上發佈，直到取得整個串列，然後就結束序列。
```csharp
	private const string connectionString = @"Data Source=.\SQLSERVER;"+
		@"Initial Catalog=AdventureWorksLT2008;Integrated Security=SSPI;"
	private static IObservable<string> GetProducts()
	{
		return Observable.Create<string>(
		o =>
		{
			using(var conn = new SqlConnection(connectionString))
			using (var cmd = new SqlCommand("Select Name FROM SalesLT.ProductModel", conn))
			{
				conn.Open();
				SqlDataReader reader = cmd.ExecuteReader(CommandBehavior.CloseConnection);
				while (reader.Read())
				{
					o.OnNext(reader.GetString(0));
				}
				o.OnCompleted();
				return Disposable.Create(()=>Console.WriteLine("--Disposed--"));
			}
		});
	}
```
這段程式就像很多現存的回傳`IEnumerable<T>`的資料存取層的程式，然而在Rx中以非同步方式存取（使用[SubscribeOn and ObserveOn](15_SchedulingAndThreading.html#SubscribeOnObserveOn))會更為簡單。
這個資料存取層的程式是延遲估值的，而且沒有提供快取功能。這是典型的"Cold observables"，呼叫此函式不會做任何事。而對其回序列`IObservable<T>`的訂閱會喚起連線至資料庫的委託。

這裡有上述程式的消費者，但它只想要三個值（全部有128個），這段程式使用`Take(3)`表示消費者的限制，但`GetProducts()`函式仍然會產生_所有_的值。
```csharp
	public void ColdSample()
	{
		var products = GetProducts().Take(3);
		products.Subscribe(Console.WriteLine);
		Console.ReadLine();
	}
```
`GetProducts()`函式是一個很單純的範例，它少了讓使用者'cancel'的功能。

這表示會讀取所有的值，即使我們只想要三個 

後續章節[scheduling](15_SchedulingAndThreading.html)我們會介紹如何正確的提供取消功能。

##Hot observables					{#HotObservables}

上述範例中，我們不會和資料庫建立連線，直到`GetProducts()`的消費者訂閱了它的回傳值。
對`GetProducts()`循序或甚至平行呼叫會回傳各自獨立的可觀察序列，且各自擁有其對資料庫的操作。

定義上，一個"Hot observable"是一個即使沒有訂閱者也會產生推送值的可觀察序列。
"Hot observable"的典型範例是"UI Events"和"Subjects"。
舉例來說，如果我們移動滑鼠，`MouseMove`事件會被觸發，而如果此事件沒有被事件處理器訂閱，不會發生任何事。如果，換個方式，我們建立了一個`Subject<int>`，可以使用`OnNext`注入值，不管有沒有觀察者訂閱此主題。

一些"cold"的可觀察序列可以看起來像是"Hot"的。幾個令人訝異的範例是`Observable.Interval`和`Observable.Timer`（如果你讀過[Creating observable sequences](04_CreatingObservableSequences.html#Unfold)就不應該訝異）。

下列範例中，我們透過`Interval`對一個實體訂閱了兩次。
這兩個訂閱中的延遲會展示雖然對同一個可觀察實體訂閱，每個訂閱所接收到的值是獨立的，所以：`Interval`是"cold"。
```csharp
	public void SimpleColdSample()
	{
		var period = TimeSpan.FromSeconds(1);
		var observable = Observable.Interval(period);
		observable.Subscribe(i => Console.WriteLine("first subscription : {0}", i));
		Thread.Sleep(period);
		observable.Subscribe(i => Console.WriteLine("second subscription : {0}", i));
		Console.ReadKey();
		/* Output: 
		first subscription : 0 
		first subscription : 1 
		second subscription : 0 
		first subscription : 2 
		second subscription : 1 
		first subscription : 3 
		second subscription : 2 
		*/ 
	}
```
##Publish and Connect				{#PublishAndConnect}

如果我們想共享正確的值本身，而不只是同一個可觀察實體，可以用`Publish()`擴充函式。
它會回傳一個`IConnectableObservable<T>`型別，此型別依靠增加了一個`Connect()`函式的擴充自`IObservable<T>`型別來達成。
依靠使用`Publish()`及`Connect()`函式，我們可以達成共享的功能。
```csharp
	var period = TimeSpan.FromSeconds(1);
	var observable = Observable.Interval(period).Publish();
	observable.Connect();
	observable.Subscribe(i => Console.WriteLine("first subscription : {0}", i));
	Thread.Sleep(period);
	observable.Subscribe(i => Console.WriteLine("second subscription : {0}", i));
```
輸出：

<div class="output">
	<div class="line">first subscription : 0 </div>
	<div class="line">first subscription : 1 </div>
	<div class="line">second subscription : 1 </div>
	<div class="line">first subscription : 2 </div>
	<div class="line">second subscription : 2 </div>
</div>

上述範例中，`observable`變數是一個`IConnectableObservable<T>`型別，透過呼叫`Connect()`，它會訂閱裡面的（`Observable.Interval`）。
在這種情況下，如果我們夠快，可以在第一個元素產生前就訂閱，但只能在第一次訂閱。第二次的訂閱較慢，且錯過了第一個推送值，我們可以移動`Connect()`函式到所有的訂閱都完成後，這樣，即使已呼叫`Thread.Sleep`，我們不會實際訂閱底層，直到兩個訂閱完成後。如下所示：
```csharp
	var period = TimeSpan.FromSeconds(1);
	var observable = Observable.Interval(period).Publish();
	observable.Subscribe(i => Console.WriteLine("first subscription : {0}", i));
	Thread.Sleep(period);
	observable.Subscribe(i => Console.WriteLine("second subscription : {0}", i));
	observable.Connect();
```
<div class="output">
	<div class="line">first subscription : 0 </div>
	<div class="line">second subscription : 0 </div>
	<div class="line">first subscription : 1 </div>
	<div class="line">second subscription : 1 </div>
	<div class="line">first subscription : 2 </div>
	<div class="line">second subscription : 2 </div>
</div>

你可以想像，當一個應用程式需要共享序列資料時這會很有用處。
在一個金融交易應用軟體中，如果你想對一個特定的資產的資料串流消費，你會想重用此公共串流以避免對提供資料的伺服器進行另一個訂閱。
在社交應用軟體中，許多`widgets`可能需要在某個人上線時得到通知，`Publish`和`Connect`會是此狀況的最佳應用。

###Disposal of connections and subscriptions	{#Disposal}

另一個有趣的事是disposal是如何被執行的。
是的，我們還沒提到`Connect`回傳的是一個`IDisposable`。依靠對'connection'的disposing，你可以讓序列啟動和關閉(`Connect()`啟動，disposing關閉)。
這個範例中，我們可以看到序列可以多次被連線及斷線：
```csharp
	var period = TimeSpan.FromSeconds(1);
	var observable = Observable.Interval(period).Publish();
	observable.Subscribe(i => Console.WriteLine("subscription : {0}", i));
	var exit = false;
	while (!exit)
	{
		Console.WriteLine("Press enter to connect, esc to exit.");
		var key = Console.ReadKey(true);
		if(key.Key== ConsoleKey.Enter)
		{
			var connection = observable.Connect(); //--Connects here--
			Console.WriteLine("Press any key to dispose of connection.");
			Console.ReadKey();
			connection.Dispose(); //--Disconnects here--
		}
		if(key.Key==ConsoleKey.Escape)
		{
			exit = true;
		}
	}
```
輸出：

<div class="output">
	<div class="line">Press enter to connect, esc to exit. </div>
	<div class="line">Press any key to dispose of connection. </div>
	<div class="line">subscription : 0 </div>
	<div class="line">subscription : 1 </div>
	<div class="line">subscription : 2 </div>
	<div class="line">Press enter to connect, esc to exit. </div>
	<div class="line">Press any key to dispose of connection. </div>
	<div class="line">subscription : 0 </div>
	<div class="line">subscription : 1 </div>
	<div class="line">subscription : 2 </div>
	<div class="line">Press enter to connect, esc to exit. </div>
</div>

最後讓我們來看看自動對連線disposal。我們想讓同一個序列被多個訂閱者訂閱，如同上述的價格串流。如果有任何訂閱者，我們也會想讓序列表現'Hot'的行為，因此，不僅顯而易見，應當存在用於自動連線（一旦訂閱已經進行）的機制，而且還有用於從序列斷開連線（一旦沒有訂閱者）的機制。

首先讓我們看看當連線沒有訂閱者，然後再取消訂閱時，序列會發生什麼：
```csharp
	var period = TimeSpan.FromSeconds(1);
	var observable = Observable.Interval(period)
		.Do(l => Console.WriteLine("Publishing {0}", l)) //Side effect to show it is running
		.Publish();
	observable.Connect();
	Console.WriteLine("Press any key to subscribe");
	Console.ReadKey();
	var subscription = observable.Subscribe(i => Console.WriteLine("subscription : {0}", i));
	Console.WriteLine("Press any key to unsubscribe.");
	Console.ReadKey();
	subscription.Dispose();
	Console.WriteLine("Press any key to exit.");
	Console.ReadKey();
```
輸出：

<div class="output">
	<div class="line">Press any key to subscribe </div>
	<div class="line">Publishing 0 </div>
	<div class="line">Publishing 1 </div>
	<div class="line">Press any key to unsubscribe. </div>
	<div class="line">Publishing 2 </div>
	<div class="line">subscription : 2 </div>
	<div class="line">Publishing 3 </div>
	<div class="line">subscription : 3 </div>
	<div class="line">Press any key to exit. </div>
	<div class="line">Publishing 4 </div>
	<div class="line">Publishing 5 </div>
</div>

在這裡有幾件事情需要注意：

1. 我用`Do`擴充函式建立了序列的邊際效應(例如：輸出至console中)，這可讓我們知道序列何時建立連線。
2. 我們先連線再訂閱，表示我們可以在沒有訂閱時推送資料；即讓序列為"Hot"。
3. 我們取消了訂閱，但沒有dispose連線，這表示序列將會繼續動作。

###RefCount					{#RefCount}

讓我們用`RefCount`擴充函式替換掉上述範例中的`Connect()`函式，這會"神奇地"實現我們對自動disposal及延遲連線的需求。當自動實作我們想要的"connect"和"disconnect"行為時，`RefCount`會代入一個`IConnectableObservable<T>`並將其轉型至`IObservable<T>`。
```csharp
	var period = TimeSpan.FromSeconds(1);
	var observable = Observable.Interval(period)
		.Do(l => Console.WriteLine("Publishing {0}", l)) //side effect to show it is running
		.Publish()
		.RefCount();
	//observable.Connect(); Use RefCount instead now 
	Console.WriteLine("Press any key to subscribe");
	Console.ReadKey();
	var subscription = observable.Subscribe(i => Console.WriteLine("subscription : {0}", i));
	Console.WriteLine("Press any key to unsubscribe.");
	Console.ReadKey();
	subscription.Dispose();
	Console.WriteLine("Press any key to exit.");
	Console.ReadKey();
```
輸出：

<div class="output">
	<div class="line">Press any key to subscribe </div>
	<div class="line">Press any key to unsubscribe. </div>
	<div class="line">Publishing 0 </div>
	<div class="line">subscription : 0 </div>
	<div class="line">Publishing 1 </div>
	<div class="line">subscription : 1 </div>
	<div class="line">Publishing 2 </div>
	<div class="line">subscription : 2 </div>
	<div class="line">Press any key to exit. </div>
</div>

`Publish`/`RefCount`代入一個“cold”的可觀察序列並將其用“hot”可觀察序列的方式共享對後續的觀察者超級有用，而`RefCount()`函式也讓我們避免了race condition。在上述範例中，我們在連線建立前訂閱了序列，這並不總是可行，特別是我們從一個函式中取得了序列。依靠使用`RefCount`函式，由於其自動連線的行為可以減輕subscribe/connect的race condition。在上述範例中，我們在連線建立前訂閱了序列，這並不總是可行，特別是我們從一個函式中取得了序列。依靠使用`RefCount`函式，由於其自動連線的行為可以減輕subscribe/connect的race condition。

##Other connectable observables			{#OtherConnectables}

`Connect`函式不是回傳`IConnectableObservable <T>`實體的唯一函式。
連接或延遲一個操作的功能的能力在其他領域也很有用。

###PublishLast 						{#PublishLast}

`PublishLast()`函式實際上是一個非阻塞的`Last()`呼叫。
你可以把它當成一個用`AsyncSubject <T>`包裝的目標序列。
你得到一個語義相同的`AsyncSubject <T>`，其中只有最後一個值被發布，且僅在序列完成後。
```csharp
	var period = TimeSpan.FromSeconds(1);
	var observable = Observable.Interval(period)
		.Take(5)
		.Do(l => Console.WriteLine("Publishing {0}", l)) //side effect to show it is running
		.PublishLast();
	observable.Connect();
	Console.WriteLine("Press any key to subscribe");
	Console.ReadKey();
	var subscription = observable.Subscribe(i => Console.WriteLine("subscription : {0}", i));
	Console.WriteLine("Press any key to unsubscribe.");
	Console.ReadKey();
	subscription.Dispose();
	Console.WriteLine("Press any key to exit.");
	Console.ReadKey();
```
輸出：

<div class="output">
	<div class="line">Press any key to subscribe </div>
	<div class="line">Publishing 0 </div>
	<div class="line">Publishing 1 </div>
	<div class="line">Press any key to unsubscribe. </div>
	<div class="line">Publishing 2 </div>
	<div class="line">Publishing 3 </div>
	<div class="line">Publishing 4 </div>
	<div class="line">subscription : 4 </div>
	<div class="line">Press any key to exit. </div>
</div>

###Replay							{#Replay}

`Replay`擴充函式讓你可以如同一個`ReplaySubject<T>`一樣的對一個已存在的可觀察序列做類似'replay'語義的操作。
做一個提醒，`ReplaySubject<T>`會快取所有值，所以後續的訂閱者也可以取得所有數值。
這個範例中，兩個訂閱者同時被建立，第三個訂閱者在序列結束後被建立，即使如此，第三個訂閱者在底層序列完成後才被建立，我們仍可以取得所有的數值：
```csharp 
	var period = TimeSpan.FromSeconds(1);
	var hot = Observable.Interval(period)
		.Take(3)
		.Publish();
	hot.Connect();
	Thread.Sleep(period); //Run hot and ensure a value is lost.
	var observable = hot.Replay();
	observable.Connect();
	observable.Subscribe(i => Console.WriteLine("first subscription : {0}", i));
	Thread.Sleep(period);
	observable.Subscribe(i => Console.WriteLine("second subscription : {0}", i));
	Console.ReadKey();
	observable.Subscribe(i => Console.WriteLine("third subscription : {0}", i));
	Console.ReadKey();
```
輸出：

<div class="output">
	<div class="line">first subscription : 1 </div>
	<div class="line">second subscription : 1 </div>
	<div class="line">first subscription : 2 </div>
	<div class="line">second subscription : 2 </div>
	<div class="line">third subscription : 1 </div>
	<div class="line">third subscription : 2 </div>
</div>

`Replay`擴充函式有數個對應至`ReplaySubject<T>`建構式的覆載；你可以用數值或時間指定緩衝區大小。

###Multicast			{#Multicast}

`PublishLast`和`Replay`函式實際上將`AsyncSubject <T>`和`ReplaySubject <T>`功能應用於內部的可觀察序列。
我們可以嘗試自己建立一個概略的實做：
```csharp
	var period = TimeSpan.FromSeconds(1);
	//var observable = Observable.Interval(period).Publish();
	var observable = Observable.Interval(period);
	var shared = new Subject<long>();
	shared.Subscribe(i => Console.WriteLine("first subscription : {0}", i));
	observable.Subscribe(shared);   //'Connect' the observable.
	Thread.Sleep(period);
	Thread.Sleep(period);
	shared.Subscribe(i => Console.WriteLine("second subscription : {0}", i));
```
輸出：

<div class="output">
	<div class="line">first subscription : 0</div>
	<div class="line">first subscription : 1</div>
	<div class="line">second subscription : 1</div>
	<div class="line">first subscription : 2</div>
	<div class="line">second subscription : 2 </div>
</div>

x函式庫為我們提供了一個很好的方法來達成這一點。你可以通過`Multicast`擴充函式來應用至subject behavior，這讓你可以用一個特定的主題的行為來共享或"multicast"一個可觀察序列。
舉例來說：

 * `.Publish()` = `.Multicast(new Subject<T>)`
 * `.PublishLast()` = `.Multicast(new AsyncSubject<T>)`
 * `.Replay()` = `.Multicast(new ReplaySubject<T>)`

熱和冷觀察者是兩種不同的共享可觀察序列的風格。
兩者都有同樣是有效的應用，但以不同的方式來表現。
冷的可觀察序列允許你為每個訂閱者獨立地對可觀察序列使用延遲估值。
而熱的功能允許你通過multicast你的序列來共享通知，即使在沒有訂閱者的狀況。
The use of `RefCount` allows you to have lazily-evaluated, multicast observable sequences, coupled with eager disposal semantics once the last subscription is disposed.

<!--
	<a name="Defer"></a>
	<h2>Defer
	<p></p>

	<a name="Synchronize"></a>
	<h2>Synchronize
	<p></p>
	-->

---
	
<div class="webonly">
	<h1 class="ignoreToc">Additional recommended reading</h1>
	<div align="center">
		<div style="display:inline-block; vertical-align: top;  margin: 10px; width: 140px; font-size: 11px; text-align: center">
			<!--C# in a nutshell Amazon.co.uk-->
			<iframe src="http://rcm-uk.amazon.co.uk/e/cm?t=int0b-21&amp;o=2&amp;p=8&amp;l=as1&amp;asins=B008E6I1K8&amp;ref=qf_sp_asin_til&amp;fc1=000000&amp;IS2=1&amp;lt1=_blank&amp;m=amazon&amp;lc1=0000FF&amp;bc1=000000&amp;bg1=FFFFFF&amp;f=ifr" 
					style="width:120px;height:240px;margin: 10px" 
					scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>

		</div>
		<div style="display:inline-block; vertical-align: top;  margin: 10px; width: 140px; font-size: 11px; text-align: center">
			<!--C# Linq pocket reference Amazon.co.uk-->
			<iframe src="http://rcm-uk.amazon.co.uk/e/cm?t=int0b-21&amp;o=2&amp;p=8&amp;l=as1&amp;asins=0596519249&amp;ref=qf_sp_asin_til&amp;fc1=000000&amp;IS2=1&amp;lt1=_blank&amp;m=amazon&amp;lc1=0000FF&amp;bc1=000000&amp;bg1=FFFFFF&amp;f=ifr" 
					style="width:120px;height:240px;margin: 10px" 
					scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
		</div>

		<div style="display:inline-block; vertical-align: top; margin: 10px; width: 140px; font-size: 11px; text-align: center">
			<!--CLR via C# v4 Amazon.co.uk-->
			<iframe src="http://rcm-uk.amazon.co.uk/e/cm?t=int0b-21&amp;o=2&amp;p=8&amp;l=as1&amp;asins=B00AA36R4U&amp;ref=qf_sp_asin_til&amp;fc1=000000&amp;IS2=1&amp;lt1=_blank&amp;m=amazon&amp;lc1=0000FF&amp;bc1=000000&amp;bg1=FFFFFF&amp;f=ifr" 
					style="width:120px;height:240px;margin: 10px" 
					scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>

		</div>
		<div style="display:inline-block; vertical-align: top; margin: 10px; width: 140px; font-size: 11px; text-align: center">
			<!--Real-world functional programming Amazon.co.uk-->
			<iframe src="http://rcm-uk.amazon.co.uk/e/cm?t=int0b-21&amp;o=2&amp;p=8&amp;l=as1&amp;asins=1933988924&amp;ref=qf_sp_asin_til&amp;fc1=000000&amp;IS2=1&amp;lt1=_blank&amp;m=amazon&amp;lc1=0000FF&amp;bc1=000000&amp;bg1=FFFFFF&amp;f=ifr" 
					style="width:120px;height:240px;margin: 10px" 
					scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>

		</div>           
	</div>
</div>