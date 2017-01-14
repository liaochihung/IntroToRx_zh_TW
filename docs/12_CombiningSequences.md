---
tags: Rx, C#
title: Rx介紹 Part 3 - Combining sequences(譯ing...)
---

##來自[Combining sequences](http://www.introtorx.com/Content/v1.0.10621.0/12_CombiningSequences.html#CombiningMultipleSequences)

到處都有資料，有些時候我們需要從不只一個地方取得資料。一般有多種輸入來源的資料，例如：多點觸控介面、news feeds、price feeds、多媒體聚合器、檔案監視器、heart-beating/polling servers等等。我們處理這些多重來源的方式也各有不同，可能想把它當成一筆大量的數據，或當成連續的資料，一次取一塊；也可以以順序的方式來取得，或將兩個來源的資料配對來處理，又或者只處理第一個響應的資料。

我們已經發現運算子組合的好處；現在我們把焦點放在序列的組合。之前，我們簡要的瞭解了諸如SelectMany、TakeUntil/SkipUntil、Catch及OnErrorResumeNext等多個序列的運算子，這些讓我們瞭解合成序列的潛在能力，藉著Rx中序列組合的特性，we find yet another layer of game changing functionality。序列的合成讓你可以在多重資料來源中建立複雜的查詢，這打開了我們編寫一些強大與簡潔程式的可能性。

現在我們將接著上一章「進階錯誤處理」中的概念，在序列錯誤後仍能繼續。我們將查驗在合成序列發生錯誤時可正常繼續而不會終止的運算子。

##Sequential concatenation
我們要看的第一個函式是能依序連結序列的函式，這很像我們之前看過能在發生錯誤時還正常處理的函式。

###Concat
Concat函式可能是最簡單的合成函式，它簡單地將兩個序列組合，一旦第一個序列完成，第二個序列隨即被訂閱，且推送出的值馬上會傳至結果序列中。它的行為就像Catch擴充函式，但只在序列結束後合成序列，而不是在OnError後的錯誤序列，它的函式定義如下：

```csharp
// Concatenates two observable sequences. Returns an observable sequence that contains the
//  elements of the first sequence, followed by those of the second the sequence.
public static IObservable<TSource> Concat<TSource>(
	this IObservable<TSource> first, 
	IObservable<TSource> second)
{
	...
}
```
Concat的使用也很像Catch或OnErrorResumeNext，我們傳進接續的序列給擴充函式。
```csharp
//Generate values 0,1,2 
var s1 = Observable.Range(0, 3);

//Generate values 5,6,7,8,9 
var s2 = Observable.Range(5, 5);

s1.Concat(s2)
	.Subscribe(Console.WriteLine);
```
```dos
s1 --0--1--2-|
s2           -5--6--7--8--|
r  --0--1--2--5--6--7--8--|
```
如果任一序列錯誤，那結果序列也會一樣。特別是若s1推送了OnError，那s2絕不會被加入。如果你不管s1如何結束也要加入s2，那只能用OnErrorResumeNext了。

Concat也有兩個有用的覆載；讓你可以用一個params陣列傳入多個可觀察序列或一個`IEnumerable<IObservable<T>>`。
```csharp
public static IObservable<TSource> Concat<TSource>(
	params IObservable<TSource>[] sources)
{...}
public static IObservable<TSource> Concat<TSource>(
	this IEnumerable<IObservable<TSource>> sources)
{...}
```
可傳入一個`IEnumerable<IObservable<T>>`代表多個序列可被延遲取值，這個需代入一個params陣列的覆載很適合在編譯期就知道多少個序列會被組合時來用，而`IEnumerable<IObservable<T>>`則適合在未知序列個數的時候使用。

