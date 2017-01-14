---
tags: Rx, C#
title: Rx介紹 Part 1 - Start(譯)
---

> This is a Traditional Chinese version of [IntroToRx](http://www.introtorx.com/), and still in progress. The source hosted on [GitHub](https://github.com/liaochihung/IntroToRx_zh_TW/), and use [ReadTheDocs](https://readthedocs.org/projects/introtorx-zh-tw/) to build.

## 為什麼要用Rx?
使用者想要即時的資料。想看到即時的推特、訂單馬上被確認、價格是最準確的、線上遊戲不延遲。身為一個開發者，你可能想要一個"發出即忘"的訊息系統，你不想因等待結果而阻塞系統，你希望事情完成時通知你，甚至，當你在等待一串事件的完成時，你想要可以分別得到每個事件完成時的通知。你不想等待所有事件完成才能看到第一筆結果。這個世界已經轉向使用"Push"了，但使用者仍等著我們迎頭趕上，我們已有了工具幫忙送出資料，這很簡單，但是我們需要工具去幫我們響應"Push data"。

##Rx適合在？
Rx提供了一個很自然的處理一連串事件的型式。事件串列中可以包含零或多個事件，Rx被證明在處理需合成的事件時最有用。
###適合以下情況
Rx是建立在管理下列事件：
* UI事件
* 領域事件，如“屬性變更“、”集合更新”、“訂單已填寫”、“註冊完成”等。
* 架構事件，如檔案變更監視器，系統及WMI事件等。
* 整合事件，如來自訊息bus的廣播或是WebSockets API的推送事件或低延遲的中介器如[Nirvana](http://www.my-channels.com)等。
* 和CEP等引擎整合的事件，如[StreamInsight](http://www.microsoft.com/sqlserver/en/us/solutions-technologies/business-intelligence/complex-event-processing.aspx)或[StreamBase](http://www.streambase.com)。

有趣的是微軟的CEP產品StreamInsight – 也是SQL Server家族之一，也使用了LINQ去建立串流事件資料的查詢。
在_減輕負載_的目的下，Rx也很適合被引進並管理同步機制，如在當前執行緒中執行一組同步的工作以減輕負載。很受歡迎的應用是維護一個響應式的介面。
你應該考慮使用Rx，如果你想用`IEnumerable<T>`來建立變化的資料, 雖然`IEnumerable<T>`也可以做到（透過延遲取值 - `yield return`），但可能失卻彈性，而且透過`IEnumerable<T>`取值時，實際上可能會阻塞到執行緒，你應該使用非阻塞式的`IObservable<T>`或考慮.NET 4.5的 `async`功能。
###Rx可以用在
Rx也可以被用在非同步呼叫中，這些是很有效的事件序列，
* Task或`Task<T>`的回傳值
* APM方法呼叫的回傳值，如`FileStream`的BeginRead/EndRead
你可能發現使用TPL, Dataflow 或 async關鍵字(.NET 4.5)對處理非同步方法會提供一種較自然的方式，而Rx也可以在這些狀況下幫到忙，但如果有更適合的frameworks可以處理，你應優先考慮使用它。
Rx可以被使用但不那麼適合的是為了彈性或執行平行計算的目的而導入及管理的同步處理，其它專用的framework如TPL (Task Parallel Library) 或 C++ AMP更適合執行平行計算的工作。

這邊可看更多關於 TPL, Dataflow, async 及 C++ AMP -  [Microsoft's Concurrency homepage](http://msdn.microsoft.com/en-us/concurrency)。

###Rx不適用在
Rx或者說`IObservable<T>`不是用來替換`IEnumerable<T>`的，我並不建議你去試著將Pull的事件變為Push的事件型態
* 僅為了想用"Rx"而轉換現有的`IEnumerable<T>`至`IObservable<T>`
* 訊息佇列，如 MSMQ 或 JMS 這種佇列系統的實作，一般來講都有它自己的事務性而且都預設是順序式的，我覺得`IEnumerable<T>`是最適合的
在工作上選擇最適合的工具會讓你更容易去維護，並提供更好的效能。

##Rx in action
適應及學習Rx可以是一個反覆式的過程，你可以慢慢的應用到你的架構及領域中，短期時間內，你應該就能夠產出程式碼，或是縮減已存在的程式，以簡單的運算子執行複合的查詢。下列這一段簡單的ViewModel示範程式展示了使用者輸入時搜尋的功能。
```csharp
public class MemberSearchViewModel : INotifyPropertyChanged
{
	//Fields removed...
	public MemberSearchViewModel(IMemberSearchModel memberSearchModel,
	ISchedulerProvider schedulerProvider)
	{
		_memberSearchModel = memberSearchModel;
		//Run search when SearchText property changes
		this.PropertyChanges(vm => vm.SearchText).Subscribe(Search);
	}
	
	//Assume INotifyPropertyChanged implementations of properties...
	public string SearchText { get; set; }
	public bool IsSearching { get; set; }
	public string Error { get; set; }
	public ObservableCollection<string> Results { get; }

	//Search on background thread and return result on dispatcher.
	private void Search(string searchText)
	{
		using (_currentSearch) { }
		IsSearching = true;
		Results.Clear();
		Error = null;
		_currentSearch = _memberSearchModel.SearchMembers(searchText)
			.Timeout(TimeSpan.FromSeconds(2))
			.SubscribeOn(_schedulerProvider.TaskPool)
			.ObserveOn(_schedulerProvider.Dispatcher)
			.Subscribe(
				Results.Add,
				ex =>
				{
					IsSearching = false;
					Error = ex.Message;
				},
				() => { IsSearching = false; });
	}
...
}
```
 雖然這段程式碼很簡短，它仍達到了下列目的：
>* 維護響應式的介面
* 逾時處理
* 知道何時完成搜尋
* 一次回傳所有結果
* 錯誤處理
* 可單元測試性，即使考慮到同步的狀況
* 如果使用者變更搜尋，會取消當前的搜尋及以新字串執行新的搜尋
產生這個範例是使用符合需求的單一查詢所合成的運算子，這個查詢簡單、易於維護、宣告式且遠比自己寫的短。重複使用經過良好測試的API是額外的益處，要寫的程式碼越短，需要測試、除錯及維護的程式碼也越少。建立如下的其它查詢也很簡單：
* 計算一串數字的移動平均值，例如服務層的平均延遲及停機時間協議
* 合成多個來源的事件資料，例如從Bing,Google,Yahoo來的搜尋結果…
* 資料集群化，如某主題或使用者的推特，或股票價值的變化等
* 資料過濾，如在一個區域內的線上遊戲伺服器，對個別遊戲或最小數量的參與者

"Push"的時代來了，在這個時代，用Rx武裝你自己是很有效的迎合使用者預期的方式，依照你對Rx的瞭解及組合Rx命令的方式，你可用之以處理依序而來的複雜訊息，Rx是為了讓你在日複一日寫程式時使用的。