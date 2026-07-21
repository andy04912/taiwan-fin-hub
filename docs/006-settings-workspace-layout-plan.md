# Taiwan Fin Hub — Settings Workspace Layout Plan

## Design Read

將設定頁視為既有財務 App Shell 內的一個管理工作區，而不是另一個巢狀 App。設計語言維持 Calm Financial Console；降低導覽層級、提高內容寬度，讓操作項目比框架更突出。

## 問題診斷

目前設定頁同時存在：

1. 全站 Sidebar
2. 全站 Header
3. 設定 Sidebar
4. 設定狀態列
5. 設定內容捲動區
6. 行動版設定專用 Hamburger Drawer

這造成視覺上像「App 裡再開一個 App」，也縮減資料來源與表單的有效寬度。

## 新資訊架構

只保留三種明確角色：

- 全站 Sidebar / Hamburger：總覽、資產、活動、發票、設定。
- 設定 Sticky Tabs：設定總覽、同步與資料來源、匯率、分類規則。
- 資料來源選擇器：只在同步與資料來源頁面選擇連接器。

## Desktop Layout

- 全站 Sidebar 與 Header 維持固定。
- Header 下方使用單列 Sticky Tabs。
- 設定內容只有一個垂直捲動容器。
- 每次只顯示一個設定分類，不使用 Scroll Spy 長頁。
- 資料來源頁採局部 Master–Detail：左側 300–340px 來源列表，右側連接器設定。

## Mobile / PWA Layout

- 使用全站 Hamburger，不建立設定專用 Drawer。
- 設定分類使用可橫向滑動的 Sticky Tabs。
- 每個觸控目標至少 44px。
- 內容保留底部 Safe Area 與全站 Bottom Navigation 空間。
- 資料來源在窄螢幕改為上下堆疊，不強迫雙欄。

## Panels

### 設定總覽

- 資料健康度主狀態
- 資料來源、啟用排程、帳戶與分類規則四個摘要
- 各資料來源目前狀態
- 快速管理入口
- 安全與隱私說明

### 同步與資料來源

- 預設同步排程
- 連接器 Master–Detail
- 憑證安全說明

### 匯率

- 獨立滿版匯率管理工作區

### 分類規則

- 獨立滿版規則管理工作區

## Motion

- Tab 切換：`fade + translateY 8px`，約 220ms。
- Tab active state：底線與背景平滑切換。
- Overview 卡片：stagger reveal。
- 連接器切換：右側內容 keyed transition。
- 按鈕、資料列與 Surface 延用全站觸覺回饋。
- 完整支援 `prefers-reduced-motion`。

## Design Dials

- `DESIGN_VARIANCE: 5`
- `MOTION_INTENSITY: 6`
- `VISUAL_DENSITY: 7`

設定頁需要比行銷頁更可預期，因此降低變異與動態強度，但保留明確狀態回饋與高資料密度。
