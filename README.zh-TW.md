# claude-telegram

[English](README.md) | 繁體中文

透過 Telegram 遙控 Claude Code 的常駐 AI 秘書，附帶跨重啟記憶持久化（轉生機制）。

---

## 為什麼需要這個？

### 核心問題：`claude --channels` 是一個不能重啟的長 session

用 `claude --channels plugin:telegram` 跑 Telegram bot 時，整個對話是一個持續累積的 session。用越久，context 越長，問題接踵而來：

- **自動壓縮不可控**：context 超長時 Claude Code 會自動 compaction，但壓縮邏輯你無法介入，重要細節可能被丟掉
- **不敢隨意重啟**：想要一個乾淨的新 session？重啟就等於完全失憶——上次做到哪裡、交代了什麼、還有哪些待辦，全部歸零
- **崩潰就消失**：服務異常重啟時，沒有任何機制保存當前狀態

結果就是：你要麼忍受越來越臃腫的 context，要麼重啟換來乾淨但失憶的 AI。

### 解法：主動式 context 轉移（轉生機制）

與其被動等待自動壓縮，不如讓 AI **主動、有意識地**替自己做摘要再重啟。

當你說「去轉生」，AI 會：
1. 把這次 session 的對話重點、待辦事項、重要結論寫進記憶檔
2. 自己執行 `systemctl restart` 開啟新 session
3. 新 session 讀取記憶，從摘要繼續——context 乾淨，記憶不丟

這是一種「有損但受控」的 context 轉移：你犧牲完整的對話歷史，換取精煉過的關鍵記憶加上全新的 context 空間。

### 第二痛點：人不在電腦前，無法重啟對話

就算你知道 context 太長了、該重啟了，如果你不在電腦前，你也什麼都做不了——只能讓那個越來越臃腫的 session 繼續撐著。

透過 Telegram，你在任何地方都可以對 AI 說「去轉生」，它會自己完成摘要、重啟、上線，完全不需要你碰終端機。

### 附帶解決：長任務透明度

長任務執行期間，強制 AI 先回一條進度訊息，每個階段更新一次，讓你隨時知道它在幹嘛，而不是等一個不知道何時出現的最終回覆。

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

**作業系統**
- Linux（需要 systemd user service）

**1. tmux**

```bash
# Debian / Ubuntu
sudo apt install tmux

# RHEL / Fedora
sudo dnf install tmux
```

**2. Node.js 18+**（Claude Code 的執行環境）

```bash
# 使用 nvm 安裝（推薦）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install --lts
```

**3. Claude Code CLI**

```bash
npm install -g @anthropic-ai/claude-code
claude login   # 以 Anthropic 帳號登入
```

**4. Telegram Bot Token**

1. 在 Telegram 找 [@BotFather](https://t.me/BotFather)
2. 發送 `/newbot`，依指示建立 bot
3. 複製取得的 Token（格式：`123456789:AAF...`）

**5. Telegram MCP Plugin**

在終端機執行 `claude`，進入 Claude Code 後執行：

```
/telegram:configure
```

依提示貼上 Bot Token，完成後 Telegram channel 即啟用。

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

## 自訂 AI 人格與行為

所有行為規範存放於 `$MEMORY_DIR/claude_telegram_prompt.md`，直接用文字編輯器修改，重啟後生效。

這個檔案是純 Markdown，沒有任何格式限制——你可以在裡面加入任何你想要的人格設定，例如：

- **個性**：話多還是話少、正經還是嘴賤、喜歡用什麼語氣
- **稱謂**：叫你什麼、自稱什麼
- **背景故事**：它是誰、從哪裡來、有什麼過去
- **偏好與禁忌**：有哪些習慣、有哪些事絕對不做
- **專業領域**：擅長什麼、遇到哪類問題要特別謹慎

範例片段：

```markdown
## 人格設定

你叫小晴，說話直接不廢話，偶爾毒舌但出發點是真心幫忙。
遇到主人問笨問題不會裝沒事，會直接說「這個你自己查一下比較快」。
喜歡在回覆結尾加一句冷靜的觀察，不帶感情那種。
```

修改完直接對 bot 說「去轉生」，它會帶著新人格重新上線。

---

## 授權

MIT