在對`IEnumerable<IObservable<T>>`延遲取值的狀況下，Concat函式對一個序列訂閱，等到它結束，再換下一個序列。為了讓你更瞭解，我們建了一個回傳序列的序列，且包含log。它會回傳三個可觀察序列，並推送三個值，[1]、[2]跟[3]，每個序列在一段時間延遲後回傳值。
```csharp
public IEnumerable<IObservable<long>> GetSequences()
{
	Console.WriteLine("GetSequences() called");
	Console.WriteLine("Yield 1st sequence");
	yield return Observable.Create<long>(o =>
	{
		Console.WriteLine("1st subscribed to");
		return Observable.Timer(
			TimeSpan.FromMilliseconds(500))
			.Select(i=>1L)
			.Subscribe(o);
	});
	Console.WriteLine("Yield 2nd sequence");
	yield return Observable.Create<long>(o =>
	{
		Console.WriteLine("2nd subscribed to");
		return Observable.Timer(
			TimeSpan.FromMilliseconds(300))
		.Select(i=>2L)
		.Subscribe(o);
	});
	Thread.Sleep(1000);     //Force a delay
	Console.WriteLine("Yield 3rd sequence");
	yield return Observable.Create<long>(o =>
	{
		Console.WriteLine("3rd subscribed to");
		return Observable.Timer(
			TimeSpan.FromMilliseconds(100))
			.Select(i=>3L)
			.Subscribe(o);
		});
	Console.WriteLine("GetSequences() complete");
}
```
當我們呼叫GetSequences函式並組合結果後，我們可以用Dump函式看到如下的輸出。
```csharp
GetSequences().Concat().Dump("Concat");
```
輸出：
```dos
GetSequences() called
Yield 1st sequence
1st subscribed to
Concat-->1
Yield 2nd sequence
2nd subscribed to
Concat-->2
Yield 3rd sequence
3rd subscribed to
Concat-->3
GetSequences() complete
Concat completed
```
下列是在GetSequences函式後套用Concat運算子的marble圖，'s1'、's2'及's3'代表序列1、2跟3，'rs'代表結果序列。
```dos
s1-----1|
s2      ---2|
s3          -3|
rs-----1---2-3|
```
注意到第二個序列的產生是在第一個序列結束後才開始，為了證明這點，我們在推送值時加了一個500ms的延遲再結束，然後第二個序列就被訂閱，再來是第三個序列。

###Repeat
另一個簡單的擴充函式是Repeat，它讓你單純的重覆一個序列，不管是以指定的次數或是無限重覆。
```csharp
// Repeats the observable sequence indefinitely and sequentially.
public static IObservable<TSource> Repeat<TSource>(
	this IObservable<TSource> source)
{...}
//Repeats the observable sequence a specified number of times.
public static IObservable<TSource> Repeat<TSource>(
	this IObservable<TSource> source, 
	int repeatCount)
{...}
```
如果你使用無限重覆的覆載，唯一能停止它的方法是序列中有錯誤被推送，或是它的訂閱被取消。而指定次數的重覆，會在錯誤發生時、訂閱被取消後或次數到達時停止，下列範例顯示序列[0,1,2]被重覆三次。
```csharp
var source = Observable.Range(0, 3);
var result = source.Repeat(3);
result.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
```
輸出：
```dos
0
1
2
0
1
2
0
1
2
Completed
```
###StartWith
另一個簡單的連結函式是StartWith擴充函式，它讓你將數值加至序列的前面。函式的定義需代入一個參數陣列以讓你加入所需的數值。
```csharp
// prefixes a sequence of values to an observable sequence.
public static IObservable<TSource> StartWith<TSource>(
	this IObservable<TSource> source, 
	params TSource[] values)
{
	...
}
```
使用StartWith給我們類似`BehaviorSubject<T>`的行為，可用以確保在consumer訂閱時提供值。然而有一點和BehaviorSubject不一樣的是，它不會快取最後一個值。

在這個範例中，我們在序列[0,1,2]前加上-3、-2及-1。
```csharp
//Generate values 0,1,2 
var source = Observable.Range(0, 3);
var result = source.StartWith(-3, -2, -1);
	result.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
```
輸出：
```dos
-3
-2
-1
0
1
2
Completed
```
##Concurrent sequences
再來要瞭解的是對正在推送數值的可觀察序列的合併的函式，這是我們瞭解Rx旅途中很重要的一步，為了簡單起見，我們避免引進跟並發性有關的概念，直到現在我們對Rx基礎概念的廣泛瞭解。

