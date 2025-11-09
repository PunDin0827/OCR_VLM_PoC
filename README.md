# Gemma3 + Paddle OCR：貨物驗收核對系統

一個結合 **Gemma3 多模態大語言模型** 與 **OCR** 的貨物驗收輔助工具。  
協助收貨人員，從 **收貨單影像** 自動整理出結構化 JSON，並與內部採購料表進行 **模糊比對**。

---

## 1. 專案簡介

- 目標：  
  把「收貨單影像 → OCR 輸出 → 結構化 JSON → 採購項目比對」做成一條自動化處理流程，減少人工對帳、抄寫與比對錯誤。

- 典型使用情境：  
  - 事先用 OCR 對收貨單做偵測與辨識，產出：
    - `cell_box.json`：每個欄位／儲存格的位置資訊。
    - `html.json`：對應的文字內容。
  - 將上述 JSON 與收貨單影像 `IMG.jpg` 丟給 Gemma3 VLM，  
    自動輸出包含 **PO 單號、品名、規格、數量、價錢** 的 JSON。  
  - 再把這些品項與內部的模擬資料表 `db_table` 做模糊比對，找出最可能對應的料號。

- 執行環境：  
  - 使用 `llama.cpp` 載入 **本地 Gemma3 VLM (gguf)** 與多模態投影 `mmproj`，  
  - 可在 **無外網** 的環境中執行。

---

## 2. 功能特色

- **VLM + OCR 整合**  
  - OCR 先負責文字偵測與辨識，將版面拆成：
    - `cell_box.json`：欄位座標與表格結構。
    - `html.json`：欄位內文字。  
  - Gemma3 VLM 同時接收：
    - 收貨單影像 `IMG.jpg` 
    - OCR 的 JSON
  - 讓模型在「看圖 + 讀 OCR」的情況下，做出更穩定的欄位對齊與欄位填寫。

- **JSON-only 結構化輸出 (structured JSON output only)**  
  - 在 prompt 中強制要求 **僅輸出 JSON，不允許多餘文字、註解或程式碼區塊**：  
    ```jsonc
    {
      "PO單號": "string or null",
      "品名與規格": [
        {
          "品名": "string or null",
          "規格": "string or null",
          "數量": "string or null",
          "價錢": "string or null"
        }
      ]
    }
    ```
  - 推論完成後，會先用 `re.sub` 去掉可能出現的 ```json / ``` 等 markdown fence，  
    再用 `json.loads` 轉成 Python dict，方便後續程式處理。

- **模糊比對 (fuzzy matching) 對應內部料表**  
  - 使用 `rapidfuzz` 套件中的 `fuzz.ratio` 來計算：
    - 模型預測的 `品名` vs `db_table` 中的品名  
    - 模型預測的 `規格` vs `db_table` 中的規格  
  - 兩者分數取平均，找出每一筆品項在 `db_table` 中 **分數最高的候選 (best match)**，並輸出相似度分數。
  
---

## 3. 系統架構 

```text
[收貨單影像 IMG.jpg]
          │
          │ (PaddleOCR)
          ▼
  +------------------+
  |  OCR 輸出 JSON  |
  |  cell_box.json   |  ← 儲存格座標、表格結構
  |  html.json       |  ← 對應文字內容 
  +------------------+
          │
          │  (input image + OCR JSON)
          ▼
+------------------------------------------+
| - Gemma3 VLM                               |
| - multi-modal reasoning                  |
| - JSON-only prompt                       |
+------------------------------------------+
          │
          │  輸出 structured JSON：
          │  { "PO單號", "品名與規格": [...] }
          ▼
+----------------------+
|  Fuzzy Matching      |
|  (rapidfuzz.fuzz)    |
+----------------------+
          │
          │  比對 db_table (模擬採購料表)
          ▼
[輸出結果]
  - 每一筆品項的最佳匹配料號
  - 相似度分數
  - 可用於後續人工確認或自動對帳

## 4. 後續規劃
 
