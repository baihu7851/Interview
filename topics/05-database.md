# 05｜資料庫與 SQL

[回目錄](../README.md) · [快速複習](../quick-review.md)

## 1. 關聯、主鍵、外鍵與正規化

**白話與用途：** 客戶和訂單像兩份名冊，用客戶編號建立關係，不必每張訂單重複所有客戶資訊。主鍵識別一筆資料；外鍵限制參照的資料存在；唯一約束防止重複，NOT NULL 限制必填。

正規化是把不同事實放在適合的地方，減少重複與更新矛盾。先掌握「一個事實別到處存」即可，不必背所有正規形式證明。

**實際場景：** `customers(id, name)`、`orders(id, customer_id, total, status, created_at)` 是一對多；訂單與商品透過 `order_items(order_id, product_id, quantity, unit_price)` 表示多對多。明細的成交單價是歷史事實，應保存，不可只讀商品現在的售價。

```sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    total DECIMAL(12, 2) NOT NULL CHECK (total >= 0),
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

以上只定義後續讀取範例用到的兩張表，不建立實際資料庫；帳戶、商品等範例的欄位另以文字說明。

**面試回答：**「主鍵識別資料，外鍵維持關聯，唯一與必填約束守住基本規則。正規化減少同一事實重複保存，但像成交價格這種歷史快照，是不同事實，應保留。」

**追問／易錯點：** 只在程式先查重再新增，併發時仍可能重複，資料庫應有唯一約束。外鍵不代表所有資料庫都自動建立外鍵欄位索引。反正規化能省查詢，但要承擔同步成本。

## 2. JOIN、GROUP BY、WHERE 與 HAVING

**白話與用途：** JOIN 把相關名冊接起來；GROUP BY 把同類資料放一堆再統計。WHERE 先篩選資料列，HAVING 再篩選分組結果，能讓資料庫完成篩選與計算，減少搬回程式處理。

**實際場景：** 列出所有客戶及其已付款訂單總額，包括沒訂單的人。

```sql
SELECT c.id, c.name,
       COUNT(o.id) AS paid_order_count,
       COALESCE(SUM(o.total), 0) AS paid_total
FROM customers AS c
LEFT JOIN orders AS o
  ON o.customer_id = c.id AND o.status = 'paid'
GROUP BY c.id, c.name
ORDER BY paid_total DESC, c.id;
```

要只保留總額至少 1000 的客戶，在 GROUP BY 後、ORDER BY 前加入 `HAVING SUM(o.total) >= 1000`。

**面試回答：**「INNER JOIN 只留配對成功的資料，LEFT JOIN 保留左邊全部，右邊沒配到會是 NULL。WHERE 篩資料列，HAVING 篩分組後的結果，例如訂單總額達標的客戶。」

**追問／易錯點：** 右表的條件若放進 WHERE，可能把 LEFT JOIN 沒配到的資料排掉。`COUNT(*)` 計所有結果列，`COUNT(o.id)` 不計 NULL；一對多 JOIN 會增加列數，不小心再 JOIN 明細會讓訂單總額重複累加。

## 3. NULL 與三值邏輯

**白話與用途：** NULL 像「還不知道」，不是數值 0，也不是空字串。拿未知值做一般比較，結果通常也是未知；SQL WHERE 只保留結果為真的資料列。

**實際場景：** 欄位尚未填寫要用 `IS NULL`。假設資料中 `cancelled_at` 可空：

```sql
SELECT id FROM orders WHERE cancelled_at IS NULL;
-- 這段假設另外有 cancelled_at 欄位，前面的最小示範表未定義它。
```

**面試回答：**「NULL 代表沒有或未知的值，要用 IS NULL 判斷。一般比較碰到 NULL 不會直接得到 true 或 false，所以統計和排除條件要特別留意。」

**追問／易錯點：** `= NULL` 不正確。`NOT IN` 的子查詢若包含 NULL，可能讓原本預期保留的資料也被排除；此時可評估 `NOT EXISTS`。`SUM` 通常忽略 NULL；沒有可加總值時可能回傳 NULL，可用 COALESCE 明確補值。

## 4. 索引與慢查詢

**白話與用途：** 索引像書的目錄，讓資料庫少翻一些頁。常見 B-tree 索引有順序，適合等值、範圍與部分排序需求。代價是空間，以及新增、修改、刪除時維護索引的工作。

**實際場景：** 經常查某位客戶在一段時間的訂單，可評估複合索引：

```sql
CREATE INDEX ix_orders_customer_created
ON orders (customer_id, created_at);

SELECT id, total, created_at
FROM orders
WHERE customer_id = 42
  AND created_at >= TIMESTAMP '2026-01-01 00:00:00'