###Amb
當我開始使用Rx時，Amb函式是一個新的概念。這是一個非確定性的函式，首先由John McCarthy介紹，單字是Ambiguous的縮寫。Rx實做將回傳來自先生成值的序列的值，並且會完全忽略其他序列。在下面的例子中，有三個序列都產生值，序列可以表示為下面的marble圖。
```dos
s1 -1--1--|
s2 --2--2--|
s3 ---3--3--|
r  -1--1--|
```
產生如上結果的程式碼如下所示：
```csharp
var s1 = new Subject<int>();
var s2 = new Subject<int>();
var s3 = new Subject<int>();
var result = Observable.Amb(s1, s2, s3);
result.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
s1.OnNext(1);
s2.OnNext(2);
s3.OnNext(3);
s1.OnNext(1);
s2.OnNext(2);
s3.OnNext(3);
s1.OnCompleted();
s2.OnCompleted();
s3.OnCompleted();
```
輸出：
```dos
1
1
Completed
```
如果我們把第一個s1.OnNext(1)註解掉，s2會變成第一個產生值的序列，而marble圖會如下：
```dos
s1 ---1--|
s2 -2--2--|
s3 --3--3--|
r  -2--2--|
```
Amb適合在當你有多個供值的可能來源，但取得值的延遲時間不一定時很有幫助。舉例來說，你可能在全球有數個重覆的伺服器，而從客戶端傳送或伺服端回應的負載都很輕，然而網路延遲時間無法預測且變化很大，透過Amb的使用，你可以同時送出查詢至多個伺服器，並只取第一個回應的結果。

Amb還有其它不一樣的型式，我們已經看了需代入一個參數陣列的狀況，你也可以把它當成擴充函式來用，並鏈結其它函式，直到所有的序列都包含（如：s1.Amb(s2).Amb(s3)）。最後，我們也可以傳入一個`IEnumerable<IObservable<T>>`。
```csharp
// Propagates the observable sequence that reacts first.
public static IObservable<TSource> Amb<TSource>(
	this IObservable<TSource> first, 
	IObservable<TSource> second)
{...}
public static IObservable<TSource> Amb<TSource>(
	params IObservable<TSource>[] sources)
{...}
public static IObservable<TSource> Amb<TSource>(
	this IEnumerable<IObservable<TSource>> sources)
{...}
```
再次使用我們在介紹Concat時的GetSequence函式，注意到外部序列的取值是取最先得到的。
```csharp
GetSequences().Amb().Dump("Amb");
```
輸出：
```dos
GetSequences() called
Yield 1st sequence
Yield 2nd sequence
Yield 3rd sequence
GetSequences() complete
1st subscribed to
2nd subscribed to
3rd subscribed to
Amb-->3
Amb completed
```
Marble圖：
```dos
s1-----1|
s2---2|
s3-3|
rs-3|
```
注意內部的可觀察序列並沒有被訂閱，一直到外部的序列已經全部產生。這表示第三個序列會最快回傳值，即時前兩個序列在一秒前已經產生（經過Thread.Sleep）。

###Merge
Merge擴充函式合併多個同部序列。當任一序列推送了值，此值馬上就會被加入結果序列，所有的序列的元素需為同一型別，如同前述函式。下圖中，我們可以看到s1跟s2同時產生值，此值也會同時被加入至結果函式。
```dos
s1 --1--1--1--|
s2 ---2---2---2|
r  --12-1-21--2|
```
Merge只會在所有輸入序列都完成時才結束，而如果任一輸入序列發生錯誤，Merge運算子也會錯誤。
```csharp
//Generate values 0,1,2 
var s1 = Observable.Interval(TimeSpan.FromMilliseconds(250))
	.Take(3);
//Generate values 100,101,102,103,104 
var s2 = Observable.Interval(TimeSpan.FromMilliseconds(150))
	.Take(5)
	.Select(i => i + 100);
s1.Merge(s2)
	.Subscribe(
		Console.WriteLine,
		()=>Console.WriteLine("Completed"));
```
上面程式的結果可以下列的marble圖表示，此例中，每個時間單位是50ms，而兩個來源序列都在750ms產生最後一個值，所以此時可能會有race condition存在，所以無法確認那一個值會先推送至結果序列(sR)中。
```dos
s1 ----0----0----0| 
s2 --0--0--0--0--0|
sR --0-00--00-0--00|
```
輸出：
```dos
100
0
101
102
1
103
104 //Note this is a race condition. 2 could be 
2 // published before 104. 
```
你可以鏈接Merge運算子的覆載函式以合併多重序列，它也提供數種覆載讓你可以傳入兩個以上的序列。靜態函式Observable.Merge可以在編譯時代入已知長度的參數陣列，也可以像Concat函式一樣代入IEnumerable序列，Merge函式也有接受`IObservable<IObservable<T>>`參數的巢狀式可觀察序列的覆載，總結一下：

