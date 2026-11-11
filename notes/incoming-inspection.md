# SAP Business One Add-on｜暫收品檢單

這是我依公司收貨流程的需求，構想並開發的 SAP Business One（SAP B1）擴充功能。

## 為什麼做這張單？

假設採購訂了 10 箱材料，倉庫點收也是 10 箱。採購問「數量到齊了嗎？」倉庫可以回答。但生產問「這批料可以用了嗎？」就還要看品檢結果。

在 SAP B1 的採購流程中，採購單記錄訂購內容，採購收貨單記錄收貨內容。我為公司設計「暫收品檢單」，是想讓「貨已到、品質待確認」這段作業有自己的紀錄。

## 設計重點

我把需求拆成兩個問題：倉庫點收了什麼？這次點收的材料，品質確認得如何？即使送來的數量完全正確，仍然值得把第二個問題單獨記下來。

文中的 10 箱為流程示例。

## 從 10 箱到貨，看三段核心程式

以下是我從 Add-on 的 C# 程式挑出的片段。變數改用有意義的名稱，省略表單取值和物件釋放；每段聚焦一個判斷。假設 10 箱來自同一張採購單的同一列，其中 7 箱合格、1 箱不合格、2 箱待判定。

### 1. 帶入採購單時，先確認供應商

一次到貨可能對應好幾張採購單。程式會記住第一張的供應商，之後每帶入一張，就比對一次。

```csharp
if (supplierCode.Length == 0)
{
    supplierCode = purchaseOrder.CardCode;
}
else if (!string.Equals(supplierCode, purchaseOrder.CardCode,
                        StringComparison.OrdinalIgnoreCase))
{
    throw new Exception("多張採購單必須屬於同一供應商。");
}
```

原因在下一步才會顯現：這批資料要轉成一張收貨採購單草稿，草稿表頭只有一個供應商。帶入時先檢查，可以讓使用者在填品檢數量前就發現選錯了單。

### 2. 儲存暫收單時，逐列核對數量

```csharp
double accountedFor = accepted + rejected + pending;
if (Math.Abs(received - accountedFor) > 0.0001)
{
    throw new Exception("請核對這一列的合格、不合格與待判定數量。");
}
```

7 + 1 + 2 剛好對上 10 箱。如果漏填待判定的 2 箱，程式會在儲存時提醒使用者。逐列檢查也有用意：若甲料少填 2 箱、乙料多填 2 箱，整張加總仍然會對。程式用很小的容許差距來比較浮點數。這個判斷只檢查數量加總，沒有判定材料品質。

### 3. 建立收貨草稿時，只帶合格數量，並保留原採購單列

```csharp
// draft 是透過 DI API 建立的草稿物件，目標單據為收貨採購單。
draft.DocObjectCode = Di.BoObjectTypes.oPurchaseDeliveryNotes;
int added = 0;
foreach (var row in inspectionRows)
{
    if (row.Accepted <= 0)
        continue;

    if (added > 0)
        draft.Lines.Add();

    draft.Lines.BaseType = (int)Di.BoObjectTypes.oPurchaseOrders;
    draft.Lines.BaseEntry = row.PurchaseOrderEntry;
    draft.Lines.BaseLine = row.PurchaseOrderLine;
    draft.Lines.Quantity = row.Accepted;
    added++;
}
if (added == 0)
    throw new Exception("沒有合格數量可轉收貨。");

Chk(company, draft.Add());
```

這段用具名欄位示意原程式取出的明細資料。即使同一張採購單有兩列相同料號，單靠料號也分不出這次收的是哪一列。原程式從採購單號查出內部識別碼（BaseEntry），搭配原始列號（BaseLine），讓草稿的 7 箱回到正確的來源列。最後呼叫 Add 儲存的是草稿，正式入庫要等使用者確認後新增。

| 操作情境（示例） | 依上述片段推演的結果 |
| --- | --- |
| 暫收 10 箱，7 合格、1 不合格、2 待判定 | 數量核對通過；轉單時草稿帶入 7 箱 |
| 暫收 10 箱，只填 7 合格、1 不合格 | 儲存暫收單時提示該列數量未對平 |
| 全單只有一列，10 箱都待判定 | 數量核對通過；轉單時提示沒有合格數量 |

儲存暫收單與建立收貨草稿是兩個按鈕。這版程式把數量核對寫在儲存動作裡；若使用者直接按轉單，轉單程式本身沒有重做這項核對。若要讓「數量已對平」成為轉單的必要條件，兩個入口都需要呼叫同一個檢查。
