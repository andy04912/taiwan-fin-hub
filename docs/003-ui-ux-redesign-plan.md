# Taiwan Fin Hub UI/UX 重構規畫

## 設計判讀

這是一個供個人長期使用的金融整合 dashboard。介面必須優先傳達可信、清楚、快速掃讀與資料狀態，而不是行銷感或視覺炫技。

- 重構模式：保留既有品牌與資訊架構的 targeted evolution
- `DESIGN_VARIANCE`：4，使用清楚網格與少量非對稱層級
- `MOTION_INTENSITY`：3，只使用狀態轉換、hover、press 與必要的內容進場
- `VISUAL_DENSITY`：8，維持高資訊密度，但減少不必要卡片與裝飾
- 技術策略：保留 Svelte 5、Tailwind CSS 4、bits-ui 與既有 query 架構
- 不變項目：route hash、主導覽名稱、API、表單欄位順序、資料語意與金融計算

## 現況稽核

### 現有品牌與介面語言

| 項目     | 現況                                                  |
| -------- | ----------------------------------------------------- |
| 畫布     | `paper #f7f7f2`，偏暖灰白                             |
| 主要文字 | `ink #1f2933`，深藍灰                                 |
| 品牌色   | `steel #3e6f7c`，搭配 moss 與 coral                   |
| 字體     | Inter、system UI，繁中實際依賴系統 fallback           |
| 圓角     | 多數為 12px，按鈕混用 6px、8px 與 full pill           |
| 表面     | 白底、1px 邊框、`shadow-xs` 幾乎套用在所有區塊        |
| 導覽     | 桌機 240px 深色側欄，平板頂部導覽，手機底部導覽       |
| 資料顯示 | 大量 tabular number、KPI 卡片、折線圖、甜甜圈圖與列表 |

### 應保留的部分

- 五個主要區域已符合使用者心智模型：總覽、資產、活動、發票、設定。
- 資產細項與手機設定子頁已有返回流程，不需要重做 route。
- 隱藏金額、PWA standalone 捲動與 swipe back 都是有價值的既有互動。
- 收入、支出、資產、負債已有穩定的語意色彩。
- 目前桌機與手機的內容順序一致，學習成本低。
- 多數控制項已具備 40px 到 44px 的可點擊尺寸與 focus ring。

### 需要淘汰或改善的模式

| 問題                             | 影響                                 | 改善方向                                            |
| -------------------------------- | ------------------------------------ | --------------------------------------------------- |
| Inter 與繁中字型 fallback 混用   | 中英文筆畫、字寬與數字基線不一致     | 建立繁中優先字型與數字字型策略                      |
| 幾乎每個區塊都是白色邊框卡片     | 視覺層級扁平，重要與次要資訊長得一樣 | 只在需要獨立語意或互動邊界時使用 Card               |
| 側欄寬且對比很重                 | 左側視覺重量長期壓過內容             | 改為 208px 到 216px 的精簡 navigation rail          |
| `Wealth OS` 為裝飾性小標         | 佔空間但沒有操作價值                 | 改成資料更新狀態或直接移除                          |
| KPI 固定四張等寬卡片             | 形成模板感，手機版尤其擁擠           | 桌機採 summary strip，手機採 2 欄資訊格             |
| 顏色同時承擔品牌、資產類型與狀態 | 圖表與狀態容易混淆                   | 分離 brand accent、semantic colors 與 chart palette |
| 手刻卡片與共用 Card 並存         | radius、border、shadow 容易漂移      | 收斂為 Surface、Metric、DataRow、Section 四種結構   |
| 載入狀態多為文字 EmptyState      | 版面會跳動，使用者不知道將出現什麼   | 使用符合最終版面的 skeleton                         |
| 圖表依賴顏色區分                 | 色弱與小螢幕辨識度不足               | 同時使用標籤、線型、數值與直接標記                  |
| 設定頁用五張 connector 卡片平鋪  | 狀態難比較，展開表單後版面跳動       | 改為 connector status list 與右側 detail panel      |
| 發票頁上半部留白較多             | 資料密度與搜尋效率不足               | KPI 改為 inline summary，搜尋與篩選固定在內容頂端   |
| 手機總覽一次放三張 KPI           | 文字擠壓且掃讀順序不清楚             | 使用 2 欄格，最重要的投資資訊獨占一列               |
| 部分次要文字對比偏低             | 淺灰文字在暖白底上閱讀吃力           | 所有正文與操作文字至少通過 WCAG AA                  |

