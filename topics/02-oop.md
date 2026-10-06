# 02｜OOP 與程式設計

[回目錄](../README.md) · [快速複習](../quick-review.md)

## 1. 封裝與抽象

**白話與用途：** 提款機提供提款按鈕，不讓你直接改銀行餘額。封裝把狀態和規則放在一起，控制合法變更；抽象讓使用者只需理解必要操作，不必知道全部細節。兩者能減少誤用與修改連動。

**實際場景：** 訂單只能從未付款轉成已付款，而非讓任何地方任意寫入狀態。`private` 是工具，真正目的在保護規則。

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount));
        Balance += amount;
    }

    public bool TryWithdraw(decimal amount)
    {
        if (amount <= 0 || amount > Balance) return false;
        Balance -= amount;
        return true;
    }
}
```

此範例只示範物件規則，沒有處理多執行緒或資料庫併發。

**面試回答：**「封裝是把資料與操作規則放在一起，避免外部繞過規則；抽象是讓呼叫者只看到必要能力。像帳戶只能透過存提款改餘額，不直接開放 setter。」

**追問／易錯點：** 全部欄位改成 public property，未必有保護到狀態。回傳內部可變 List，也可能讓外部繞過規則。

## 2. 繼承、組合與多型

**白話與用途：** 繼承表達「是一種」；組合表達「使用某個能力」。多型讓呼叫端使用同一操作，而不同實作呈現不同結果，避免到處判斷具體型別。

**實際場景：** 不同通知管道都提供 `Send`；訂單服務使用通知能力，不必繼承通知器。若只是想重用一小段程式，通常不值得建立很深的繼承樹。

```csharp
public abstract class Notifier
{
    public abstract string Format(string message);
}

public sealed class EmailNotifier : Notifier
{
    public override string Format(string message) => $"電子郵件：{message}";
}

public sealed class SmsNotifier : Notifier
{
    public override string Format(string message) => $"簡訊：{message}";
}
// 多型：透過父型別呼叫，依實際物件執行覆寫。
// Notifier notifier = new SmsNotifier();
// Console.WriteLine(notifier.Format("訂單成立"));
```

**面試回答：**「多型讓同一個操作依實作有不同結果。繼承用在合理的型別關係；如果只是使用或替換一個能力，我會優先考慮組合，避免父類別改動影響所有子類別。」

**追問／易錯點：** 多載是同名方法有不同參數，編譯時依型別等條件選擇；覆寫是子類別改變虛擬方法行為。用 `new` 隱藏方法不等同 `override` 的多型。

## 3. Interface 與抽象類別

**白話與用途：** Interface 像服務合約，約定要提供哪些能力；抽象類別像半成品，除了約定，也能保存共同狀態與實作。它們有助於隔離真正會替換的部分。

| 選擇 | 重點 | 適合場景 |
| --- | --- | --- |
| Interface | 能力契約；類別可實作多個 | 外部付款服務有多個實作或測試替身 |
| 抽象類別 | 共用狀態、建構與行為；類別只能繼承一個類別 | 子類別確實共享同一基礎模型 |
| 具體類別 | 直接表達單一實作 | 沒有替換需求的小型功能 |

**實際場景：** 外部付款服務不適合每次測試都真刷卡，可以用契約隔離。但每個服務都配一個同名 Interface，不一定帶來價值。

**面試回答：**「Interface 著重共同能力，抽象類別可提供共同狀態與實作。我會依替換、測試和共用需求選擇；如果只有簡單單一實作，就直接用類別。」

**追問／易錯點：** 現代 C# Interface 可有預設實作，別說它『永遠不能有方法內容』。差異仍在設計角色與類別繼承限制，無須背所有語法細節。

## 4. SOLID 的實用理解

**白話與用途：** SOLID 是幫助控制修改範圍的設計提醒。重點是讓變更局部、契約可信，並讓難替換的細節不要綁死核心邏輯。

| 原則 | 白話 | 實際例子 |
| --- | --- | --- |
| S 單一職責 | 不同原因的修改不要全擠在一起 | 金額計算不混入寄信與檔案輸出 |
| O 開放封閉 | 常變的部分可擴充，減少改動穩定核心 | 已有多種折扣時，可隔離折扣規則 |
| L 里氏替換 | 替換子型別後，仍遵守原本承諾 | 子類別不能突然拒絕父契約允許的操作 |
| I 介面隔離 | 不逼呼叫者依賴用不到的能力 | 查詢端不必依賴刪除管理功能 |
| D 相依反轉 | 核心政策不要綁死易變的技術細節 | 計價規則不直接依賴特定外部 API |

**實際場景：** 如果每次改寄信都要碰計價，很可能職責混在一起。但只有一條固定折扣規則時，直接寫清楚條件就好，無須先做策略全集。

**面試回答：**「我把 SOLID 當成降低修改連動的提醒，例如把計價和通知分開、讓替換實作維持契約。會在實際變更或測試需要時抽象，不為了符合縮寫就增加很多層。」

**追問／易錯點：** 單一職責不是一個類別只能有一個方法。開放封閉也不代表程式永遠不能修改；是為常見變動建立合理的擴充邊界。
