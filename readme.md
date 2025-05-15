# Invoice OCR & Matching Agent (n8n Workflow)

從使用者上傳發票 PDF／圖片，到自動 OCR、欄位擷取、產品模糊比對，最後輸出標準化 JSON，並在各種邊界條件下發出人工通知或預設回應。

---

## 目錄

1. [特色與功能](#特色與功能)
2. [整體流程](#整體流程)
3. [環境需求](#環境需求)
4. [安裝與設定](#安裝與設定)
5. [主要節點說明](#主要節點說明)
6. [邊界條件處理](#邊界條件處理)
7. [視覺化流程圖](#視覺化流程圖)
8. [測試範例](#測試範例)

---

## 特色與功能

* **多引擎 OCR**：同時支援 Azure Document Intelligence（PDF）與 Google Vision（影像）。
* **欄位擷取**：Regex 提取「品號／品名／單位／數量」。
* **向量檢索**：OpenAI Embeddings + Pinecone Top-1 模糊比對。
* **低分通知**：若 match\_score < 0.7，自動發 Gmail 請人工確認。
* **邊界檢查**：
  * 無檔案 → 通知 `Send empty file message to human`，回 `status: "no_file"`
  * OCR 回傳空白 → 通知 `Send empty content message to human`，回 `status: "no_content"`
  * 完全無比對 → `status: "no_match"` + 空 `items: []`
* **最終輸出**：結合 `customer_name`、`order_date`、`items[]` 與 `status`。

---

## 整體流程

```text
Webhook → RespondToWebhook → check attached file or not
    ├─ False → Send empty file message → Respond { status: "no_file" }
    └─ True → Check If file is pdf
           ├─ PDF → Azure Post → wait for succeeded → Azure Get → Code1
           └─ Image → base64 → Google Vision → If image content is not empty
                         ├─ False → Send empty content message → Respond { status: "no_content" }
                         └─ True → Extracts OCR text → extract product fields
                                   → Embeddings OpenAI → Pinecone Vector Store2
                                   → merge output data with quantity
                                   → If score >= 0.7
                                         ├─ False → Send to human (Gmail)
                                         └─ True → Google Sheets2 → remove duplicates → reconstruct file structure → Merge4 → into items list → Merge customer and items → status (pending/completed) → Image final output
```

---

## 環境需求

* **n8n** v1.19+
* API 金鑰／憑證：

  * Google Vision API
  * Azure Document Intelligence
  * OpenAI API
  * Pinecone
  * Google Sheets OAuth2
  * Gmail OAuth2

---

## 安裝與設定

1. **匯入** `invoice-agent.json` 至 n8n 畫布。
2. 在 Credentials 管理中新增並授權所有 API／OAuth。
3. 啟動 `Webhook` 節點，記下上傳 URL。
4. （首次）執行 `Build Pinecone Vector Store` 節點以建立索引。

---

## 主要節點說明

| 節點名稱                                                                         | 功能說明                                     |
| ---------------------------------------------------------------------------- | ---------------------------------------- |
| **Webhook**                                                                  | 接收發票檔案及表單欄位 (`name`, `date`)             |
| **Respond to Webhook**                                                       | 初步回應所有上傳資料                               |
| **check attached file or not**                                               | 判斷 `$binary` 是否存在                        |
| **Check If file is pdf**                                                     | 根據 MIME type 分流 Azure vs. Google OCR     |
| **Azure Post request** / **Azure Get request1**                              | 呼叫並輪詢 Azure Document Intelligence        |
| **base64** → **google vision ai4**                                           | 圖像轉 base64 → 呼叫 Google Vision OCR        |
| **If image content is not empty**                                            | 判斷 OCR 回傳文字是否非空                          |
| **Extracts OCR text**                                                        | 將 OCR 原始文字分行                             |
| **extract product fields**                                                   | Regex 擷取品名、數量、單位                         |
| **Embeddings OpenAI** → **Pinecone Vector Store2**                           | 生成向量並 Top-1 匹配                           |
| **merge output data with quantity**                                          | 合併原始輸入、匹配結果、數量、match\_score              |
| **If score less than 70% occur human confirm1**                              | 分數門檻檢查：發 Gmail 或進行 Google Sheets 查表      |
| **remove duplicates** → **reconstruct file structure** → **into items list** | 去重、重組 JSON items 陣列                      |
| **Merge customer and items** / **Merge status**                              | 合併客戶欄位 (`name`, `date`)、items 及 `status` |
| **Image final output**                                                       | 輸出最終 JSON / 文件                           |

---

## 邊界條件處理

1. **無檔案**

   * If `Object.keys($binary||{}).length === 0` → Gmail 通知 & 回 `{ status: "no_file", items: [] }`
2. **OCR 回字串空白**

   * If `responses[0].textAnnotations` 為空 → Gmail 通知 & 回 `{ status: "no_content", items: [] }`
3. **完全無比對**

   * 在最終 `into items list` 之後，If `items.length === 0` → Set `{ status: "no_match", items: [] }`

---

## 視覺化流程圖

下面為方框示意，並簡要說明每段流程：

```text
┌────────────────────┐      ┌──────────────────────┐
│    Webhook         │─────▶│ Respond to Webhook   │
└────────────────────┘      └──────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────┐          ┌───────────────────────────┐
                    │ check attached file or  │  False   │ If no file:               │
                    │ not                     │─────────▶│  Send empty file message  │
                    └─────────────────────────┘          │  Respond {no_file}        │
                            │ True                       └───────────────────────────┘
                            ▼                          
                    ┌─────────────────────────┐ 
     ┌────────────┌─│ Check If file is pdf    │
     │            │ └─────────────────────────┘              
     │            │                                          
   PDF?         Image?                                       
     │            │                                          
     ▼            ▼                                          
┌──────────┐    ┌───────────────┐                                   False   ┌──────────────────────┐ 
│ Azure    │    │ base64        │────If image content is not empty ───────▶ │ Send empty           │ 
│ Post→Get │    │ → Google      │                │                          │ content message      │ 
└──────────┘    │ Vision OCR    │                │ True                     │ Respond {no_content} │ 
     │          └───────────────┘                │                          └──────────────────────┘ 
     │                                           ▼                                                   
┌──────────┐                              ┌───────────────────────┐
│ Azure    │                              │ Extracts OCR text     │
│ Get      │◀─────┐                       │ → extract product     │
└──────────┘      │                       │  fields               │
     │            │                       └───────────────────────┘
     └─ wait if Azure still running              │
     │                                           │
     │                                           ▼
     │                                      ┌──────────────────┐    ┌────────────────────────┐ 
     │                                      │ OpenAI Embedding │──▶ │ Pinecone Vector Store2 │
     │                                      └──────────────────┘    │ Fuzzy Matching         │
     │                                                              └────────────────────────┘
     │                                                                        ▼                        
     │                                                              ┌─────────────────────────┐
     │                                          ┌───────────────────│ merge and revise output │
     │                                          │                   │ data with quantity      │
     │                                          ▼                   └─────────────────────────┘
     │             ┌─────────────┐ False┌───────────────────────┐
     │             │Send to human│◀─────│ If item score >= 0.7? │
     │             └─────────────┘      └───────────────────────┘
     │                                          ▼  True   
     │                                     ┌──────────────────────────────────────────────────────────────────┐
     │                                     │ Google Sheets2 (use "品名" and "單位" to retreive "品號" and "幣別")│
     │                                     └──────────────────────────────────────────────────────────────────┘
     │                                                  │                                          
     │                                                  ▼                                          
     │                                  ┌───────────────────────────────────────────────────┐
     │                                  │ remove duplicates → reconstruct file structure    │
     │                                  │ → merge matched name and original input           │
     │                                  │ → reconstruct file structure → merge quantity     │
     │                                  │ → transform data structure into items list        │
     │                                  └───────────────────────────────────────────────────┘
     │                                          │                                          
     │                                          ▼                                          
     │                                  ┌───────────────────────┐
     │─────────────────────────────────▶│ Merge customer &      │
                                        │ items (name/date)     │
                                        └───────────────────────┘
                                                   │             
                                                   ▼             
                                        ┌────────────────────┐   
                                        │  PDF/Image final   │
                                        │    output          │   
                                        └────────────────────┘
```

---

## 測試範例

```
前往 https://docs.google.com/forms/d/e/1FAIpQLSe5SNvI37fQbJJ4v7ipUHbgNc7DGohK3-pKgq3GJTRrLHISWQ/viewform 
填寫姓名，日期，及上傳檔案
```

**成功結果**

```json
{
  "name": "Test",
  "date": "2025-05-09",
  "items": [
    {
    "product_id": "S023200",
    "matched_name": "熟花生 斤",
    "original_input": "熟花生3斤",
    "match_score": 0.996118307,
    "quantity": 3
    },
    {
    "product_id": "F009080",
    "matched_name": "海帶絲 斤",
    "original_input": "海帶絲 3斤",
    "match_score": 0.999877214,
    "quantity": 3
    }
  ],
  "status": "pending"
}

```