## 目標視覺方向：Calm Financial Console

介面應像一份會即時更新的個人資產報表，而不是銀行行銷網站。整體採冷靜的淺色金融工作台，使用單一 muted teal 作為品牌操作色，正負值維持獨立語意色。

### 字體

- 繁中與 UI：自託管的 `Noto Sans TC` 子集，fallback 使用 `PingFang TC`、`Microsoft JhengHei`、system-ui。
- 英文品牌文字：與繁中共用 sans family，不加入無必要的 serif。
- 金額、百分比、日期：使用 tabular figures；必要時加入單一 mono family，但不讓所有數字呈現程式碼感。
- 頁面標題使用 28px 到 32px、600 weight，避免 dashboard 使用過度巨大的 display text。
- 最小正文維持 14px，輔助資訊不得低於 12px。

### 色彩角色

| Token                    | 建議用途                     |
| ------------------------ | ---------------------------- |
| `--color-canvas`         | 全站冷灰白畫布               |
| `--color-surface`        | 主要內容表面                 |
| `--color-surface-subtle` | 分組、hover、表頭與 skeleton |
| `--color-ink`            | 主要文字                     |
| `--color-ink-muted`      | 次要資訊，仍需符合對比       |
| `--color-brand`          | 連結、選取、focus、主要操作  |
| `--color-positive`       | 收入、成長、正常狀態         |
| `--color-negative`       | 支出、負債、失敗狀態         |
| `--color-warning`        | 需要人工處理、資料即將失效   |

Chart palette 與 semantic colors 分開管理。圖表色只表示資產或分類，不拿來表示成功或錯誤。

### 形狀與表面

- 頁面級 panel：12px radius。
- input 與 button：8px radius。
- badge 與短狀態：可以使用 full pill。
- 大部分資料區域以 whitespace、section title 與單側 divider 分組。
- shadow 只用在浮動 header、popover 與 overlay；一般 Card 不預設 shadow。
- 邊框統一使用低對比冷灰，不混用 `ink/10`、`border-border` 與任意 Tailwind gray。

### 動態

- hover：150ms 到 200ms，僅改變 background、border、opacity 或 transform。
- press：`scale(0.98)` 或下移 1px，提供明確觸覺回饋。
- 內容更新：數值淡入，不使用計數器式動畫。
- chart range 切換：200ms crossfade，避免整張圖從零重畫。
- 同步中：只在真正執行同步的來源顯示進度，不放全頁無限動畫。
- 全部自動 motion 必須支援 `prefers-reduced-motion`。

## 資訊架構

既有 IA 保留，不更改 route 與主導覽名稱。

| 主區域 | 使用者主要問題       | 第一層內容                                 |
| ------ | -------------------- | ------------------------------------------ |
| 總覽   | 我現在的財務狀態如何 | 淨資產、同步狀態、資產走勢、配置、近期異常 |
| 資產   | 我的錢在哪裡         | 資產總額、銀行、信用卡、投資、其他資產     |
| 活動   | 最近發生哪些財務變動 | 月現金流、分類、交易時間軸、待分類         |
| 發票   | 我在哪裡買了什麼     | 搜尋、月份、商家、金額、品項明細           |
| 設定   | 資料如何進來並被整理 | 來源、排程、匯率、分類規則、安全狀態       |

## 全域 App shell

### 桌機

- navigation rail 縮為 208px 到 216px，品牌區只保留名稱與資料新鮮度。
- active item 使用品牌色左側 indicator、字重與淡色背景，不用大面積灰色按鈕。
- header 分成頁面名稱、最後更新時間、隱私切換與 context action。
- 主內容 max width 維持 1440px，但使用 12 欄 grid，避免所有頁面都從同一個 card grid 開始。
- 隱藏金額切換加入 tooltip 與短暫狀態回饋。

### 平板

- 頂部導覽保留單行，不允許換行。
- context action 可收進 overflow menu，避免 header 擁擠。
- 圖表與列表優先垂直堆疊，不做窄版雙欄。

### 手機與 PWA

- 底部導覽保留四個主要入口：總覽、資產、活動、更多。
- 發票與設定放在更多，不改變既有操作邏輯。
- header 高度固定，處理 safe area；頁面標題不重複顯示描述。
- KPI 改為 2 欄 grid，重要數值可以跨兩欄。
- 所有 chart filter 改為可水平捲動的 segmented control，避免換行。
- detail view 保留 swipe back，同時維持明確返回按鈕。