ORDER BY created_at, id;
```

**面試回答：**「索引用空間和寫入成本換查找效率。我會根據 WHERE、JOIN 和 ORDER BY 設計，再看執行計畫與實際耗時確認，不能只看『有索引』就認定會快。」

**追問／易錯點：** 複合 B-tree 通常在前導欄位有條件時更有效；不能把『沒有最左欄就絕對不能用』當通則，實際看資料庫與計畫。若查出大半張表，全表掃描可能更省。欄位套函式、型別轉換與前綴萬用字元可能降低一般索引效益。[PostgreSQL：複合索引](https://www.postgresql.org/docs/current/indexes-multicolumn.html)

慢查詢先看時間花在等待鎖、執行 SQL 還是傳輸；再看掃描列數、索引、JOIN、排序和回傳欄位。PostgreSQL 的 `EXPLAIN` 看估計計畫，`EXPLAIN ANALYZE` 會真的執行，含寫入的 SQL 必須特別小心。ORM 要檢查實際 SQL，避免 N+1：先查一份清單，再每筆各查一次關聯。

## 5. 交易與 ACID

**白話與用途：** 轉帳不能只扣款、不入帳。交易把多步資料庫操作包成一個單位：成功一起提交，失敗回復交易內可回復的修改。

| 特性 | 白話 |
| --- | --- |
| A 原子性 | 全部成功或全部不生效 |
| C 一致性 | 配合約束與正確邏輯，維持合法資料規則 |
| I 隔離性 | 控制同時進行的交易彼此能看到什麼、如何影響 |
| D 持久性 | 成功提交的資料依資料庫保證持續保存 |

**實際場景：** 假設 `accounts(id, balance)` 存在，轉帳可先做有條件扣款，再入帳：

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100
WHERE id = 1 AND balance >= 100;
-- 呼叫端檢查受影響列數：必須等於 1，否則 ROLLBACK 並停止。
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- 入帳也必須影響 1 列；否則 ROLLBACK。全部成功才 COMMIT。
COMMIT;
```

這是交易步驟示意，不是可忽略註解直接使用的完整轉帳實作；另需規劃帳戶更新順序、金額驗證與重送處理。

**面試回答：**「交易讓多步操作一起成功或失敗，ACID 是原子性、一致性、隔離性與持久性。它保障資料操作，但商業規則仍要正確寫，外部寄信或付款也不會自動跟著回滾。」

**追問／易錯點：** 交易不要長時間包住外部 HTTP 呼叫；會延長鎖與連線占用。Commit 後的外部副作用要另設計可靠處理，見 [後端可靠性](08-backend.md)。

## 6. 隔離層級與避免丟失更新

**白話與用途：** 多人同時編輯同一份文件，需要約定彼此看得到哪個版本。隔離層級控制交易互相干擾的程度；更強的保證也可能帶來等待或交易失敗重試。

**實際場景：** 常見讀取問題與 SQL 標準最低保證：

| 層級 | 重點 |
| --- | --- |
| Read Uncommitted | 標準允許髒讀：看到別人未提交的值 |
| Read Committed | 不讀其他交易未提交值；重讀仍可能不同 |
| Repeatable Read | 避免髒讀與不可重複讀；標準仍允許幻讀 |
| Serializable | 結果等同某種逐一執行順序，不表示實際完全排隊 |

不可重複讀是同一筆資料重讀變了；幻讀是同樣條件查出的資料集合變了。各資料庫實作不同：PostgreSQL 的 Read Uncommitted 等同 Read Committed，Repeatable Read 也防止幻讀，但仍可能有序列化異常。[PostgreSQL：交易隔離](https://www.postgresql.org/docs/current/transaction-iso.html)

庫存常用單次條件更新，避免先讀後寫：

```sql
-- 假設 products(id, stock)，購買一件。
UPDATE products SET stock = stock - 1
WHERE id = 10 AND stock >= 1;
-- 影響 1 列才代表成功；0 列表示不存在或庫存不足。
```

另一種是樂觀併發控制：讀取 `version`，更新時 `WHERE id = ... AND version = 舊值`，並遞增版本；0 列表示有人先改了。悲觀方式則先鎖住要修改的資料，讓其他寫入者等待。

**面試回答：**「隔離層級決定交易能受到哪些併發影響。有交易仍可能丟失更新；單筆庫存可用條件更新，多步規則再選適當隔離或鎖。樂觀控制是更新時檢查版本，悲觀控制是先鎖住再改。」

**追問／易錯點：** 避免超賣與避免同一訂單重扣是不同問題；後者還需唯一業務鍵或冪等處理。Serializable 也可能中止衝突交易，應重試整個交易且控制次數。資料庫鎖與 C# lock 不是同一層的保護。
