# 04｜非同步、執行緒與併發

[回目錄](../README.md) · [快速複習](../quick-review.md)

## 1. Process、Thread、併發與平行

**白話與用途：** Process（行程）像一間餐廳，有自己的資源與位址空間；Thread（執行緒）像廚師，同一餐廳的廚師共用許多資源。併發是多件工作在同一段時間推進；平行是同一時間真的由多個執行單位工作。

**實際場景：** 多個 API 請求同時進行，可能在等待資料庫時交替推進；大量影像計算則可能用多核心平行。Thread Pool 重用執行緒，減少一直建立與銷毀的成本。

**面試回答：**「行程有自己的資源與位址空間，執行緒是行程內的執行單位並共用資料。併發是工作交錯推進，平行是同時執行；單核心也能有併發。」

**追問／易錯點：** 共用資料帶來競爭風險；跨行程也能透過 IPC 共用或交換資料。執行緒越多不一定越快，會增加切換與資源成本。

## 2. async／await 與 I/O-bound、CPU-bound

**白話與用途：** 點餐後拿號碼牌，等待時可以去做別的事，不必一直站在櫃台。`Task` 表示操作的完成狀態；`await` 在 Task 尚未完成時，暫停這個方法的後續流程並把控制權交還呼叫端，完成後再繼續。

**實際場景：** 等資料庫、檔案、HTTP 是 I/O-bound，使用真正的非同步 API 可避免占住執行緒等待。大量計算是 CPU-bound，仍要消耗 CPU；在桌面 UI 可用 `Task.Run` 把計算移到背景，後端則要評估容量，不能靠包一層 Task.Run 增加 CPU。

```csharp
// httpClient 為可重用或由工廠管理的 HttpClient；token 由呼叫端傳入。
static async Task<string> LoadAsync(
    HttpClient httpClient, string url, CancellationToken token)
{
    string body = await httpClient.GetStringAsync(url, token);
    return body;
}
```

**面試回答：**「async／await 讓等待非同步操作時不用阻塞目前執行緒，常用在 I/O。它不保證開新執行緒，也不會把 CPU 計算自動變快；主要讓等待期間的資源能服務其他工作。」

**追問／易錯點：** async 方法通常會同步執行到第一個尚未完成的 await；Task 已完成時可以直接繼續。避免 `.Result`／`.Wait()` 阻塞；特定同步內容可能死結，後端大量阻塞也可能耗盡執行緒池。除了事件處理器，通常回傳 Task，避免 `async void` 難以等待與處理例外。[Microsoft：非同步情境](https://learn.microsoft.com/en-us/dotnet/standard/async-in-depth)

## 3. Race Condition、原子性與鎖

**白話與用途：** 兩位店員都看到最後一件商品，各自答應賣出，就超賣了。競爭條件（Race Condition）表示結果受執行時序影響；「讀取 → 判斷 → 修改」不是自然就不可分割的原子操作。

**實際場景：** 共用計數器的 `count++` 包含讀、加、寫，可能丟失更新。單一計數可用 `Interlocked`；多步驟的共用記憶體規則可用 lock 保護。

```csharp
public sealed class Stock
{
    private readonly object _gate = new();
    private int _remaining = 1;

    public bool TryTake()
    {
        lock (_gate)
        {
            if (_remaining <= 0) return false;
            _remaining--;
            return true;
        }
    }
}
```

**面試回答：**「競爭條件是多個工作共用資料，交錯執行讓結果不正確。我會把需要一起完成的步驟視為臨界區，用適合的原子操作或同步機制保護。」

**追問／易錯點：** lock 只能協調使用同一鎖物件的程式，這裡要共用同一個 Stock 實例。它不能保護另一台主機或另一個行程的庫存；資料庫問題要靠交易、條件更新或約束。執行緒安全集合也不保證多個操作組合後仍具原子性。

## 4. Deadlock 與避免長時間持鎖

**白話與用途：** A 拿著鑰匙等 B 的文件，B 拿著文件等 A 的鑰匙，兩人都不放手，就是死結。鎖保護正確性，但互相等待可能讓工作永遠停住。

**實際場景：** 兩段程式以相反順序取得兩把鎖；或交易以不同順序更新兩筆帳戶。常見防法是固定取得順序、縮小臨界區，並避免持鎖呼叫外部服務。

**面試回答：**「死結是工作互相持有對方需要的資源，形成循環等待。我會固定鎖定順序、縮短持鎖時間；資料庫偵測並中止交易後，才評估能否安全重試整個交易。」

**追問／易錯點：** lock 區塊不能直接 await。需要非同步協調時可用 `SemaphoreSlim.WaitAsync`，並在成功取得後用 finally Release；它也只保護同一個實例。設定逾時能停止等待，但不等於根除死結。

## 5. 多個 Task、取消與限制併發

**白話與用途：** 可以同時向兩家店詢價，最後一起看結果；但不能一次派十萬人衝進同一家店。獨立工作可一起啟動，用 `Task.WhenAll` 等待；併發量要配合外部服務、CPU 與連線池容量。

**實際場景：** 同時取得兩個獨立 API 的結果，減少總等待時間。此處是 I/O 操作重疊，並不代表兩個專用執行緒一直工作。

```csharp
static async Task<string[]> LoadBothAsync(
    HttpClient client, string urlA, string urlB, CancellationToken token)
{
    Task<string> first = client.GetStringAsync(urlA, token);
    Task<string> second = client.GetStringAsync(urlB, token);
    return await Task.WhenAll(first, second);
}
```

**面試回答：**「獨立工作可以先一起啟動，再用 WhenAll 等待，但要限制併發避免塞滿連線池。取消是合作式訊號，應一路傳給支援的 API，並在需要時檢查 token。」

**追問／易錯點：** WhenAll 不會自動取消其他工作；它等所有 Task 結束。呼叫端取消不代表伺服器一定沒完成寫入；逾時後重送仍要考慮冪等性。別在請求內隨便啟動無人追蹤的背景 Task 做重要工作。