## 各頁重構規畫

### 總覽

1. 將淨資產保留為唯一 dominant surface。
2. 同步健康度從同尺寸卡片改為 header 下方的 compact status summary；只有異常時升級為 warning panel。
3. 投資、存款、其他資產與月淨流入改為同一條 summary strip，不使用四張獨立 Card。
4. 資產走勢保留為最大資料視覺，range 與分類切換收斂在 chart toolbar。
5. 資產配置使用直接標記與數值，不只依賴色條。
6. 新增「需要處理」區域，只在同步失敗、待分類或資料過期時出現。
7. 近期活動只顯示 5 到 8 筆，提供前往活動的明確入口。

### 資產

1. 資產總額與信用卡負債形成同一個 balance summary，避免把負債視為普通資產 KPI。
2. 保留全部、銀行、信用卡、投資、其他資產的 segmented navigation。
3. 銀行與投資改用統一 DataRow，左側為機構與帳戶，右側為原幣與換算金額。
4. 同步狀態使用文字與 status badge，不用顏色小點作為唯一提示。
5. 桌機 detail 可以使用 8/4 內容與 context panel；手機全部單欄。

### 活動

1. 月收入、月支出、淨流入與待分類改成 summary strip。
2. 收入與支出甜甜圈整合為一個可切換視圖，預設顯示支出分類。
3. 現金流趨勢作為主要 chart，月份篩選與 include/exclude 規則放在 toolbar。
4. 交易時間軸使用 sticky month group、來源 icon、分類 badge 與原幣資訊。
5. 待分類交易提供 inline classify，不要求離開目前頁面。

### 發票

1. 本月消費、總數、均消改為 inline summary，不使用三張 KPI Card。
2. 搜尋、月份、商家與金額篩選整合成 sticky filter bar。
3. 桌機採高密度 disclosure table，手機採目前的 accordion row。
4. 展開品項時顯示數量、單價與小計，合計保持固定層級。
5. 搜尋結果顯示命中欄位，讓使用者知道是商家、號碼或品項命中。

### 設定

1. 桌機改成左側 section navigation 與右側內容，手機維持目前的 More 子頁。
2. connector 從五張平鋪 Card 改為 status list，便於比較已設定、上次成功、排程與需要處理。
3. 點選 connector 後以固定 detail panel 顯示設定，不讓整個 grid 高度跳動。
4. 預設同步排程改為單一 configuration bar，顯示 timezone 與跟隨預設的來源數。
5. 匯率與分類規則拆成各自頁面，避免設定首頁變成超長 dashboard。
6. 帳密表單附近持續顯示加密與不回顯說明，但不使用大型裝飾性安全 Card。

## 共用元件策略

| 元件           | 責任                                                |
| -------------- | --------------------------------------------------- |
| `AppShell`     | navigation rail、top bar、mobile tab bar、safe area |
| `PageHeader`   | title、description、freshness、context action       |
| `SummaryStrip` | 2 到 5 個緊密 KPI，不建立多張 Card                  |
| `Metric`       | label、value、comparison、semantic tone             |
| `Surface`      | 需要獨立視覺邊界的 panel                            |
| `Section`      | 無 Card 的內容分組、標題與 action                   |
| `DataRow`      | 帳戶、持倉、交易、發票的共用掃讀結構                |
| `StatusBadge`  | success、warning、error、neutral                    |
| `ChartFrame`   | title、legend、range、empty、loading、error         |
| `FilterBar`    | search、date、category、sort、clear                 |
| `Skeleton`     | 對應 Metric、DataRow 與 chart 的載入形狀            |
| `InlineAlert`  | 同步失敗、待驗證、離線與資料過期                    |

若新增自訂 CSS class，使用 BEM 與 namespace，例如 `.c-summary-strip__metric--negative`；現有 Tailwind utility 可以繼續使用，不為了 BEM 強制重寫所有 utility。

## 狀態矩陣

每個資料區域都要明確支援下列狀態：

