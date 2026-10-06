# 07｜常見資安觀念

[回目錄](../README.md) · [快速複習](../quick-review.md)

## 1. 身分驗證、授權與最小權限

**白話與用途：** 身分驗證是確認「你是誰」，授權是決定「你能做什麼」。進了公司大門，不代表可以打開每個人的抽屜。最小權限只給完成工作必要的能力，減少誤用與外洩範圍。

**實際場景：** 使用者已登入，但把 `/orders/100` 改成 `/orders/101` 可能讀到別人的訂單。每次操作都要驗證資料所屬與權限，不能只檢查登入狀態或前端是否顯示按鈕。

```sql
-- @current_user_id 必須來自伺服器驗證過的身分，不相信客戶端自報。
SELECT id, total FROM orders
WHERE id = @order_id AND customer_id = @current_user_id;
```

此處 `@...` 是參數化表示法，實際符號依資料庫驅動而異；管理員等不同政策需另行明確判斷。

**面試回答：**「驗證確認身分，授權確認可以做什麼。後端每次都要檢查操作與資料範圍，例如訂單是否屬於目前使用者；前端隱藏按鈕不能代替後端權限檢查。」

**追問／易錯點：** 難猜的 UUID 只能降低猜中機率，不能代替授權。請求 DTO 只接受可編輯欄位，避免使用者偷偷送 `IsAdmin` 或伺服器計價欄位。

## 2. SQL Injection 與參數化

**白話與用途：** 資料應該填進表單格子，不能讓填表的人改寫整張指令。SQL Injection 是把外部輸入拼進 SQL，讓資料被當成語法；參數化把 SQL 結構和資料分開。

**實際場景：** 登入、搜尋、篩選 API 都可能接收惡意字串。以下 C# 假設傳入已開啟的 `DbConnection`：

```csharp
static async Task<object?> FindNameAsync(
    System.Data.Common.DbConnection connection, int id,
    CancellationToken token)
{
    using var command = connection.CreateCommand();
    command.CommandText = "SELECT name FROM customers WHERE id = @id";
    var parameter = command.CreateParameter();
    parameter.ParameterName = "@id";
    parameter.DbType = System.Data.DbType.Int32;
    parameter.Value = id;
    command.Parameters.Add(parameter);
    return await command.ExecuteScalarAsync(token);
}
```

驅動必須支援此命名參數格式；不要照抄到使用不同參數規則的 provider。

**面試回答：**「SQL Injection 是輸入被解讀成 SQL 指令。我會用參數化查詢，讓輸入只能作為資料，並限制資料庫帳號權限；不是只把單引號替換掉。」

**追問／易錯點：** 表名、欄名與排序方向通常不能用一般值參數代入，應對應固定允許清單。ORM 也可能因拼接 raw SQL 而出現漏洞。[OWASP：SQL Injection 防護](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

## 3. XSS 與 CSRF

**白話與用途：** XSS 是把惡意程式混進網站內容，讓它在使用者瀏覽器中以該網站的權限執行。CSRF 是利用瀏覽器可能自動附帶登入憑證，誘使使用者對另一個網站送出非本意的操作。兩者需要不同防護。

**實際場景：** 留言內容未編碼就當 HTML 插入，可能造成 XSS。使用 Cookie 驗證的轉帳 API 若沒有防護，可能被惡意網站誘導提交請求。

| 問題 | 常見防護 |
| --- | --- |
| XSS | 按輸出情境編碼、避免不受信任 HTML；需接受 HTML 時用成熟 sanitizer；CSP 作額外防線 |
| CSRF | CSRF token、合適 SameSite、驗證來源等；有副作用的操作不放在 GET |

**面試回答：**「XSS 是惡意程式在我們網站的瀏覽器環境執行，重點是安全輸出；CSRF 是借用自動帶上的登入憑證送操作，重點是確認請求來自合法流程，例如驗證 CSRF token。」

**追問／易錯點：** HttpOnly 降低直接讀取 Cookie 的風險，但 XSS 仍可能以登入者身分送請求。Authorization header 不會像 Cookie 自動附加，可降低典型 CSRF 風險，但 token 被 XSS 竊取仍危險；CORS 不能代替授權或完整 CSRF 防護。[OWASP：XSS 防護](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)、[CSRF 防護](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)

## 4. Hash、加密與密碼保存

**白話與用途：** 加密像上鎖，持有正確金鑰可以解開；Hash 像指紋，用來比對而非還原。登入只需要驗證輸入是否正確，不需要把密碼解回來，因此通常保存專用密碼雜湊結果。

**實際場景：** 密碼採用 Argon2id、bcrypt 或合適設定的 PBKDF2 等成熟方案，搭配每筆獨立 salt 與成本參數，增加離線猜密碼的成本。使用框架或成熟函式庫，不自行寫演算法。

**面試回答：**「加密可以用金鑰還原，Hash 用來比對。密碼要用有 salt、可調成本的專用雜湊，不保存明文，也不只做一次 SHA-256；salt 不必保密，但能讓相同密碼得到不同結果。」

**追問／易錯點：** Hash 不代表不會被猜中，弱密碼仍能被暴力猜測。一般雜湊、密碼雜湊與簽章用途不同；.NET `GetHashCode()` 不能拿來存密碼。[OWASP：密碼保存](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)

## 5. 輸入驗證、機密與日誌

**白話與用途：** 所有外部資料都像陌生人交來的表單，要確認格式、範圍和是否可用。驗證能維持商業規則與資源上限，但不能代替 SQL 參數化或 HTML 安全輸出。

**實際場景：** 限制上傳大小、允許的檔案種類、分頁筆數；伺服器重新計算價格，不相信客戶端金額。密碼、token、連線字串不進 Git，也不完整寫進日誌；使用受控的設定或機密管理，外洩後要撤銷／輪替。

**面試回答：**「後端會檢查外部資料的格式、範圍和權限，不能只靠前端。機密不放原始碼和日誌；除錯只記必要資訊，外洩就輪替，刪掉檔案不代表歷史已消失。」

**追問／易錯點：** 伺服器若替使用者抓指定 URL，要防止存取內部服務等 SSRF 風險。檔案路徑不能直接由輸入任意拼接。資安措施要針對實際入口與信任邊界，而不是只加一個全域正規表示式。
