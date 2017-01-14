---
tags: Rx, C#
title: Rx介紹 Part 2 - Sequence basics(譯ing...)
---
>嗯，正在學Rx，學了一輪後實際應用時發現還有不懂的地方，再重讀一次，順便簡單的翻譯下…*翻譯不出來*的或是*覺得不重要*的就以"…"符號替換，或顯示原文。
當然，辭不達義的地方也會有，請包含…
## [Part 2 - Sequence basics](http://www.introtorx.com/Content/v1.0.10621.0/04_CreatingObservableSequences.html#CreationOfObservables)
你想開始寫些Rx程式了，不過要從那裡開始呢？我們已經瞭解了關鍵型別，而且知道不應該實作自己的`IObserver<T>`和`IObservable<T>`，儘量用工廠方法而不是subjects。假設我們已經有了一個observable序列，要如何中從取得想要的資料呢？基本上我們需要瞭解建立observable序列、賦值並取得值的方式。

本章我們將會發現建立及查詢observable序列的基本方式，我們宣稱LINQ是使用及瞭解Rx的基礎，進一步說，我們發現函數式編程是深入LINQ的路徑並能帶你精通Rx。為達成目標，我們將查詢運算子分為三個群組，每個群組會有一個主運算子，其餘運算子可基於其擴展功能。這個解構的方式不僅能帶你更深入Rx、函數式編程及查詢的合成；它應能在你覺得不夠用時建立自訂的運算子。

##建立序列
上一章我們使用了第一個Rx的擴充方法，Subscribe和它的其它覆載。我們也看到了我們的第一個工廠方法，Subject.Create()，我們將會看到更多的擴充了`IObservable<T>`的方法，它讓Rx成為了Rx。也許你會訝異於Rx函式庫中僅有幾個public instance方法，但其實它有很多公開的靜態方法，或者說是擴充方法，因為這大量的擴充方法，所以我們把它們分門別類介紹。

也許有的讀者會覺得他們可以跳過接下來這幾章，但我建議除非你對LINQ及函數合成很熟悉才跳過。這本書的目的是在提供你一步一步的了解Rx，讓身為讀者的你能在軟體中使用Rx。完整的瞭解Rx是你使用它的基礎，最常見的錯誤發生在人們誤解了Rx建構的原則，所以我建議你依續閱讀。

從subjects建構的方式介紹起應該很合理。我們第一個分類就是建構函式：建立`IObservable<t>`的方式，這些方式一般採用一個seed來產生序列，不管是單一型別的值，或者是型別本身，在函數式編程中，這可以稱呼為"anamorphism"或是分類為"unfold"。

##Simple factory methods
###Observable.Return
我們的第一個也是最基礎的範例中要介紹的是`Observable.Return<T>(T value)`函式，這個方法接受一個型別`T`的值，並回傳一個型別為`IObservable<T>`的值並結束，它unfolded一個`T`型別的值到一個observable的序列。
```csharp
var singleValue = Observable.Return<string>("Value");
//which could have also been simulated with a replay subject
var subject = new ReplaySubject<string>();
subject.OnNext("Value");
subject.OnCompleted();
```
注意上面這個範例中我們可以使用工廠方法，也可以使用replay subject的方式，它的區別在於工廠方法僅用一行且使用宣告式的方式而不是命令式的編程方法，上述範例中我們說明參數的型態是字型，這可省略因編譯器會自行推論出。
```csharp
singleValue = Observable.Return<string>("Value");
//Can be reduced to the following
singleValue = Observable.Return("Value");
```
###Observable.Empty
接下來這兩個方法只需要型別參數以加入至observable序列中，第一個是`Observable.Empty<T>()`，這個方法回傳一個空的`IObservable<T>`序列，它只推送一個`OnCompleted`通知。
```csharp
var empty = Observable.Empty<string>();
//Behaviorally equivalent to
var subject = new ReplaySubject<string>();
subject.OnCompleted();
```
###Observable.Never
`Observable.Never<T>()`方法回傳一個無窮序列但沒有任何通知。
```csharp
var never = Observable.Never<string>();
//similar to a subject without notifications
var subject = new Subject<string>();
```
###Observable.Throw
`Observable.Throw<T>(Exception)`方法需要一個型別參數，及一個`Exception`以讓`OnError`使用，這個方法建立只有一個包含傳入的例外的`OnError`通知。
```csharp
var throws = Observable.Throw<string>(new Exception()); 
//Behaviorally equivalent to
var subject = new ReplaySubject<string>(); 
subject.OnError(new Exception());
```
###Observable.Create
這個`Create`方法跟上述方法有點不一樣，它的signature也許第一眼看到時會覺得很複雜，一旦熟悉了就會習慣。
```csharp
//Creates an observable sequence from a specified Subscribe method implementation.
public static IObservable<TSource> Create<TSource>(
Func<IObserver<TSource>, IDisposable> subscribe)
{...}
public static IObservable<TSource> Create<TSource>(
Func<IObserver<TSource>, Action> subscribe)
{...}
```
這個方法允許你指定一個會在被訂閱時執行的委託，訂閱時的`IObserver<T>`會被傳至委託中，所以你可以在你需要時呼叫`OnNext`/`OnError`/`OnCompleted`方法，這是很少的你需要注意`IObserver<T>`介面的時候，你的委託會是回傳`IDisposable`的一個Func，這個`IDisposable`的`Dispose()`方法會在取消訂閱後被呼叫。

