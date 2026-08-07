---
description: 切換 planner / implementer 的後端與模型(改 .claude/cowork/config.md,下次委派生效)
---

切換 cowork 的 planner 或 implementer 後端 / 模型。使用者附加的
參數:$ARGUMENTS

1. 讀 `.claude/cowork/config.md`,取現行 `planner` / `planner_model` /
   `implementer` / `implementer_model`。檔案不存在就請使用者先跑
   `/cowork:init`,結束。

2. 決定角色與新值:
   - $ARGUMENTS 第一個字是 `planner` 或 `implementer` → 指定該
     角色,其餘參數是新值;沒指明角色但有給值 → 預設 implementer。
   - 值可以只給模型(如 `sonnet`、`zai/glm-5.2`,後端不變)或
     後端 + 模型(如 `codex gpt-5.5`)。
   - 完全沒參數就用 AskUserQuestion 問:先選角色,再選後端與模型,
     現行值標為預設:
     - 後端:`claude` / `codex` / `opencode`
     - 模型:claude → planner 可選 `fable` | `opus` | `sonnet`,
       implementer 可選 `opus` | `sonnet`;codex → `gpt-5.6` |
       `gpt-5.5`;opencode → 完整 `provider/model` 字串
   - 改完後 planner 與 implementer 會同時是 codex 的話,擋下來說明
     原因(--resume 單 thread 會搶線),請使用者改選。

3. 用 Edit 改 `.claude/cowork/config.md` 對應的那幾行,不要動其他內容。

4. 依新後端提醒(只講有變的那項):
   - 該角色從 `claude` 改成 `codex` 或 `opencode`,且 AGENTS.md
     還沒有 cowork 協定段落:請使用者跑 `/cowork:init` 補上,否則
     task-loop 委派前會擋下來。
   - 新後端是 `codex`:若 `/codex:rescue` 不支援指定模型,要同步把
     `~/.codex/config.toml` 的 `model` 改成同一個值,並 kill 掉
     `app-server-broker.mjs` 與 `codex app-server` 進程讓新設定生效。
   - 新後端是 `opencode`:確認該 provider 已 `opencode auth login`。

5. 回報 `角色:舊值 → 新值`,並說明下一次委派生效;任務進行到一半
   換人可行(協定狀態在 `.claude/cowork/artifacts/task-<id>/`),但舊後端的 thread
   記憶不會跟過去,建議在任務邊界切換。