| 狀態         | 顯示原則                                        |
| ------------ | ----------------------------------------------- |
| Loading      | 保留最終 layout 尺寸的 skeleton                 |
| Empty        | 說明為何沒有資料，提供下一個可執行動作          |
| Error        | 顯示失敗範圍、重試與必要的重新驗證入口          |
| Syncing      | 只標示正在同步的來源與開始時間                  |
| Stale        | 顯示資料時間與「重新同步」操作                  |
| Needs action | 使用 warning tone，清楚描述 OTP、登入或驗證需求 |
| Offline      | 保留已快取資料，停用會寫入或同步的操作          |
| Money hidden | 金額使用一致遮罩，圖表 tooltip 與 axis 同步隱藏 |

## 無障礙與內容規範

- 所有文字與控制項符合 WCAG AA，正文目標 4.5:1。
- chart 提供文字摘要、資料表替代或可讀 legend。
- 不以顏色作為唯一狀態提示。
- icon-only button 必須有可辨識的 `aria-label` 與 tooltip。
- focus 順序符合視覺順序，sticky bar 不遮住聚焦元素。
- 支援 200% zoom、320px viewport 與 `prefers-reduced-motion`。
- 可見文案使用繁體中文，不混用裝飾性英文小標。
- 金融數值格式統一：正負號、幣別、千分位、小數位與原幣顯示規則集中處理。
- 避免 `·` 連續串接資訊，改用欄位、換行或單一分隔符號。
- 空值統一顯示 `-` 或「尚無資料」，不混用多種 dash 字元。

## 實作階段

### Phase 0：基準與保護

- 保存目前八張 reference screenshots。
- 建立主要 viewport 的 Playwright visual snapshots。
- 列出既有 route、analytics hook、form field 與 API contract。
- 建立 accessibility 與 performance baseline。

### Phase 1：Design foundations

- 重整 `styles.css` semantic tokens、字體、spacing、radius、border、shadow 與 z-index。
- 收斂 Button、Card、Badge、Input、Select、Tabs 與 EmptyState。
- 新增 SummaryStrip、Surface、Section、DataRow、Skeleton、InlineAlert。
- 不改變任何頁面資料流程。

### Phase 2：App shell 與 responsive behavior

- 重構 desktop navigation rail、tablet top nav 與 mobile bottom nav。
- 統一 PageHeader、safe area、sticky behavior 與 money visibility feedback。
- 保留 hash route、PWA standalone 與 swipe back。

### Phase 3：總覽與資產

- 先完成總覽，建立新的 KPI、chart 與 status pattern。
- 將相同 pattern 套用至資產、銀行、信用卡、投資與其他資產。
- 完成 desktop、tablet、mobile 三組 visual QA。

### Phase 4：活動與發票

- 整合活動 chart、filter 與 transaction timeline。
- 重做發票 filter、desktop disclosure table 與 mobile accordion。
- 補齊待分類、搜尋無結果與大量資料狀態。

### Phase 5：設定

- 重組 settings section navigation。
- 將 connectors 改為 status list 與固定 detail panel。
- 整理排程、匯率、分類規則與 credential states。

### Phase 6：品質驗收

- 執行 typecheck、unit tests、E2E 與 build。
- 檢查 keyboard、screen reader、contrast、200% zoom 與 reduced motion。
- 檢查 320、375、768、1024、1440、1536 px viewport。
- 對照 visual snapshots，確認沒有 route、資料或表單回歸。
- 驗證 LCP、INP、CLS 與 mobile scroll performance。

## 主要影響檔案

- `apps/web/src/styles.css`
- `apps/web/src/App.svelte`
- `apps/web/src/components/ui/*`
- `apps/web/src/features/overview/*`
- `apps/web/src/features/assets/*`
- `apps/web/src/features/activity/*`
- `apps/web/src/features/invoices/*`
- `apps/web/src/features/settings/*`
- `apps/web/e2e/*`
- `images/screenshots/*`

## 驗收標準

- 使用者在 5 秒內能回答目前淨資產、主要資產位置、月淨流入與是否有同步異常。
- 桌機每個頁面只有一個主要視覺焦點，不再由同尺寸 Card 平均分散注意力。
- 手機 375px 不出現 KPI 文字擠壓、control wrap、橫向頁面 overflow 或底部導覽遮擋。
- 所有列表在 100 筆資料下仍可快速掃讀並保有清楚分組。
- 所有主要操作具備 hover、focus、pressed、disabled、loading、success 與 error 狀態。
- 金額隱藏時，頁面、chart、tooltip、aria label 不洩露實際金額。
- 不改變既有 navigation label、route hash、API contract、form field 或金融計算結果。
- 完成後所有既有測試通過，並新增關鍵 visual 與 accessibility coverage。