The `Create` factory method is the preferred way to implement custom observable sequences. The usage of subjects should largely remain in the realms of samples and testing. Subjects are a great way to get started with Rx. They reduce the learning curve for new developers, however they pose several concerns that the Create method eliminates. Rx is effectively a functional programming paradigm. Using subjects means we are now managing state, which is potentially mutating. Mutating state and asynchronous programming are very hard to get right. Furthermore many of the operators (extension methods) have been carefully written to ensure correct and consistent lifetime of subscriptions and sequences are maintained. When you introduce subjects you can break this. Future releases may also see significant performance degradation if you explicitly use subjects.
Create工廠方法應是你實作自訂的`IObservable`序列的優先選擇。

The Create method is also preferred over creating custom types that implement the IObservable interface. There really is no need to implement the observer/observable interfaces yourself. Rx tackles the intricacies that you may not think of such as thread safety of notifications and subscriptions.

A significant benefit that the Create method has over subjects is that the sequence will be lazily evaluated. Lazy evaluation is a very important part of Rx. It opens doors to other powerful features such as scheduling and combination of sequences that we will see later. The delegate will only be invoked when a subscription is made.

這個範例中，我們先展示一個blocking及早求值式的方法，然後展示一個使用延遲取值且非blocking式的建立observable序列的方式。
```csharp
private IObservable<string> BlockingMethod()
{
	var subject = new ReplaySubject<string>();
	subject.OnNext("a");
	subject.OnNext("b");
	subject.OnCompleted();
	Thread.Sleep(1000);
	return subject;
}
private IObservable<string> NonBlocking()
{
	return Observable.Create<string>(
		(IObserver<string> observer) =>
		{
			observer.OnNext("a");
			observer.OnNext("b");
			observer.OnCompleted();
			Thread.Sleep(1000);
			return Disposable.Create(() => Console.WriteLine("Observer has unsubscribed"));
			//or can return an Action like 
			//return () => Console.WriteLine("Observer has unsubscribed"); 
		});
}
```
這範例看上去很奇怪，它旨在讓你知道當我們呼叫及早求值、阻塞式的方法，在你取得`IObservable<string>`結果前會被阻塞至少1秒，不管是已經訂閱或是還沒，而非阻塞式方法是延遲求值式的，所以我們會馬上取得`IObservable<string>`且只在訂閱時受執行緒睡眠影響。

