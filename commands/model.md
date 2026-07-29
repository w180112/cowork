---
description: 切換 implementer 後端與模型(改 .claude/cowork.md,下次委派生效)
---

切換 cowork 的 implementer 後端 / 模型。使用者附加的參數:$ARGUMENTS

1. 讀 `.claude/cowork.md`,取現行 `implementer` 與 `implementer_model`。
   檔案不存在就請使用者先跑 `/cowork:init`,結束。

2. 決定新值:
   - $ARGUMENTS 已指定就直接採用,可以只給模型(如 `sonnet`、
     `zai/glm-5.2`,後端不變)或給兩者(如 `codex gpt-5.5`)。
   - 沒指定就用 AskUserQuestion 問,現行值標為預設:
     - 後端:`claude` / `codex` / `opencode`
     - 模型:claude → `opus` | `sonnet`;codex → `gpt-5.6` |
       `gpt-5.5`;opencode → 完整 `provider/model` 字串

3. 用 Edit 改 `.claude/cowork.md` 的那一(兩)行,不要動其他內容。

4. 依新後端提醒(只講有變的那項):
   - 舊值是 `claude`、新值是 `codex` 或 `opencode`:AGENTS.md 缺
     協定段落,請使用者跑 `/cowork:init` 補上,否則 task-loop 委派
     前會擋下來。
   - 新後端是 `codex`:若 `/codex:rescue` 不支援指定模型,要同步把
     `~/.codex/config.toml` 的 `model` 改成同一個值,並 kill 掉
     `app-server-broker.mjs` 與 `codex app-server` 進程讓新設定生效。
   - 新後端是 `opencode`:確認該 provider 已 `opencode auth login`。

5. 回報 `舊值 → 新值`,並說明下一次委派生效;任務進行到一半換人可行
   (協定狀態在 `artifacts/task-<id>/`),但舊實作端的 thread 記憶
   不會跟過去,建議在任務邊界切換。
