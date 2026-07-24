# Upstream 同步操作手冊

本文件用來把 `TedLin1993/taiwan-fin-hub:main` 的更新同步到 `andy04912/taiwan-fin-hub:main`，並避免再次踩到無共同 Git 歷史、前端大量 rename、錯誤覆蓋 UI 與 GitHub Actions 權限等問題。

## 固定資訊

```text
TARGET_REPO=andy04912/taiwan-fin-hub
UPSTREAM_REPO=TedLin1993/taiwan-fin-hub
TARGET_BRANCH=main
UPSTREAM_BRANCH=main
LAST_SYNCED_UPSTREAM_SHA=67ea96de8b0a716dcce88697d02f30ab6cdfd985
LAST_SYNC_COMMIT=e405862e3121043a59a4f901581a47fab511e616
```

每次同步成功後，必須更新上面的 `LAST_SYNCED_UPSTREAM_SHA`。下次同步只處理該 SHA 之後的 upstream 增量，不要重新合併整段歷史。

## 重要前提

本 Repository 不是 GitHub 認定的原生 Fork，與 upstream 沒有可靠的共同 Git ancestry。因此：

- 不要直接執行 `git merge upstream/main`。
- 不要使用 `--allow-unrelated-histories`。
- 不要使用 force push 更新 `main`。
- 不要把所有衝突一次選成 upstream 版本。
- 不要完全相信 Git 的 rename detection，尤其是 `apps/web/src`。

## 下次同步標準流程

### 1. 建立同步分支

```bash
git fetch origin
git checkout main
git pull --ff-only origin main
git checkout -b sync/upstream-$(date +%Y%m%d)
```

確認工作目錄乾淨：

```bash
git status --short
```

### 2. 取得 upstream 最新版本

```bash
git remote get-url upstream >/dev/null 2>&1 || \
  git remote add upstream https://github.com/TedLin1993/taiwan-fin-hub.git

git fetch upstream main
```

讀取本文件中的基準：

```bash
BASE=67ea96de8b0a716dcce88697d02f30ab6cdfd985
UPSTREAM_HEAD=$(git rev-parse upstream/main)
```

若 `BASE` 等於 `UPSTREAM_HEAD`，代表沒有新內容，不需要同步。

### 3. 只套用 upstream 增量

不要 merge 完整 upstream 歷史。改成產生從上次同步點到最新 upstream 的 patch：

```bash
git diff \
  --binary \
  --full-index \
  --find-renames \
  "$BASE" "$UPSTREAM_HEAD" \
  > /tmp/taiwan-fin-hub-upstream.patch

git apply --3way --index /tmp/taiwan-fin-hub-upstream.patch
```

這樣只會處理「上次同步後 upstream 新增的變更」，不會再次處理過去所有檔案。

若 `git apply --3way` 發生衝突，先查看：

```bash
git status
git diff --name-only --diff-filter=U
```

### 4. 衝突處理原則

依照以下優先順序處理：

#### 預設採用 upstream

- `apps/worker/**`
- `packages/db/**`
- `migrations/**`
- 後端 API、同步排程、通知、OCR 與連接器
- 文件與測試
- upstream 新增且本地沒有客製化的檔案

#### 預設保留目前 Repo

- 已完成的 App Shell
- 現有頁面布局與視覺設計
- `apps/web/src/features/**` 中已客製化的 UI
- 本地導覽、動畫、樣式與互動設計

#### Upstream 新前端架構可以保留

Upstream 新增的下列目錄可以納入，但不要直接取代目前入口：

```text
apps/web/src/app
apps/web/src/data
apps/web/src/shared
apps/web/src/testing
```

若 upstream 搬動大量前端檔案，不要讓 Git 自動把舊 UI 內容誤套到新路徑。必須逐一確認檔案用途。

#### 型別擴充必須同步補齊 UI

若 upstream 新增 connector 或 union type，例如：

```ts
ConnectorId = "..." | "taishin"
```

必須檢查所有：

```ts
Record<ConnectorId, ...>
switch (connectorId)
connector list
settings fields
```

避免 Typecheck 因缺少新 key 而失敗。

### 5. 完整驗證

合併後必須全部通過：

```bash
npm ci
npm run typecheck
npm run test:backend
npm run test:unit
npm run build
git diff --check
```

任何一步失敗都不能推入 `main`。

### 6. 更新同步基準

確認驗證通過後，把本文件中的：

```text
LAST_SYNCED_UPSTREAM_SHA=
```

更新為：

```bash
git rev-parse upstream/main
```

接著提交：

```bash
git add -A
git commit -m "chore: 同步 upstream/main"
```

### 7. 透過 PR 合併

```bash
git push -u origin HEAD
gh pr create \
  --base main \
  --title "chore: 同步 upstream/main" \
  --body "同步 upstream 最新增量，保留目前 UI，並已通過 Typecheck、後端測試、前端測試與 production build。"
```

CI 通過後再合併 PR。不要直接 force push 到 `main`。

## 同步失敗時的復原

所有操作都應該在 `sync/upstream-*` 分支執行。需要重來時：

```bash
git reset --hard origin/main
git clean -fd
```

這不會影響遠端 `main`。

## 禁止事項

```text
禁止：git merge upstream/main
禁止：git merge --allow-unrelated-histories
禁止：git push --force origin main
禁止：遇到前端衝突時整批選 theirs
禁止：驗證未通過就更新 main
禁止：同步成功後忘記更新 LAST_SYNCED_UPSTREAM_SHA
```

## 下次可以直接使用的請求

將下面這段直接交給執行者：

> 請依照 `docs/upstream-sync-playbook.md` 同步 `TedLin1993/taiwan-fin-hub:main` 到本 Repo。使用文件中的 `LAST_SYNCED_UPSTREAM_SHA`，只套用該 SHA 到最新 upstream/main 的增量，不要直接 merge unrelated histories。衝突時保留目前 App Shell 與客製化 UI，納入 upstream 後端、資料庫、連接器與新架構更新。完成後執行 Typecheck、後端測試、前端測試與 production build，更新文件中的同步基準，透過 PR 合併到 main，禁止 force push。

## 為什麼要這樣做

之前同步困難的根本原因不是單純程式衝突，而是兩個 Repository 沒有可靠共同歷史。使用「最後同步 upstream SHA → 最新 upstream SHA」的增量 patch，可以把問題範圍限制在真正的新變更，避免每次重新處理整個專案歷史。