- 鏈接Merge運算子，例如：s1.Merge(s2).Merge(s3)
- 傳入一序列的參數陣列至Observable.Merge靜態函式中，例如：Observable.Merge(s1,s2,s3)
- 傳入一`IEnumerable<IObservable<T>>`至Merge運算子中
- 傳入一`IObservable<IObservable<T>>`至Merge運算子中

```csharp
/// Merges two observable sequences into a single observable sequence.
/// Returns a sequence that merges the elements of the given sequences.
public static IObservable<TSource> Merge<TSource>(
	this IObservable<TSource> first, 
	IObservable<TSource> second)
{...}
// Merges all the observable sequences into a single observable sequence.
// The observable sequence that merges the elements of the observable sequences.
public static IObservable<TSource> Merge<TSource>(
	params IObservable<TSource>[] sources)
{...}
// Merges an enumerable sequence of observable sequences into a single observable sequence.
public static IObservable<TSource> Merge<TSource>(
	this IEnumerable<IObservable<TSource>> sources)
{...}
// Merges an observable sequence of observable sequences into an observable sequence.
// Merges all the elements of the inner sequences in to the output sequence.
public static IObservable<TSource> Merge<TSource>(
	this IObservable<IObservable<TSource>> sources)
{...}
```
對於合併已知數量的序列，前兩個覆載其實是一樣的，使用那一種只是個人的風格，若不是以參數式陣列提供，就是以鏈接的方式，而第三和第四個覆載讓你可以在執行時以延遲估值的方式進行對序列的合併。Merge函式中代入序列的序列是一個有趣的概念，不管你用pull或pushed式的可觀察序列，都會馬上被訂閱。

如果我們再使用一個GetSequences函式，我們可以看到Merge函式如何和序列的序列合作。
```csharp
GetSequences().Merge().Dump("Merge");
```
Output:
```dos
GetSequences() called
Yield 1st sequence
1st subscribed to
Yield 2nd sequence
2nd subscribed to
Merge-->2
Merge-->1
Yield 3rd sequence
3rd subscribed to
GetSequences() complete
Merge-->3
Merge completed
```
如下marble圖中可見，s1及s2產生後馬上就被訂閱，s3一秒後才產生且被訂閱，一旦所有的輸入序列已完成，結果序列也跟著完成。
```dos
s1-----1|
s2---2|
s3          -3|
rs---2-1-----3|
```
###Switch
從巢狀的可觀察序列取得所有值並不總是我們需要的，某些狀況下，你可以只想取得最近的內部序列的值，而不是接收所有的值。一個很好的例子是即時搜索。在輸入時，文字被傳送至搜索服務，結果將做為一可觀察序列回傳。大多的實作在傳送前會有輕微的延遲，從而不會有多餘的工作。想像一下，我想搜尋“Intro to Rx"，我快速輸入”Into to"後發現少了字母"r"，我停下並變更成"Intro "，此時，兩個搜尋已被傳送至伺服端，而第一個搜尋的結果不是我想要的。此外，如果我要接受第一次與第二次合併的數據，對於使用者來說會很奇怪。這種情況下很適合使用Switch函式。

本範例中，有一個代表搜索文字序列的來源，使用者輸入的表代表來源序列，使用Select函式，我們傳入搜尋字串至一參數為字串並回傳`IObservable<string>'的函式，這會建立我們的巢狀序列結果，`IObservable<IObservable<string>>`

搜索函式的定義：
```csharp
private IObservable<string> SearchResults(string query)
{
	...
}
```
在此重覆搜尋中使用Merge函式：
```csharp
IObservable<string> searchValues = ....;
IObservable<IObservable<string>> search = searchValues
	.Select(searchText=>SearchResults(searchText));
var subscription = search
	.Merge()
	.Subscribe(
		Console.WriteLine);
