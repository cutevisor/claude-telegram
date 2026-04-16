# claude-telegram

透過 Telegram 遙控 Claude Code 的常駐 AI 秘書，附帶跨重啟記憶持久化（轉生機制）。

---

## 為什麼需要這個？

### 問題一：Claude Code 被鎖在終端機裡

Claude Code 是目前最強的 AI coding agent，可以讀檔、執行指令、修改程式碼。但它有一個根本限制：**你必須坐在電腦前，開著終端機，才能用它**。

手機上沒有它。出門在外沒有它。你睡著的時候，它也沉睡著。

這個專案把 Claude Code 接上 Telegram，讓它變成一個隨時待命的 AI 秘書——不管你在哪裡，只要打開 Telegram，就能派它去執行任務。

### 問題二：AI 每次重啟都失憶

Claude Code 是無狀態的：每次啟動就是全新的對話，完全不記得上次做了什麼、你交代了什麼、有哪些未完成的事。

服務一崩潰，記憶歸零。這讓 AI 永遠只是一個工具，而不是一個真正的秘書。

這個專案引入了**轉生機制**：每次結束前，AI 自己把對話重點、待辦事項、你的偏好寫進記憶檔；下次啟動時，記憶自動注入 system prompt，它就知道自己是誰、上次做到哪裡、接下來該幹嘛。

### 問題三：長任務像黑盒子

當 AI 在後台默默執行一個需要幾十個步驟的任務時，你完全不知道它在幹嘛——直到它回覆，或者沒有回覆。

這個專案強制 AI 遵守進度回報協議：立刻先回一條「處理中」的訊息，然後每完成一個階段就更新它，讓你隨時知道進度。

---

## 特色

- **Telegram 遠端控制**：隨時隨地透過手機下指令，AI 在你的伺服器上執行
- **轉生記憶**：對話摘要自動持久化，重啟不失憶
- **即時進度回報**：長任務逐步更新，不是黑盒子
- **常駐服務**：systemd 管理，崩潰自動重啟
- **完全可自訂**：AI 名稱、行為規範、記憶格式都可以修改

---

## 運作原理

```
Telegram 訊息
    ↓
Claude Code + Telegram MCP Plugin
    ↓  (system prompt 注入)
    ├─ 環境設定（chat_id、AI 名稱）
    ├─ 上輩子記憶（上次對話摘要）
    └─ 行為規範（回應準則、轉生流程）
    ↓
reply / edit_message / react
    ↓
Telegram 回覆
```

轉生記憶流程：

```
主人說「去轉生」
    ↓
AI 將對話摘要寫入 past_life_memory.md
    ↓
systemctl restart claude-telegram.service
    ↓
新實例啟動，讀取記憶，發送上線招呼
```

---

## 快速開始

### 前置需求

- [Claude Code CLI](https://claude.ai/code) 已安裝並登入
- Telegram Bot Token（向 [@BotFather](https://t.me/BotFather) 申請）
- Telegram MCP Plugin 已設定（`/telegram:configure`）
- tmux、systemd（Linux）

### 安裝步驟

**1. 取得程式碼**

```bash
git clone https://github.com/cutevisor/claude-telegram.git
cd claude-telegram
```

**2. 填寫個人設定**

```bash
# 編輯 config.sh，填入以下欄位：
# - OWNER_CHAT_ID：你的 Telegram chat_id
# - AI_NAME：AI 秘書的名字
# - HOME_DIR：你的家目錄（如 /home/username）
nano config.sh
```

> 不知道自己的 chat_id？對 bot 發任意訊息，bot 會在回覆中包含你的 chat_id，或使用 [@userinfobot](https://t.me/userinfobot) 查詢。

**3. 建立工作目錄並複製記憶檔**

```bash
WORK_DIR=$(grep WORK_DIR config.sh | cut -d'"' -f2 | envsubst)
mkdir -p "$WORK_DIR/memory"
cp past_life_memory.md "$WORK_DIR/memory/"
cp claude_telegram_prompt.md "$WORK_DIR/memory/"
```

**4. 安裝啟動腳本**

```bash
chmod +x claude-telegram-wrapper.sh claude-telegram-cmd
cp claude-telegram-wrapper.sh ~/.local/bin/
cp claude-telegram-cmd ~/.local/bin/
```

**5. 安裝 systemd service**

```bash
# 將 YOUR_USERNAME 替換為你的 Linux 使用者名稱
sed "s/YOUR_USERNAME/$USER/g" claude-telegram.service \
  > ~/.config/systemd/user/claude-telegram.service

systemctl --user daemon-reload
systemctl --user enable --now claude-telegram.service
```

**6. 確認啟動**

```bash
systemctl --user status claude-telegram.service
# 約 20 秒後，你的 Telegram 應該會收到 AI 的上線招呼
```

---

## 使用方式

直接在 Telegram 對 bot 說話即可。

| 操作 | 方式 |
|------|------|
| 一般對話 | 直接發訊息 |
| 壓縮對話（節省 context） | 說「壓縮對話」 |
| 重啟並保存記憶 | 說「去轉生」 |
| 查看服務狀態 | `systemctl --user status claude-telegram.service` |
| 進入 Claude 會話 | `tmux attach-session -t claude-telegram` |

---

## 自訂 AI 行為

所有行為規範存放於 `$MEMORY_DIR/claude_telegram_prompt.md`，直接編輯即可，重啟後生效。記憶摘要格式存放於 `$MEMORY_DIR/past_life_memory.md`。

---

## 授權

MIT
