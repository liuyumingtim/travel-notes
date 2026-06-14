# 10｜GitHub、VPS、Hermes 與 Telegram 同步更新流程

## 1. 目標

讓 Yoyo 可以在 Codex 裡直接修改 Markdown 資料庫，修改完成後自動或半自動同步到：

```text
GitHub 主資料庫
↓
VPS / Hermes AI
↓
Telegram Bot
```

之後不需要每次重新下載、重新上傳或手動貼 MD。

---

## 2. 建議架構

最推薦架構：

```text
Codex 本機資料夾
↓ git commit / push
GitHub Repo 主資料庫
↓ VPS 自動 pull 或 webhook 更新
VPS / Hermes AI 讀最新 MD
↓
Telegram Bot 回覆最新內容
```

### 角色分工

| 工具 | 用途 |
|---|---|
| Codex | 編輯、整理、修改 MD |
| GitHub | 主資料庫、版本紀錄、備份、回復 |
| VPS | 長期執行 Telegram Bot / Hermes Gateway |
| Hermes AI | 理解問題、讀取 MD、產出回覆 |
| Telegram | 使用入口 |

---

## 3. 為什麼用 GitHub 當主資料庫

GitHub 適合當主資料庫，原因：

- 每次修改都有紀錄。
- 改錯可以回復。
- VPS 可以很容易同步。
- Hermes 或其他 AI 工具通常比較容易讀 GitHub / repo / markdown。
- 不需要每次手動上傳檔案。

不建議只放 Hermes 雲端，因為：

- 修改紀錄可能不好追蹤。
- 若 Hermes 知識庫更新失敗，不一定容易回復。
- 未來若換工具，資料搬移比較麻煩。

Google Drive 可以當備份或人工查看，但不建議當 Bot 的主資料庫。

---

## 4. Codex 更新資料流程

### 4.1 一般更新

使用者可以直接對 Codex 說：

```text
幫我把泰國曼谷必買資料補進 09。
```

或：

```text
幫我更新越南中越巴拿山提醒。
```

Codex 應該：

1. 找到對應 MD。
2. 修改內容。
3. 確認格式沒有壞掉。
4. 告訴 Yoyo 修改了哪些地方。
5. 等 Yoyo 要求時再 commit / push 到 GitHub。

### 4.2 高風險更新

以下內容不可自動直接改：

- 簽證
- 入境表單
- 海關規定
- 菸酒限制
- 電子菸 / 加熱菸
- 航班資訊
- 匯率
- 天氣
- 票價與營業時間

建議流程：

```text
先查證
↓
列出目前 MD 與最新資料差異
↓
Yoyo 確認
↓
再修改 MD
```

---

## 5. GitHub 同步流程

### 5.1 第一次設定

建立 GitHub repo，例如：

```text
yoyo-tour-chatbox-md
```

把 `出團訊息chat box md` 資料夾推上 GitHub。

### 5.2 每次更新

每次在 Codex 修改 MD 後：

```text
git add 出團訊息chat box md/
git commit -m "Update tour chatbox markdown database"
git push
```

未來可以讓 Codex 幫忙做，但執行前應先讓 Yoyo 確認本次修改範圍。

---

## 6. VPS 同步方式

VPS 有兩種同步方式。

### 6.0 重要原則：不要在每次 Telegram 問題中同步

Telegram Bot / Hermes AI 收到旅客公告產出需求時，不應每次都先執行 `git pull`。

建議分工：

```text
同步資料庫 = 定時同步或 webhook 處理
產出旅客公告 = 讀取 VPS 本機現有 MD
```

原因：

- `git pull` 可能因 GitHub 網路、SSH key、權限或鎖定問題卡住。
- Telegram 使用者會覺得 Bot 一直轉圈。
- 即使資料庫暫時不是最新，也比卡住無法回覆更可用。

若真的需要在使用者要求時手動同步，應加上短 timeout，例如 10 秒。超過時間就停止同步，改用本機現有 MD 先產出，並簡短提醒資料庫同步未完成。

### 6.1 簡單版：定時同步

VPS 每 1 到 5 分鐘跑一次：

```text
git pull
```

優點：

- 設定簡單。
- 不需要 webhook。
- 適合第一版。

缺點：

- 不是真的即時，會延遲 1 到 5 分鐘。

### 6.2 正式版：Webhook 即時同步

GitHub push 後，GitHub webhook 通知 VPS：

```text
收到 GitHub push
↓
VPS 執行 git pull
↓
重載 Bot / Hermes knowledge cache
```

優點：

- 幾乎即時。
- 比定時同步有效率。

缺點：

- 需要設定 VPS API endpoint 與 webhook secret。

---

## 7. Telegram Bot 讀取最新資料

Bot 應避免把 MD 永久存在記憶裡不更新。

建議有兩種做法：

### 7.1 每次回覆前讀取最新 MD

最簡單，也最保險。這裡的「最新 MD」是指 VPS 本機目前已同步的檔案，不是每次問題都重新 `git pull`。

```text
收到 Telegram 問題
↓
讀取目前 VPS 本機上的最新 MD
↓
產出回覆
```

適合 MD 數量不大時使用。

### 7.2 使用快取，但更新後重載

若未來資料很多，可以使用快取：

```text
啟動時讀取 MD
↓
平常用快取回覆
↓
GitHub 更新後重載快取
```

---

## 8. 推薦第一版做法

第一版建議不要太複雜：

```text
GitHub 當主資料庫
VPS 每 3 分鐘 git pull
Telegram Bot 每次收到問題時讀取最新 MD
Hermes 根據最新 MD 產出回覆
```

這樣已經能達成：

- Codex 修改 MD
- GitHub 保存版本
- VPS 自動同步
- Telegram 使用最新資料

---

## 9. 未來更新資料範例

### 9.1 更新推薦資料

使用者：

```text
幫我把泰國必買補進 09：鼻通、乳膠枕、泰奶茶包、草本藥膏。
```

Codex：

```text
已更新 09_必買必吃必拍旅遊推薦資料庫.md 的泰國通用必買伴手禮。
```

Yoyo 確認後：

```text
commit / push 到 GitHub
```

VPS：

```text
自動 pull 最新版本
```

Telegram：

```text
幫我產出泰國必買推薦
```

Bot 讀取最新 09，產出回覆。

### 9.2 更新高風險資料

使用者：

```text
幫我查新加坡電子菸規定有沒有變，先不要改 MD。
```

Codex / Hermes：

```text
查證官方來源
列出差異
等待 Yoyo 確認
```

Yoyo：

```text
確認更新
```

Codex / Hermes 才更新 MD。

---

## 10. 成功標準

整套同步成功時，應該可以做到：

- Yoyo 在 Codex 修改 MD。
- 修改推上 GitHub。
- VPS 自動同步最新 MD。
- Telegram Bot 不需要重新上傳資料就能讀到更新內容。
- 必買、必吃、必拍、注意事項都能依最新 MD 產出。
- 高風險規定更新前會先確認，不會自動亂改。