```
如果我們夠幸運，並且在searchValues的下一個元素產生之前完成了每個搜索，輸出將顯得合理。然而，更有可能的是，多個搜索將導致重疊的搜索結果。 這個marble圖顯示了合併功能在這種情況下可以做什麼。

- SV是searchValues序列
- S1是searchValues / SV中第一個值的搜索結果序列
- S2是searchValues / SV中第二個值的搜索結果序列
- S3是searchValues / SV中第三個值的搜索結果序列
- RM是合併（結果合併）序列的結果序列

```dos
SV--1---2---3---|
S1  -1--1--1--1|
S2      --2-2--2--2|
S3          -3--3|
RM---1--1-2123123-2|
```
注意搜索結果中的值如何混合在一起。這不是我們想要的東西。如果我們使用Switch擴展方法，我們將獲得更好的結果。 交換機將訂閱外部序列，並且隨著每個內部序列被產生，它將訂閱新的內部序列並且丟棄對先前內部序列的訂閱。 這將產生以下marble圖，其中RS是Switch（Result Switch）序列的結果序列：

```dos
SV--1---2---3---|
S1  -1--1--1--1|
S2      --2-2--2--2|
S3          -3--3|
RS --1--1-2-23--3|
```
還要注意，即使S1和S2的結果仍在推送，它們會被忽略掉，因為他們的訂閱已被處理。這消除了來自巢狀序列的重疊值的問題。

##Pairing sequences
之前的函式讓我們可以將多重序列展開，以共享同型別進入同型別的結果序列。下一組函式仍然使用多個序列當做輸入，但嘗試從每個序列的值去配對，以產生用於結果序列的單一輸出值。一下條件下，他們也讓你可以提供不同類型的序列。

###CombineLatest
CombineLatest擴充函式允許從兩個序列獲取最近的值，並且使用給定的函數將它們轉換為結果序列的值。每個輸入序列有像Replay(1)一樣被快取的最後一個值。一旦兩個序列都產生了至少一個值，則每次每個序列產生一個值時，每個序列的最新輸出被傳遞給resultSelector函式。定義如下。

```csharp
// Composes two observable sequences into one observable sequence by using the selector 
//  function whenever one of the observable sequences produces an element.
public static IObservable<TResult> CombineLatest<TFirst, TSecond, TResult>(
	this IObservable<TFirst> first, 
	IObservable<TSecond> second, 
	Func<TFirst, TSecond, TResult> resultSelector)
{...}
```
下面的marble圖顯示了CombineLatest與產生數字(N)和其他字母(L)的一個序列的用法。如果resultSelector函式只是將數字和字母連接在一起成對，這將是結果(R)：
```dos
N---1---2---3---
L--a------bc----
R---1---2-223---
    a   a bcc   
```
如果我們慢慢來看上面的marble圖，我們首先看到L產生字母'a'。N沒有產生任何值，所以沒有什麼要配對，沒有為結果(R)產生值。接下來，N產生數字'1'，所以我們現在有一個'1a'在結果序列中產生。然後我們從N接收數字'2'。最後一個字母仍然是'a'，所以下一配對是'2a'。然後產生字母“b”，建立配對“2b”，隨後是“c”給出“2c”。最後，產生數字3，我們得到配對'3c'。

如果你需要評估一些狀態的組合，且需要已變更狀態的最新值，這很有用處。一個簡單的範例是監視系統，每個服務代表了一個會回傳布林值以代表服務可用性的序列，如果所有的服務都可用則監視狀態為綠色；我們可以通過讓結果選擇器執行邏輯And的動作來實作這個，下列是一個範例：
```csharp
IObservable<bool> webServerStatus = GetWebStatus();
IObservable<bool> databaseStatus = GetDBStatus();
//Yields true when both systems are up.
var systemStatus = webServerStatus
	.CombineLatest(
		databaseStatus,
		(webStatus, dbStatus) => webStatus && dbStatus);
```
一些讀者可能已經注意到，這種方法可能產生很多重複的值。例如，如果Web伺服器關閉，結果序列將產生'false'。如果資料庫關閉，將產生另一個（不必要的）“false”值。這將是使用DistictUntilChanged擴充函式的適合時間。更新的程式將如下面的範例。
```csharp
//Yields true when both systems are up, and only on change of status
var systemStatus = webServerStatus
	.CombineLatest(
		databaseStatus,
		(webStatus, dbStatus) => webStatus && dbStatus)
	.DistinctUntilChanged();
```
為提供一個更好的服務，我們可以通過為序列加上'false'來提供預設值。
```csharp
//Yields true when both systems are up, and only on change of status
var systemStatus = webServerStatus
	.CombineLatest(
		databaseStatus,
		(webStatus, dbStatus) => webStatus && dbStatus)
	.DistinctUntilChanged()
	.StartWith(false);