一個練習，試著使用`Create`建立`Empty`、`Return`、`Never`及`Throw`擴充方法。如果你使用Visual Studio或是[LINQPad](http://www.linqpad.net/)，寫看看，如果你沒有(或者你正在去公司的電車上)，試著想看看要如何實作，你完成後再往下看我們如何實作它們...

<hr style="page-break-after: always" />

使用`Observable.Create`建立`Empty`、`Return`、`Never`及`Throw`方法：
```csharp
public static IObservable<T> Empty<T>()
{
	return Observable.Create<T>(o =>
	{
		o.OnCompleted();
		return Disposable.Empty;
	});
}
public static IObservable<T> Return<T>(T value)
{
	return Observable.Create<T>(o =>
	{
		o.OnNext(value);
		o.OnCompleted();
		return Disposable.Empty;
	});
}
public static IObservable<T> Never<T>()
{
	return Observable.Create<T>(o =>
	{
		return Disposable.Empty;
	});
}
public static IObservable<T> Throws<T>(Exception exception)
{
	return Observable.Create<T>(o =>
	{
		o.OnError(exception);
		return Disposable.Empty;
	});
}
```
你可以看到`Observable.Create`方法給了我們建立自己的工廠方法的能力。你也許注意到一旦我們產生我們的OnNext通知後，我們就可以取得被回傳的訂閱的token(`IDisposable`的實作)，這是因為我們提供的委託中的值都是序列式的，但它也讓這個token顯得沒什麼作用。現在讓我們來看看讓這些回傳值較有用的方式，先看我們在委託中建立一個Timer，它會在每個tick時呼叫`OnNext`方法。
```csharp
//Example code only
public void NonBlocking_event_driven()
{
	var ob = Observable.Create<string>(
	observer =>
	{
		var timer = new System.Timers.Timer();
		timer.Interval = 1000;
		timer.Elapsed += (s, e) => observer.OnNext("tick");
		timer.Elapsed += OnTimerElapsed;
		timer.Start();
		return Disposable.Empty;
	});
	var subscription = ob.Subscribe(Console.WriteLine);
	Console.ReadLine();
	subscription.Dispose();
}
private void OnTimerElapsed(object sender, ElapsedEventArgs e)
{
	Console.WriteLine(e.SignalTime);
}
```
輸出：
```bash
tick
01/01/2012 12:00:00
tick
01/01/2012 12:00:01
tick
01/01/2012 12:00:02
01/01/2012 12:00:03
01/01/2012 12:00:04
01/01/2012 12:00:05
```
上面這個範例是有錯的。當我們取消訂閱時，我們會停止看到"tick"的顯示；但我們並沒有釋放我們的第二個event handler `OnTimerElasped`，而且timer也沒有被disposed掉，所以我們會繼續看到`ElapsedEventArgs.SignalTime`被顯示在畫面上，雖然我們已經把它dispose掉了，最簡單的修正方式是把`timer`當作`IDisposable`回傳的代表。
```csharp
//Example code only
var ob = Observable.Create<string>(
observer =>
{
	var timer = new System.Timers.Timer();
	timer.Interval = 1000;
	timer.Elapsed += (s, e) => observer.OnNext("tick");
	timer.Elapsed += OnTimerElapsed;
	timer.Start();
	return timer;
});
```
現在當我們取消訂閱後，`Timer`也會disposed。

`Observable.Create`方法有一個需要你的Func回傳一個Action而不是`IDisposable`的覆載，跟上個範例很像，我們使用一個action去取消註冊事件處理器，避免因持有timer的參考導致的memory leak。
```csharp
//Example code only
var ob = Observable.Create<string>(
	observer =>
	{
		var timer = new System.Timers.Timer();
		timer.Enabled = true;
		timer.Interval = 100;
		timer.Elapsed += OnTimerElapsed;
		timer.Start();
		return ()=>{
			timer.Elapsed -= OnTimerElapsed;
			timer.Dispose();
		};
	});
```
上面這幾個範例教你如何使用`Observable.Create`方法，不過這些只是範例，實際上有更好的方式從一個timer中產生值，後續你將會看到這方式。上面只是讓你知道使用`Observable.Create`並採延遲取得的方式產生observable序列，我們將在後續章節更深入延遲取值及`Create`工廠方法的應用，特別是當我們談到同步及排程時。

##Functional unfolds
As a functional programmer you would come to expect the ability to unfold a potentially infinite sequence. An issue we may face with Observable.Create is that it can be a clumsy way to produce an infinite sequence. Our timer example above is an example of an infinite sequence, and while this is a simple implementation it is an annoying amount of code for something that effectively is delegating all the work to the System.Timers.Timer class. The Observable.Create method also has poor support for unfolding sequences using corecursion.

###Corecursion
Corecursion is a function to apply to the current state to produce the next state. Using corecursion by taking a value, applying a function to it that extends that value and repeating we can create a sequence. A simple example might be to take the value 1 as the seed and a function that increments the given value by one. This could be used to create sequence of [1,2,3,4,5...].

Corecursion可透過簡單的`yield return`語法來建立一個`IEnumerable<int>`的序列。
```csharp
private static IEnumerable<T> Unfold<T>(T seed, Func<T, T> accumulator)
{
	var nextValue = seed;
	while (true)
	{
		yield return nextValue;
		nextValue = accumulator(nextValue);
	}
}
```
上述範例可用以產生如下的自然數序列。
```csharp
var naturalNumbers = Unfold(1, i => i + 1);
Console.WriteLine("1st 10 Natural numbers");
foreach (var naturalNumber in naturalNumbers.Take(10))
{
	Console.WriteLine(naturalNumber);
}
```
輸出：
```bash
1st 10 Natural numbers
1
2
3
4
5
6
7
8
9
10
```
記住`Take(10)`是用來中斷這個無窮數列。

無窮或任意長度的數列很有用，再來讓我們先看一些Rx提供的，然後再看一般的建立無窮observable數列。

###Observable.Range
`Observable.Range(int, int)`回傳一串指定範圍的整數，第一個參數是起始值，第二個是個數，下列範例會顯示從10至24的數字並結束。
```csharp
var range = Observable.Range(10, 15);
range.Subscribe(Console.WriteLine, ()=>Console.WriteLine("Completed"));
```
###Observable.Generate
使用`Observable.Create`去模擬建立可指定範圍的工廠方法很難，在於遵守延遲取值及資源要能被dispose的原則，我們可以在這裡使用corecursion方式去提供較多的unfold方法，在Rx中就是使用`Observable.Generate`。

`Observable.Generate`較簡單的版本需要下列參數：

* 一個初始狀態
* 一個定義序列結束的predicate
* 可依當前狀態產生下一狀態的一個函式
* 可轉換當前狀態至輸出的函式
```csharp
public static IObservable<TResult> Generate<TState, TResult>(
	TState initialState, 
	Func<TState, bool> condition, 
	Func<TState, TState> iterate, 
	Func<TState, TResult> resultSelector)
```
寫一個你自己的使用`Observbale.Generate`的`Range`工廠方法當做練習。

Consider the Range signature Range(int start, int count), which provides the seed and a value for the conditional predicate. You know how each new value is derived from the previous one; this becomes your iterate function. Finally, you probably don't need to transform the state so this makes the result selector function very simple.

當你完成自己的版本後再繼續…

下列使用`Observable.Generate`建立`Range`工廠方法的一種方式。
```csharp
//Example code only
public static IObservable<int> Range(int start, int count)
{
	var max = start + count;
	return Observable.Generate(
		start, 
		value => value < max, 
		value => value + 1, 
		value => value);
}
```
###Observable.Interval
Earlier in the chapter we used a `System.Timers.Timer` in our observable to generate a continuous sequence of notifications. As mentioned in the example at the time, this is not the preferred way of working with timers in Rx. As Rx provides operators that give us this functionality it could be argued that to not use them is to re-invent the wheel. More importantly the Rx operators are the preferred way of working with timers due to their ability to substitute in schedulers which is desirable for easy substitution of the underlying timer. There are at least three various timers you could choose from for the example above:
本章前面我們在observable中使用System.Timers.Timer以產生一個連續數列的通知，如同那時提到的，在Rx中不適合用timer，因為Rx已提供，所以不要再自己重新發明輪子。…

* `System.Timers.Timer`
 * `System.Threading.Timer`
 * `System.Windows.Threading.DispatcherTimer`

使用schedulemr將timer抽象化，讓我們可以將同樣的程式碼用在不同的平台上，更重要的是可以寫出不依賴平台的可測試的程式碼，Schedulers較本章的範例更複雜，後續Scheduling and threading章節會繼續談論。

有三種較好的以常數時間工作的方式，每一種都是更一般化的型式，第一種`Observable.Interval(TimeSpan)`會遞增從零開始的數，以你選擇的頻率，下列範例每250ms產生值。
```csharp
var interval = Observable.Interval(TimeSpan.FromMilliseconds(250));
interval.Subscribe(
	Console.WriteLine, 
	() => Console.WriteLine("completed"));
```
輸出：
```bash
0
1
2
3
4
5
```
一旦你訂閱後，你必須自己取消訂閱以停止序列，這是一個無窮數列的示範。

###Observable.Timer
第二種以常數時間產生的數列是`Observable.Timer`，它有很多個覆載；我們將看到的第一種非常簡單，這個方法只需要一個`TimeSpan`當作區間參數，但這個方法在時間到達後只產生一個值，然後就結束。
```csharp
var timer = Observable.Timer(TimeSpan.FromSeconds(1));
timer.Subscribe(
	Console.WriteLine, 
	() => Console.WriteLine("completed"));
```
輸出：
```bash
0
completed
```
另外，你可以提供一個`DateTimeOffset`當做`dueTime`的參數，這個方法會產生0值並在dueTime後結束。

更進一步的覆載增加了一個`TimeSpan`代表要產生子序列值的時間間隔，這讓我們可以產生無窮序列且以`Observable.Timer`建立了`Observable.Interval`。
```csharp
public static IObservable<long> Interval(TimeSpan period)
{
	return Observable.Timer(period, period);
}
```
注意它現在回傳一個`long`的`IObservable`而不是`int`。原先`Observable.Interval`會等待一段時間後再產生第一個值，而`Observable.Timer`覆載會在你選擇的時間開始序列，用`Observable.Timer`你可以如下所示的馬上開始一個內部的序列。

###Observable.Timer(TimeSpan.Zero, period)
這是我們第三個也是最常用的產生時間相關序列的方法。現在我們來看一個更複雜的覆載，它讓你提供一個可指定週期來產生值的函式。
```csharp
public static IObservable<TResult> Generate<TState, TResult>(
	TState initialState, 
	Func<TState, bool> condition, 
	Func<TState, TState> iterate, 
	Func<TState, TResult> resultSelector, 
	Func<TState, TimeSpan> timeSelector)
```
使用這個複載，並指定額外的`timeSelector`參數，我們可以產生自己的`Observable.Timer`及`Observable.Interval`實作函式。
```csharp
public static IObservable<long> Timer(TimeSpan dueTime)
{
	return Observable.Generate(
		0l,
		i => i < 1,
		i => i + 1,
		i => i,
		i => dueTime);
}
public static IObservable<long> Timer(TimeSpan dueTime, TimeSpan period)
{
	return Observable.Generate(
		0l,
		i => true,
		i => i + 1,
		i => i,
		i => i == 0 ? dueTime : period);
}
public static IObservable<long> Interval(TimeSpan period)
{
	return Observable.Generate(
		0l,
		i => true,
		i => i + 1,
		i => i,
		i => period);
}
```
這代表你可以使用`Observable.Generate`來產生無窮序列。我會把這個當成你的習題和練習。我也發現這些方法不僅可在日常工作中使用，當你需要產生dummy資料時更有用處。

##轉換成`IObservable<T>`
產生一個observablable序列討論到函數式編程中較複雜的方面，如corecursion及unfold，你也可以將一個已存在的同步或非同步範式轉成Rx範式來當做起始一個序列的方式。

###From delegates
`Observable.Start`方法將一個費時的`Func<T>`或`Action`轉換成單一值的observable序列，系統預設此動作將會在一個ThreadPool中以非同步的方式來執行。如果你使用的覆載是一個`Func<T>`它的回傳值會是`IObservable<T>`，當函式回傳值後，此值會被推送然後結束此序列。如果你使用的覆載是一個`Action`，回傳的序列會是`IObservable<Unit>`。型別`Unit`是一種函數式編程的架構，等同於void。這個狀況下`Unit`被用來推送一個`Action`已完成的通知，然而此序列會在`Unit`被推送後馬上結束。型別`Unit`本身並沒有值；它只是一個空的荷載，只是作為`OnNext`推送。下列示範這兩種覆載的使用：
```csharp
static void StartAction()
{
	var start = Observable.Start(() =>
		{
			Console.Write("Working away");
			for (int i = 0; i < 10; i++)
			{
				Thread.Sleep(100);
				Console.Write(".");
			}
		});
	start.Subscribe(
		unit => Console.WriteLine("Unit published"), 
		() => Console.WriteLine("Action completed"));
}

static void StartFunc()
{
	var start = Observable.Start(() =>
	{
		Console.Write("Working away");
		for (int i = 0; i < 10; i++)
		{
			Thread.Sleep(100);
			Console.Write(".");
		}
		return "Published value";
	});
	start.Subscribe(
		Console.WriteLine, 
		() => Console.WriteLine("Action completed"));
}
```
注意Observable.Start和Observable.Return這兩個方法的不同之處；Start延遲取值，而Return及早求值，這讓Start方法像是一個Task，但也可能造成使用上的疑惑，兩種都是合適的工具，但選用時取決於你的問題。Tasks適合平行計算，也可靠continuations提供工作流程以用在大量計算的工作。Tasks也能從文件化及強制單一值得到好處(？)，而Start在整合已大量使用observable序列的程式碼中的重計算的工作也較適合，在後續章節我們會看到較進階的組合序列的方式。

###From events
如同我們稍早提到的，.NET已經提供了一個Reactive、事件導向式程式設計的event model，當然Rx在這方面更適合，但由於較晚推出，所以它整合進了現有的event model。Rx提供各式方法可將事件轉成可觀察序列，你可以使用不同的方式來建立，下列是一般事件的使用：
```csharp
//Activated delegate is EventHandler
var appActivated = Observable.FromEventPattern(
	h => Application.Current.Activated += h,
	h => Application.Current.Activated -= h);
//PropertyChanged is PropertyChangedEventHandler
var propChanged = Observable.FromEventPattern
	<PropertyChangedEventHandler, PropertyChangedEventArgs>(
		handler => handler.Invoke,
		h => this.PropertyChanged += h,
		h => this.PropertyChanged -= h);
//FirstChanceException is EventHandler<FirstChanceExceptionEventArgs>
var firstChanceException = Observable.FromEventPattern<FirstChanceExceptionEventArgs>(
	h => AppDomain.CurrentDomain.FirstChanceException += h,
	h => AppDomain.CurrentDomain.FirstChanceException -= h);
```
這些覆載可能會讓你困惑，所以關鍵是要找到事件的簽名，如果簽名只是EventHandler委託，你可以使用第一個範例，如果委託是EventHandler的子類別，那要用第二個，且要提供特定的EventArgs，如果委託是泛型參數，那就用第三種，同時並給定泛型型別和事件參數。

一般來說將屬性變更轉成可觀察序列是大家都想做的。這些事件現在可透過INotifyPropertyChanged介面、DependencyProperty或靠著事件名稱對應到的屬性，如果你想自己包裝類似的行為，我強列的建議你先參考在[http://Rxx.codeplex.com](http://Rxx.codeplex.com)的Rxx函式庫，很多需求已經被很優雅的實現。

###From Task
Rx提供一組很有用且命名良好的覆載函式以讓現存的範式轉成可觀察範式。ToObservable()方法覆載提供一個簡單的路徑去實現轉換。

如前所述，`AsyncSubject<T>`很類似`Task<T>`，兩者皆從一個非同步的來源回傳給你單一值，也都快取此值供任何重覆或較慢的請求。我們要看的第一個ToObservable()覆載是至`Task<T>`，這個實做很簡單；
* 如果這個task的狀態是RanToCompletion，此值會加至序列中並結束此序列
* 如果這個task的狀態是Cancelled，此序列會發出TaskCanceledException的錯誤
* 如果這個task的狀態是Faulted，此序列會發出它內部Exception的錯誤
* 如果這個task尚未執行完成，此訂閱會被加入至continuation並繼續執行

有兩個使用這個擴充方法的理由：
1. 從Framework 4.5開始，幾乎所有的I/O-bound類的函式都回傳`Task<T>`
2. 如果適用`Task<T>`，那就用它，不要硬轉成`IObservable<T>`–因為它在型別系統中使用單一值通訊。換句話說，一個在未來某個時刻會回傳一個值的狀態下應回傳一個`Task<T>`，而不是`IObservable<T>`，然後你如果需要將它和其它的可觀察序列組合，那就使用ToObservable()函式。

這擴充方法的使用也很簡單：
```csharp
var t = Task.Factory.StartNew(()=>"Test");
var source = t.ToObservable();
source.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("completed"));
```
Output:
```bash
Test
completed
```
另有一個將Task(非泛型)轉成一個`IObservable<Unit>`的覆載。

###From `IEnumerable<T>`
最後一個ToObservable的覆載需要一個`IEnumerable<T>`參數，它在語義上向是一個Observable.Create的幫忙函式，裡面並有一個foreach迴圈可讓你走訪序列。
```csharp
//Example code only
public static IObservable<T> ToObservable<T>(this IEnumerable<T> source)
{
	return Observable.Create<T>(o =>
	{
		foreach (var item in source)
		{
			o.OnNext(item);
		}
		//Incorrect disposal pattern
		return Disposable.Empty;
	});
}
```
然而這初步的實作是有點單純，它無法明確的被disposal，處理例外的方法也不像後續章節說明的那樣正確，它的同步模型不對味。但是你不用過度擔心，因為這一版的Rx已經處理好了。

當從`IEnumerable<T>` 轉型至 `IObservable<T>`時，你應謹甚的考慮你要做的是什麼，也要小心的量測你的決定的效能。考慮到`IEnumerable<T>`天然的阻塞式同步拉的模型，並不一定適用在`IObservable<T>`非同步推的模型。記且你傳入IEnumerable、	IEnumerable<T>`、陣列或集合等資料型別以產生可觀察序列是完全合法的，如果這個序列可以一入性的被materialized，你應該想避免將它當成一個IEnumerable，若是合適的話，你應該傳入一個不可變型別，像是一個陣列或`ReadOnlyCollection<T>` ，後續我們將看到使用`IObservable<IList<T>>`當作運算子以提供一個批次的資料。

###From APM
最後我們來看一組將APM轉成可觀察序列的覆載。在.NET中的這種是一Begin及End開頭並傳入IAsyncResult型別的參數方法，常被用在I/O相關的API中。
```csharp
class WebRequest
{    
	public WebResponse GetResponse() 
	{...}
	public IAsyncResult BeginGetResponse(
		AsyncCallback callback, 
		object state) 
	{...}
	public WebResponse EndGetResponse(IAsyncResult asyncResult) 
	{...}
	...
}
class Stream
{
	public int Read(
		byte[] buffer, 
		int offset, 
		int count) 
	{...}
	public IAsyncResult BeginRead(
		byte[] buffer, 
		int offset, 
		int count, 
		AsyncCallback callback, 
		object state) 
	{...}
	public int EndRead(IAsyncResult asyncResult) 
	{...}
	...
}
```
寫這本書時.NET 4.5仍然在preview release，到.NET 4.5時，APM模型會被Task和新的async及await關鍵字取代，Rx 2.0也仍在beta release，且它將會整合新的特性，但這就不再本書的討論範圍了。

APM, or the Async Pattern, has enabled a very powerful, yet clumsy way of for .NET programs to perform long running I/O bound work. If we were to use the synchronous access to IO, e.g. WebRequest.GetResponse() or Stream.Read(...), we would be blocking a thread but not performing any work while we waited for the IO. This can be quite wasteful on busy servers performing a lot of concurrent work to hold a thread idle while waiting for I/O to complete. Depending on the implementation, APM can work at the hardware device driver layer and not require any threads while blocking. Information on how to follow the APM model is scarce. Of the documentation you can find it is pretty shaky, however, for more information on APM, see Jeffery Richter's brilliant book CLR via C# or Joe Duffy's comprehensive Concurrent Programming on Windows. Most stuff on the internet is blatant plagiary of Richter's examples from his book. An in-depth examination of APM is outside of the scope of this book.

我們可以用Observable.FromAsyncPattern方法來善用APM並避免它其怪的API，Jeffery van Gogh gives a brilliant walk through of the Observable.FromAsyncPattern in Part 1 of his Rx on the Server blog series. While the theory backing the Rx on the Server series is sound, it was written in mid 2010 and targets an old version of Rx.

我們將會看到Observable.FromAsyncPattern超過三十種的覆載方法，你可以選擇適合的來應用。我們先看以BeginXXX開頭的一般APM模式覆載，它需要零或多個資料參數，和AsyncCallback及一個物件，BeginXXX方法也回傳IAsyncResult。
```csharp
//Standard Begin signature
IAsyncResult BeginXXX(AsyncCallback callback, Object state);
//Standard Begin signature with data
IAsyncResult BeginYYY(string someParam1, AsyncCallback callback, object state);
```
EndXXX方法接受一個從BeginXXX回傳的IAsyncResult型別，並回傳一值。
```csharp
//Standard EndXXX Signature
void EndXXX(IAsyncResult asyncResult);
//Standard EndXXX Signature with data
int EndYYY(IAsyncResult asyncResult);
```
FromAsyncPattern方法的參數來自BeginXXX方法的參數，再加上EndXXX方法的回傳值，而在上述Stream.Read(byte[], int, int, AsyncResult, object)範例的使用中，可以看到我們要傳入一byte[]，二個整數，一個AsyncResult，再一個object給BeginRead。
```csharp
//IAsyncResult BeginRead(
//  byte[] buffer, 
//  int offset, 
//  int count, 
//  AsyncCallback callback, object state) {...}
Observable.FromAsyncPattern<byte[], int, int ...
```
而EndXXX方法回傳一個整數，這就完成了我們對FromAsyncPattern調用所需要的方法簽名。
```csharp
//int EndRead(
//  IAsyncResult asyncResult) {...}
Observable.FromAsyncPattern<byte[], int, int, int>
```
Observable.FromAsyncPattern並不回傳一個可觀察序列，而是回傳一個會回傳可觀察序列的委託，這個委託的方法的定義需符合前述FromAsyncPattern的簽名。
```csharp
var fileLength = (int) stream.Length;
//read is a Func<byte[], int, int, IObservable<int>>
var read = Observable.FromAsyncPattern<byte[], int, int, int>(
	stream.BeginRead, 
	stream.EndRead);
var buffer = new byte[fileLength];
var bytesReadStream = read(buffer, 0, fileLength);
bytesReadStream.Subscribe(
	byteCount =>
	{
		Console.WriteLine("Number of bytes read={0}, buffer should be populated with data now.",
		byteCount);
	});
```
記住這個實作只是一個範例，有一個很好的建立在Rx之上的實作，你可以瞭解看看：Rxx project，[http://rxx.codeplex.com](http://rxx.codeplex.com)。

本章談論到查詢的典型應用：建立可觀察序列。我們看到了各種不管是及早或是延遲建立序列的方式，也介紹了corecursion的概念，並使用Generate方法以unfold可能的無窮序列。也學到用不同的工廠方法建立時間相關的序列，也熟悉了如何將同步或非同步範式轉成序列，並可決定不同的轉換方式適用性，如下列方法：

* Factory Methods
	* Observable.Return
	* Observable.Empty
	* Observable.Never
	* Observable.Throw
	* Observable.Create
* Unfold methods
	* Observable.Range
	* Observable.Interval
	* Observable.Timer
	* Observable.Generate
* Paradigm Transition
	* Observable.Start
	* Observable.FromEventPattern
	* Task.ToObservable
	* `Task<T>.ToObservable`
	* `IEnumerable<T>.ToObservable`
	* Observable.FromAsyncPattern

建立可觀察序列是我們應用Rx的第一步：建立序列然後使用它，現在我們已經對建立的方式有了深刻的瞭解，再然後就可以學習操作可觀察序列的方法。

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