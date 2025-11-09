# Gemma3 VLM for OCR + Visual Inspection（看圖驗貨系統）

> 使用 **Gemma3 Vision-Language Model (VLM)** 結合 **OCR + 圖片輸入**，協助收貨/倉儲人員「看圖驗貨」，自動比對收貨單與實際貨品狀態。  
> *Gemma3 VLM + OCR pipeline to verify delivery notes against real product images.*

> 🛠 **Project status：Work In Progress (WIP)** — 目前仍在開發中，API 介面與模組設計可能會調整。  
> *This project is under active development; APIs and module design may change.*

---

## 1. 專案簡介 (Overview)

本專案目標是從「收貨單影像 + 貨品實拍照片」自動產生可用於驗收的結構化資訊，並將結果與專案採購清單進行比對，協助人員快速發現以下問題：

- 收貨單上的品名、數量、序號與實際貨物不一致  
- 收貨單上「有列但沒收到」或「收到但沒列出」的項目  
- 單據上資訊模糊/手寫/拍攝角度不佳時，提供 VLM 輔助判讀

*The system transforms delivery note images and product photos into structured data, then compares them against purchase records to highlight mismatches and potential risks.*

---

## 2. 核心功能 (Core Features)

> ✅：已規劃並有初步實作　　☐：尚未完成 / 計畫中

### 2.1 OCR 結構化欄位萃取 (Structured OCR Extraction)

- ✅ 從收貨單影像中萃取：  
  - 品名（item name / description）  
  - 數量（quantity）  
  - 序號 / 批號（serial / lot number）  
  - 日期（date / delivery date）  
- ✅ 轉為結構化 JSON / 表格格式，方便後續比對。  
  *Extract item name, quantity, serial/lot number and date from delivery note images into structured JSON form.*

### 2.2 VLM 圖文交叉驗證 (Cross-check with VLM: Image + Text)

- ✅ 將 **收貨單影像 + OCR 文本 + 貨品照片** 一併輸入 Gemma3 VLM。  
- ✅ 請 VLM 回答：「實際貨品是否與收貨單描述相符？是否有缺漏或多收？」  
- ✅ 透過 prompt 設計降低 hallucination，要求模型務必標註「不確定」情況。  
  *Feed both the delivery note and product images plus OCR text into the VLM to verify consistency, with prompt design to reduce hallucinations.*

### 2.3 與採購清單的模糊比對 (Fuzzy Matching with Purchase List)

- ✅ 將 OCR 產生的結構化欄位與資料庫中的採購明細做比對。  
- ✅ 使用字串相似度 / token-based similarity 做 **模糊匹配 (fuzzy matching)**。  
  - 例如：Levenshtein distance、Jaro-Winkler、或 embedding-based similarity。  
- ✅ 為每一列產生匹配分數（0.0–1.0），標示：  
  - 高風險不一致（紅色標註）  
  - 可疑 / 需人工確認（黃色）  
  - 正常匹配（綠色）  
  *Compare extracted fields against purchase records using fuzzy string or embedding-based similarity, and compute a risk score per line item.*

### 2.4 看圖驗貨輔助 (Visual Inspection Assistant)

- ✅ 從貨品影像中抽取：型號標籤、規格文字、外觀特徵等。  
- ✅ 搭配收貨單 OCR 結果，一併輸入 VLM，請模型回答：  
  - 「圖片中顯示的貨品是否為 XXX 型號？有看到幾個？是否包含附件/配件？」  
- ☐ 規劃加入「多張圖整合」：同一批貨多張照片交由 VLM 做整體判斷。  
  *Let the VLM inspect product photos and verify whether they match the ordered model/spec and quantity.*

---

## 3. 系統架構 (System Architecture)

```text
[收貨單影像]       [貨品實拍影像]
      │                │
      │                ├────────────┐
      │                             │
[OCR 模組]                         │
(文字偵測 + 辨識)                   │
      │                             │
      ↓                             │
[結構化欄位抽取]                    │
(item_name, quantity, SN, date, ...)│
      │                             │
      ├──────────────┐              │
      │              │              │
      ↓              ↓              ↓
[採購清單資料庫]  [VLM：Gemma3 + 圖片 + OCR 文本]
      │              │
      │              └──→ 圖文一致性判斷 (consistency check)
      ↓
[模糊匹配 / 風險評分引擎]
      ↓
[驗收報表 / Dashboard 輸出]
(高風險項目標記、建議人工覆核)