```
###Zip
Zip擴充函式是另一個有趣的合併功能。就像在服裝或包包上的拉鍊，Zip函式將兩個值序列作為配對；兩個兩個。關於Zip函式的注意事項是，當第一個序列完成時，結果序列將完成，如果任一序列錯誤，則它將會錯誤，並且一旦它具有來自每個來源序列的一對新值，它將僅推送一次。因此，如果一個來源序列公佈的值比另一個序列快，則發佈速率將由兩個序列中較慢的一個決定。

```csharp
//Generate values 0,1,2 
var nums = Observable.Interval(TimeSpan.FromMilliseconds(250))
	.Take(3);
//Generate values a,b,c,d,e,f 
var chars = Observable.Interval(TimeSpan.FromMilliseconds(150))
	.Take(6)
	.Select(i => Char.ConvertFromUtf32((int)i + 97));
//Zip values together
nums.Zip(chars, (lhs, rhs) => new { Left = lhs, Right = rhs })
	.Dump("Zip");
```
這可以看下方的marble圖，注意結果序列使用兩列，所以我們可以呈現較複雜的型別，例如：有Left及Right兩個屬性的匿名型別。
```dos
nums  ----0----1----2| 
chars --a--b--c--d--e--f| 
result----0----1----2|
          a    b    c|
```
實際程式碼的輸出：
```csharp
{ Left = 0, Right = a }
{ Left = 1, Right = b }
{ Left = 2, Right = c }
```
注意nums序列在完成之前只產生三個值，而chars序列產生六個值。結果序列因此具有三個值，因為這是可以做出的最多的配對。

我看到的Zip的第一個用途是展示拖放。該範例從MouseMove事件跟踪游標移動，該事件將產生具有當前X，Y坐標的事件參數。首先，範例將事件轉換為可觀察序列，然後他們巧妙地用相同序列的Skip(1)版本壓縮序列。這允許程式獲得游標位置的增量，即現在(sequence.Skip(1))減去它在哪裡(序列)。然後，它將delta差值應用到它拖動的控制項。

為了讓概念可視化，讓我們看另一個marble圖，這裡我們有滑鼠移動(MM)及Skip 1(S1)，數字顯示滑鼠移動的索引。
```dos
MM --1--2--3--4--5
S1    --2--3--4--5
Zip   --1--2--3--4
        2  3  4  5
```
這邊是我們用我們的subject假裝滑鼠位移的範例程式。
```csharp
var mm = new Subject<Coord>();
var s1 = mm.Skip(1);
var delta = mm.Zip(s1,
	(prev, curr) => new Coord
	{
		X = curr.X - prev.X,
		Y = curr.Y - prev.Y
	});
delta.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
mm.OnNext(new Coord { X = 0, Y = 0 });
mm.OnNext(new Coord { X = 1, Y = 0 }); //Move across 1
mm.OnNext(new Coord { X = 3, Y = 2 }); //Diagonally up 2
mm.OnNext(new Coord { X = 0, Y = 0 }); //Back to 0,0
mm.OnCompleted();
```
這是我們使用的簡單Coord(inate)類別。
```csharp
public class Coord
{
	public int X { get; set; }
	public int Y { get; set; }
	public override string ToString()
	{
		return string.Format("{0},{1}", X, Y);
	}
}
```
輸出：
```dos
0,1
2,2
-3,-2
Completed
```
值得注意的是Zip有第二個覆載，可代入一個`IEnumerable<T>'當做第二個輸入序列。
```csharp
// Merges an observable sequence and an enumerable sequence into one observable sequence 
//  containing the result of pair-wise combining the elements by using the selector function.
public static IObservable<TResult> Zip<TFirst, TSecond, TResult>(
	this IObservable<TFirst> first, 
	IEnumerable<TSecond> second, 
	Func<TFirst, TSecond, TResult> resultSelector)
{...}
```
這讓我們從`IEnumerable <T>`和`IObservable <T>`範式中壓縮序列！

