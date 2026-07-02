# 📊 我的 SQL 踩坑與優化筆記

記錄學習 SQL 過程中遇到的經典錯誤（Bug）與正確解法的筆記，方便未來撰寫語法時隨時查閱與複習。

## 🔍 目錄

1. [LEFT JOIN 導致的欄位名稱模糊 (Ambiguous column name)](#1-left-join-導致的欄位名稱模糊)
2. [NULL 值的正確篩選方式 (= NULL vs IS NULL)](#2-null-值的正確篩選方式)

---

## 1. LEFT JOIN 導致的欄位名稱模糊

### 💡 遇到情境

當使用 `WITH ... AS` (CTE) 或 `LEFT JOIN` 結合兩個表格，且兩表有相同的欄位名稱（例如 `user_id`）時，若直接在後續的 `SELECT` 呼叫該欄位，會導致 SQL 無法辨識它是哪一個表格的欄位而報錯。

### ❌ 錯誤示範 (Bad Practice)

```sql
WITH combined_table AS (
    SELECT *  -- 錯誤：直接用 * 把兩邊同名欄位都抓進來了
    FROM table_A a
    LEFT JOIN table_B b ON a.user_id = b.user_id
)
SELECT user_id -- 報錯：Ambiguous column name (欄位名稱模糊)
FROM combined_table;
```

### ⭕ 正確示範 (Best Practice)

在第一階段的 `SELECT` 中就明確指定表格來源，並使用 `AS` 重新命名別名以作區隔：

```sql
WITH combined_table AS (
    SELECT
        a.user_id AS a_user_id,   -- 明確指定 table_A 的 user_id
        b.user_id AS b_user_id,   -- 明確指定 table_B 的 user_id
        a.username,
        b.login_time
    FROM table_A a
    LEFT JOIN table_B b ON a.user_id = b.user_id
)
SELECT a_user_id -- 這樣寫 SQL 就能完全辨識，順利執行！
FROM combined_table;
```

---

## 2. NULL 值的正確篩選方式

### 💡 遇到情境

想要在資料庫中，篩選出某個欄位值為空（Missing Value / 缺少資料）的資料列。

### ❌ 錯誤示範 (Bad Practice)

```sql
-- ❌ 這樣寫會找不到任何資料！
-- 因為 NULL 代表「未知」，在 SQL 中不能用等號（=）做比較。
SELECT *
FROM combined_table
WHERE user_id = NULL;
```

### ⭕ 正確示範 (Best Practice)

在 SQL 中必須使用專門的邏輯運算式 `IS NULL` 或 `IS NOT NULL`：

```sql
-- 尋找空值（正確寫法）
SELECT *
FROM combined_table
WHERE user_id IS NULL;
```

```sql
-- 延伸應用：找出在 table_B 沒有購買紀錄的 table_A 會員 (Anti-Join)
SELECT a.user_id, a.username
FROM table_A a
LEFT JOIN table_B b ON a.user_id = b.user_id
WHERE b.user_id IS NULL; -- 篩選出沒出現在 B 表的 A 表會員
```

## 4. GROUP BY 的欄位限制錯誤 (Column must appear in the GROUP BY clause)

### 💡 遇到情境
在 `SELECT` 中同時挑選了多個欄位（例如 `part` 和 `assembly_step`），但在 `GROUP BY` 後面只寫了一個欄位（`GROUP BY part`），導致 PostgreSQL 噴出錯誤：
> `ERROR: column "..." must appear in the GROUP BY clause or be used in an aggregate function`

### 🔍 原因解析
當 SQL 幫你把資料依照 `part` 壓縮合併成同一列時，如果你在 `SELECT` 後面叫它顯示沒有被分組的 `assembly_step`，SQL 會不知道在這一列中，到底該顯示該零件旗下哪一個步驟的值（因為有很多個），因而直接報錯。

### ❌ 錯誤示範 (Bad Practice)
```sql
SELECT part, assembly_step -- 錯誤：assembly_step 沒有在 GROUP BY 中，也沒有被聚合函數包裹
FROM parts_assembly
WHERE finish_date IS NULL
GROUP BY part;
