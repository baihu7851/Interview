# 03｜型別、記憶體與資源

[回目錄](../README.md) · [快速複習](../quick-review.md)

## 1. 值型別、參考型別與參數傳遞

**白話與用途：** 值型別像複印表格，修改副本通常不影響原本；參考型別像複印住址，兩張紙可能指向同一棟房子。差別是複製了什麼，能幫助判斷某次修改是否會影響其他地方。

**實際場景：** `int`、`decimal`、`struct` 是值型別；一般 `class`、陣列、`string` 是參考型別。C# 預設參數按值傳遞：參考型別傳的是「參考的副本」。

```csharp
int a = 10;
int b = a;
b = 20;                        // a 仍是 10

var first = new List<int> { 1 };
var second = first;
second.Add(2);                  // first 也看得到 2
second = new List<int> { 9 };    // first 仍指向原清單
```

**面試回答：**「值型別預設複製值，參考型別複製參考，所以可能共用同一物件。參考型別參數也預設按值傳遞；改物件內容會影響原物件，但把參數改指向新物件不會改掉呼叫端變數。」

**追問／易錯點：** `ref` 可讓方法修改呼叫端變數本身；`out` 用來輸出且方法需賦值。struct 若含 List 等參考欄位，複製 struct 後仍可能共用該物件，並非深層複製。[Microsoft：值型別](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-types)

## 2. Stack、Heap 與 Boxing

**白話與用途：** 呼叫堆疊（Stack）像一層層待完成的工作單，記錄方法呼叫與返回；受控堆積（Managed Heap）主要放由 GC 管理的物件。理解它們有助於判斷配置成本、物件生命週期與堆疊溢位。

**實際場景：** 遞迴太深可能耗盡呼叫堆疊；大量建立暫時物件會增加 GC 壓力。值型別轉成 `object` 時，通常發生 Boxing：建立物件存放值的副本；Unboxing 再取回原值型別。

```csharp
int number = 7;
object boxed = number;  // Boxing
number = 8;
int copy = (int)boxed;  // Unboxing，得到 7
```

**面試回答：**「Stack 主要管理方法呼叫，Heap 主要放 GC 管理的物件。不能直接說值型別一定在 Stack；例如 class 裡的 int 欄位就在物件裡。Boxing 把值包成物件，反覆發生可能增加配置成本。」

**追問／易錯點：** 實際儲存位置受上下文與執行階段最佳化影響。泛型集合能避免許多不必要的 Boxing，但不是所有泛型使用都保證零配置。

## 3. GC 與記憶體洩漏

**白話與用途：** GC 像管理倉庫的人，回收程式已無法再找到的物件，減少手動釋放記憶體的錯誤。它不是看「業務上還要不要」，而是看物件是否仍可從存活的根參考到達。

**實際場景：** static 清單持續累積、快取沒有上限、長命物件的事件訂閱保留短命物件，都可能讓已不需要的資料仍無法回收。

.NET 用分代管理許多物件：短命物件通常在較年輕世代回收，存活者可能晉升。大量配置會增加回收工作，也可能造成暫停；不是有 GC 就可以完全不管配置量。

**面試回答：**「GC 自動回收不可達的受控物件，但還被 static、快取或事件引用的物件仍會留下，所以有 GC 也可能洩漏。我會觀察記憶體趨勢，再找是哪個參考把物件留住。」

**追問／易錯點：** 記憶體上升不一定是洩漏，也可能是快取、工作量或 GC 尚未回收。不要先用 `GC.Collect()` 當通用修正。[Microsoft：GC](https://learn.microsoft.com/dotnet/standard/garbage-collection/)

## 4. IDisposable 與 using

**白話與用途：** GC 管記憶體，檔案把手、連線等資源卻常需要及時歸還。`IDisposable` 提供明確清理的契約，`using` 確保離開範圍時呼叫 `Dispose`，即使中途發生例外也一樣。

**實際場景：** 檔案沒關會被占用；資料庫連線沒歸還可能耗盡連線池。Dispose 不等於把物件記憶體立刻交給 GC；清理內容取決於實作。

```csharp
static string ReadFirstLine(string path)
{
    using var reader = new StreamReader(path);
    return reader.ReadLine() ?? "";
} // 離開方法時 Dispose，關閉檔案
```

**面試回答：**「GC 負責受控記憶體，Dispose 負責明確釋放或歸還資源。using 能保證範圍結束時清理；資料庫連線 Dispose 通常是歸還連線池，不一定真的斷開網路。」

**追問／易錯點：** 非同步清理可用 `await using`。資源所有權要清楚：不應任意 Dispose 別人傳進來且仍會使用的物件；也不要把已 Dispose 的物件回傳出去。

## 5. 不可變、相等與 null

**白話與用途：** 不可變物件像已定稿的文件，修改時建立新版本，能減少共用資料被偷偷改動。相等則要先問：是在比同一物件，還是比內容？null 表示沒有值，不等於空字串或 0。

**實際場景：** `string` 是參考型別但不可變，`==` 比字串內容。一般 class 預設的 `Equals` 比參考；`record` 提供值相等語意，但其中的 List 等成員不會自動逐項深層比較。

```csharp
string name = "小明";
string other = name;
other += "同學";               // name 仍是「小明」

string? input = null;
int length = input?.Length ?? 0;
```

**面試回答：**「不可變物件建立後內容不變，適合共用。相等要看型別規則：同一物件和內容相同是兩件事。可空型別提醒我處理沒有值的情況，不能把 null 和空字串混在一起。」

**追問／易錯點：** `readonly` 參考欄位限制重新賦值，不保證物件內容不能改。Nullable reference types 主要提供編譯期警告，不會自動擋住所有執行期 null。

## 6. LINQ 的延遲執行

**白話與用途：** 建立查詢像寫好購物清單，真正走去採買才取得東西。許多 LINQ 操作在列舉時才執行，能串接條件並避免過早建立中間清單。

**實際場景：** 重複列舉可能重算；資料庫查詢可能重送 SQL。`ToList()` 會立即列舉並保存當時結果，代價是配置記憶體。

```csharp
var numbers = new List<int> { 1, 2, 3 };
var query = numbers.Where(n => n > 1);
numbers.Add(4);
Console.WriteLine(string.Join(",", query)); // 2,3,4
var snapshot = query.ToList();
```

**面試回答：**「許多 LINQ 查詢在列舉時才執行，所以查詢建立後資料變動可能影響結果，重複列舉也可能重複工作。需要固定結果或多次使用時可 ToList，但要評估資料量。」

**追問／易錯點：** `IEnumerable<T>` 是列舉能力；`IQueryable<T>` 還能讓 provider 處理查詢運算式，不保證所有 C# 方法都能翻成 SQL。`Count()` 等終端操作會執行；不是所有 LINQ 都延遲。
