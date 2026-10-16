---
name: keygon-quote-system
description: 使用 Des Sir 的報價系統（quotegen）製作 KEYGON Proforma Invoice（PI）時使用。適用於：根據客戶需求產生報價 PDF、預覽報價、匯入或更新價格邏輯表、以及寄出前的品質檢查。本文件只描述流程，不含任何價格、利潤、成本或客戶資料。
---

# KEYGON 報價系統（Proforma Invoice）

本 Skill 說明如何用 Des Sir 的報價系統產生 KEYGON Proforma Invoice。

> 本 repo 為公開 repo：這裡**只記錄流程**。價格、利潤率、等級（Tier）百分比、成本、客戶資料、銀行資料及產品資料一律存放於私有系統 repo，**不得**複製到此處或任何公開位置。

## 1. 系統位置與環境需求

- 系統位置：私有 Cursor Origin repo
  https://cursor.com/codebase/93-1/quote-system-demo-v1 （branch `main`）
- 環境需求：
  - Python 依賴：

    ```bash
    pip install -r requirements.txt
    ```

  - LibreOffice（用於產生 PDF）
  - poppler（用於 PNG 預覽及 `pdftotext` 檢查）

## 2. 製作報價

```bash
python -m quotegen quote request.xlsx      # 也可使用 request.yaml / request.json
python -m quotegen quote request.xlsx --preview   # 另外輸出 PNG 預覽
```

輸出（位於 `output/`）：

- 客戶版 PDF（Proforma Invoice）
- 內部明細 `*_internal.md`（含計算過程，僅供內部使用）

### Request 檔案格式

- `request.xlsx` 使用 KEYGON request 版面。
- **Tier** 與 **Scenario** 寫在 **Order Remarks** 欄，例如：

  ```text
  Tier: 終端(小); Scenario: parking-free-1lane-1car; Lanes: 1
  ```

- 以平方米（m²）或米計價的品項：
  - `QTY` 填總平方米數或總米數；或
  - 在 Description 寫尺寸，例如 `18 m x 1.6 m`，由系統計算。
- 找不到的型號會直接報錯並停止——**絕對不要自行編造價格**，請先把資料補進價格邏輯表。
- 必要的配套品項會依邏輯表中的 Scenario 規則**自動加入**，無需手動填寫。

## 3. 新增 / 更新資料

```bash
python -m quotegen ingest path/to/pricing_logic.xlsx            # 匯入價格邏輯表
python -m quotegen ingest path/to/pricing_logic.xlsx --sheet NAME   # 指定工作表
python -m quotegen list                                         # 檢查匯入結果
python -m pytest                                                # 執行測試
```

測試通過後，將 `data/` 的變更 commit 到私有系統 repo。

## 4. 規則

- 所有價格以 **USD** 計，**不做匯率換算**。
- 售價公式：`售價 = 成本欄 ÷ (1 − 等級利潤率) ÷ 0.87`
  （此處只記公式；實際數值僅存在私有 repo。）
- m² 品項：
  - 數量 = 總平方米數
  - 單價取小數點後 2 位
  - 行總額 = 數量 × 單價
  - 金額使用千位分隔符號
- 客戶版 PDF **永遠不得**顯示成本、利潤率、Tier 或公式。
- 只交付 **PDF** 給客戶（不交付 xlsx、md 等其他檔案）。
- **絕不自動寄送 email**——一律由人工寄出。
- 寄出前必須檢查：
  - 內部明細 `*_internal.md` 中所有標示「待確認」的項目
  - `docs/questions_for_owner.md` 中尚未解決的問題

## 5. QA 檢查清單（寄出前）

- [ ] 打開 PDF，確認版面正常。
- [ ] 確認產品照片與規格正確、對應正確品項。
- [ ] 核對各行數量、單價、行總額及總計。
- [ ] 確認沒有洩漏內部資訊：

  ```bash
  pdftotext output/<報價檔>.pdf - | grep -niE "cost|margin|tier|成本|利潤|等級|÷|0\.87"
  ```

  預期**沒有任何輸出**；若有輸出，不得寄出，先修正。
- [ ] 已處理所有「待確認」項目及 `docs/questions_for_owner.md` 的問題。
- [ ] 由人工寄出 PDF。
