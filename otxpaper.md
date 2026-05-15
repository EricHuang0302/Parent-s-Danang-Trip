# 論文重點

## 一句話看懂這篇 paper

這篇 paper 不直接判斷「單筆 CTI 是不是真」，而是把多個 CTI source 放進同一個 `world view`，持續計算 `10 個品質參數`，再用加權與時間衰減算出每個 source 在時間 `t` 的 `trust indicator`。

## 這篇 paper 想解決什麼問題

- CTI source 很多，但品質會隨時間變動；只靠一次性評估或人工印象，無法穩定判斷哪個 source 比較可信。
- 作者要做的是 `持續、量化、可自動化` 的 source quality evaluation。
- 論文的輸出是「這個 source 目前有多值得信任」，不是「這一筆 IOC 一定是真的」。

## 核心方法：先算參數，再算 trust indicator

`Paper:` 採用 `closed world assumption`，假設一組預先選定的 CTI sources 所提供的資料，足以構成這個評估場景中的完整 threat intelligence 世界。  
`人話：` 它不是跟全世界比，而是先框出一個比較集合，所有 source 都在這個集合裡互相比。

`Paper:` 這個比較集合會形成 `world view`；每收到一筆新情資，就先對該訊息做參數評估，再在固定時間點更新該 source 的 trust indicator。  
`人話：` 先評「這筆資料帶來什麼品質訊號」，再評「這個來源現在值不值得信」。

`Paper:` 方法以 `STIX 2.x` 當代表性的 threat intelligence sharing standard，但其他格式也可以先轉成 STIX 再納入。  
`人話：` STIX 是主要資料模型，不代表只能處理 STIX feed。

`人話版流程：` `收到新情資 -> 更新 world view -> 重算該 source 的參數 -> 到指定時間點更新 trust indicator`

## 10 個指標怎麼看

| 參數 | 在看什麼 | 大致怎麼算 | 高分代表什麼 |
| --- | --- | --- | --- |
| `p1 Extensiveness` | source 願不願意把非必填欄位補齊 | 對每則 message 計算 `已填 optional 欄位 / 該類型最大 optional 欄位`，再對該 source 取平均 | 記錄更完整、描述更細 |
| `p2 Maintenance` | source 會不會持續更新舊資料 | 看該 source 的更新物件數，和 `world view` 內其他 source 的平均更新程度相比，再正規化到 `[0,1]` | 會持續修正或補充既有情資 |
| `p3 False Positives` | source 的資料有多常被撤銷或標成無效 | 用該 source 的 false positives 數量，去對比整個 `world view` 的 false positives 總量 | 錯誤或被撤銷的資料相對較少 |
| `p4 Verifiability` | source 有沒有附上可外部查證的依據 | 觀察 `external_references` 等引用數量，相對於 `world view` 平均值後再正規化 | 資料更容易追溯、驗證 |
| `p5 Intelligence` | source 有沒有提供 IOC 以外的附加價值 | 看訊息和其他物件的關聯數量，例如 `related-to`、`derived-from` 等，再和 `world view` 平均比較 | 不只丟指標，還有脈絡、分析或關聯 |
| `p6 Interoperability` | source 的資料格式是否容易被工具消化 | 依 source 採用的格式在 `world view` 裡的普及度與排名估分，越主流標準分數越高 | 格式更通用、較容易被平台整合 |
| `p7 Compliance` | source 的資料是否真的符合它聲稱使用的標準 | 用 validator 檢查訊息是否完全符合標準，再相對於 `world view` 平均值正規化 | 資料品質較穩、結構較標準化 |
| `p8 Similarity` | source 對同一事件的描述和其他 source 有多像 | 先比對 `id` 或 `type`，再用 Jaccard / Cosine 類似度等方法計算相似度後取平均 | 和其他 source 的同事件敘述較一致 |
| `p9 Timeliness` | source 是不是比較早釋出同一事件的情報 | 對每個可比對事件，計算 `最早出現時間 / 該 source 發布時間`，再取平均 | 越接近第一時間發布 |
| `p10 Completeness` | 單一 source 覆蓋整個 `world view` 的程度 | 看 `world view` 中有多少物件能被該 source 對應或涵蓋，公式為 `(|B| - |A|) / |B|` | 只看這個 source 也能涵蓋較多世界資訊 |

補充：paper 將各參數輸出限制在 `[0,1]`；其中部分參數是先和 `world view` 平均值比較，再做正規化。

## Trust Indicator 怎麼算

論文把最終信任值寫成加權平均加上時間衰減：

```text
TI_sx(t) = (D * TI_sx(t - 1) + Σ(n=1..m) ω_n * (p_n)_sx(t)) / (D + Σ(n=1..m) ω_n)
```

各符號的意思：

- `(p_n)_sx(t)`：source `sx` 在時間 `t` 的第 `n` 個品質參數值。
- `ω_n`：第 `n` 個參數的權重；不同 use case 可以自己調，像是有些場景更重視 `timeliness`，有些更重視 `false positives`。
- `D`：ageing factor，保留過去信任值，但會讓舊證據權重低於新證據。
- `t`：重新計算 trust indicator 的時間粒度，例如每小時、每天、每週或每月。

這條式子的人話版意思：

- 新的品質證據不會直接覆蓋舊信任值，而是和上一期 `TI` 一起做加權平均。
- `D` 越大，模型越保守，較不容易因為短期波動劇烈改變評分。
- `ω_n` 決定哪些參數在你的場景裡更重要。
- 論文也特別提到，只有在某個時間框架內 `Σ ω_n * p_n` 真的變動時，才需要重新觸發 trust indicator 計算。

## 論文中的計算例子

paper 用 `p1 Extensiveness` 示範怎麼算。

資料樣本來自 3 筆 STIX messages，分別是 MISP event ID `24504`、`21271`、`24362`。三筆資料對應的 optional 欄位填寫比例為：

- 第 1 筆：`10 / 47`
- 第 2 筆：`22 / 193`
- 第 3 筆：`54 / 172`

因此：

```text
p1 = (1 / 3) * ((10 / 47) + (22 / 193) + (54 / 172)) ≈ 0.21
```

這個 `0.21` 的意思不是「這個 source 很差」，而是說：  
在這個例子裡，該 source 平均每則 message 只填了約 `21%` 的可選欄位。也就是說，它有提供基本資訊，但在額外細節的補充上並不算特別完整。

## 這篇 paper 的價值與限制

這篇 paper 最有價值的地方，在於它把 CTI source 的評估從「主觀印象」拉成「可持續更新的量化流程」。只要有新情資進來，系統就可以持續修正各 source 的信任值，這很適合自動化情資平台。

但它有幾個很重要的限制：

- 這是一套 `相對評分`，非常依賴 `world view` 內到底放了哪些 source。
- `closed world assumption` 很強，代表系統預設「我目前納入的 sources 已足夠代表這個世界」；這在真實環境不一定成立。
- `p8 Similarity`、`p9 Timeliness`、`p10 Completeness` 的效果，很吃比對規則與相似度演算法怎麼設。
- `p3 False Positives` 的假設也不是絕對的；一個常主動撤銷錯誤訊息的 source，不見得真的比較差。
- 它衡量的是 `source quality / trust`，不是直接證明某筆 IOC 的真偽。

## 參考資料

- [A Quantitative Evaluation of Trust in the Quality of Cyber Threat Intelligence Sources - PDF](https://services.phaidra.univie.ac.at/api/object/o%3A1076811/download)
- [University of Vienna metadata page](https://ucrisportal.univie.ac.at/en/publications/a-quantitative-evaluation-of-trust-in-the-quality-of-cyber-threat/)
