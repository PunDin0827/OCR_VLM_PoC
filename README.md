# Gemma3 + Paddle OCR：貨物驗收核對系統  

使用 Gemma3 VLM 結合 OCR 輸出，  
將收貨單影像整理成結構化 JSON，並用模糊比對（fuzzy matching）對應到實際採購料表。

---

## 1. 專案簡介
- 目標

## 2. 目前功能

### 2.1 VLM + OCR → 結構化 JSON

Notebook `Gemma3.ipynb` 內主要流程：

- 載入 OCR JSON：
  - `cell_box.json`：欄位座標、表格結構。
  - `html.json`：欄位文字內容。
- 建立 prompt，要求 Gemma3 只輸出 JSON，格式大致如下（示意）：

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