###And-Then-When
If Zip only taking two sequences as an input is a problem, then you can use a combination of the three And/Then/When methods. These methods are used slightly differently from most of the other Rx methods. Out of these three, And is the only extension method to IObservable<T>. Unlike most Rx operators, it does not return a sequence; instead, it returns the mysterious type Pattern<T1, T2>. The Pattern<T1, T2> type is public (obviously), but all of its properties are internal. The only two (useful) things you can do with a Pattern<T1, T2> are invoking its And or Then methods. The And method called on the Pattern<T1, T2> returns a Pattern<T1, T2, T3>. On that type, you will also find the And and Then methods. The generic Pattern types are there to allow you to chain multiple And methods together, each one extending the generic type parameter list by one. You then bring them all together with the Then method overloads. The Then methods return you a Plan type. Finally, you pass this Plan to the Observable.When method in order to create your sequence.
如果Zip只能有兩個序列作為輸入是一個問題，那麼你可以使用三個And / Then / When函式的組合。這些函式與大多數其他Rx函式的使用略有不同，在這三個之中，並且是`IObservable <T>`的唯一擴充函式。與大多數Rx運算子不同，它不回傳序列；相反，它回傳奇妙的型別`Pattern <T1，T2>`。`Pattern <T1，T2>`型別是public(顯然地)，但它的所有屬性都是internal。你可以用`Pattern <T1，T2>`做的只有兩個（有用的）呼叫它的And或Then函式。在`Pattern<T1，T2>`上呼叫的And函式回傳`Pattern<T1，T2，T3>`。在那種型別上，你還會​​發現And和Then函式。通用模式類型允許你將多個And方法鏈接在一起，每個函式將通用類型參數列表擴展一個。然後將它們全部與Then方法覆載。Then函式回傳一個Plan型別。最後，你將此計劃傳遞給Observable.When函以建立序列。

這聽起來很複雜，但是比較一些程式碼範例應該更容易理解。它還可以讓你選擇自己喜歡的風格。

要將三個序列一起壓縮，您可以使用鏈接在一起的Zip方法，如下所示：
```csharp
var one = Observable.Interval(TimeSpan.FromSeconds(1)).Take(5);
var two = Observable.Interval(TimeSpan.FromMilliseconds(250)).Take(10);
var three = Observable.Interval(TimeSpan.FromMilliseconds(150)).Take(14);
//lhs represents 'Left Hand Side'
//rhs represents 'Right Hand Side'
var zippedSequence = one
	.Zip(two, (lhs, rhs) => new {One = lhs, Two = rhs})
	.Zip(three, (lhs, rhs) => new {One = lhs.One, Two = lhs.Two, Three = rhs});
zippedSequence.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
```
或者使用And/Then/When較好的語法：
```csharp
var pattern = one.And(two).And(three);
var plan = pattern.Then((first, second, third)=>new{One=first, Two=second, Three=third});
var zippedSequence = Observable.When(plan);
zippedSequence.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
```
如果你想，也可以縮減成這樣：
```csharp
var zippedSequence = Observable.When(
	one.And(two)
	.And(three)
	.Then((first, second, third) => 
		new { 
			One = first, 
			Two = second, 
			Three = third 
		})
	);
zippedSequence.Subscribe(
	Console.WriteLine,
	() => Console.WriteLine("Completed"));
```
The And/Then/When trio has more overloads that enable you to group an even greater number of sequences. They also allow you to provide more than one 'plan' (the output of the Then method). This gives you the Merge feature but on the collection of 'plans'. I would suggest playing around with them if this functionality is of interest to you. The verbosity of enumerating all of the combinations of these methods would be of low value. You will get far more value out of using them and discovering for yourself.
And / Then /When三元組有更多的覆載，讓你能夠分組更多的序列。它們還允許你提供多個“plan”（Then方法的輸出）。這給你合併功能但是是在“plan”集合中。如果你對這個功能感興趣，我建議你瞭解下它。列舉所有這些函式的組合是冗長且低效的，靠著你自己去使用及發現會比較有用。

當我們深入了解Rx函式庫為我們提供的功能時，我們可以看到更多的實用用法。用Rx建構序列使我們能夠輕鬆地理解問題領域公開的多個資料來源。我們可以用StartWith，Concat和Repeat來連接值或序列。我們可以用Merge同時處理多個序列，或者一次使用Amb和Switch處理單個序列。將值與CombineLatest，Zip和And / Then / When操作配對可以簡化其他操作，如拖放範例和監視系統狀態。