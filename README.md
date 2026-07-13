# cowork

Claude Code plugin:task-loop 雙 agent 工作流。Claude Code 負責
規劃 / code review / audit / commit 交接,實作委派給可替換的
implementer,後端與模型都在 init 時選、之後可隨時改:

| 後端 | 模型選項 | 委派方式 |
|---|---|---|
| `claude` | opus(預設)/ sonnet | plugin 內建 implementer subagent |
| `codex` | gpt-5.6(預設)/ gpt-5.5 | openai-codex plugin `/codex:rescue` |
| `opencode` | 任意 `provider/model`,如 `zai/glm-5.2` | opencode CLI `opencode run` |

源自 fastrg-node 的 task-loop 協定,抽掉專案特定內容後參數化。

## 安裝

```
/plugin marketplace add /root/cowork
/plugin install cowork@cowork-marketplace
```

前提:

- implementer 要用 codex 的話,需先安裝 openai-codex plugin;Codex
  sandbox 若需放寬(`.codex/config.toml` 等)屬環境層設定,init 只
  提醒不代管。
- implementer 要用 opencode 的話,需先安裝 opencode CLI,且所選
  provider 已 `opencode auth login`。
- 建 PR 需要 `gh` 已登入;純本地 repo 也能用,committer 會跳過
  push/PR 並註明原因。

## 首次初始化:`/cowork:init`

在目標專案下執行,問答收集參數:

| 參數 | 說明 |
|---|---|
| `main_branch` | 主分支名(自動偵測後請你確認) |
| `implementer` | 後端:`claude` / `codex` / `opencode` |
| `implementer_model` | 模型:依後端選(見上表) |
| commit gates | 依序執行的測試指令(如 `make test`),全過才能 commit |
| gate 注意事項 | gate 之間的坑(如「unit test 後要重編 production binary」) |
| `approval` | `ask`:branch/commit/PR 都先問你(預設)/ `auto`:全自動 |
| 語言 | commit/PR 語言、對話語言 |
| 審查提醒 | review 時要人工特別檢查的改動類型(如 lock-free) |

init 會生成:

- `.claude/cowork.md` — 專案參數,**之後改參數直接編輯它即可**
- `WORKFLOW.md` — 協定規格 snapshot(檔案格式、STATUS 欄位、版本規則)
- implementer 選 codex 或 opencode 時,把協定段落合併進
  `AGENTS.md`(append,不覆寫既有內容;外部 CLI 只讀得到 repo 內
  的檔案,選 claude 時不需要)
- `artifacts/` 加入 `.gitignore`

最後跑 doctor 檢查(git repo、gh 登入、codex plugin)。

## 日常使用:`/cowork:task-loop`

1. **首次**:提供工作範圍,skill 會生成 `root_plan.md` 任務清單,
   停下來等你確認拆分合理才開跑。
2. **每一輪**(一個任務):
   - 提 branch 名(`approval: ask` 時等你核准)→ branch-setup 開分支
   - Claude Code 寫 `artifacts/task-<id>/plan.md`
   - 委派 implementer 實作 + 跑 gates
   - Claude Code review(gates 逐項驗、audit diff 是否符合摘要),
     有問題就退回修正,直到 `STATUS: PASS`
   - `approval: ask` 時列 commit message / PR 標題等你核准 →
     committer 執行 commit / push / PR
   - root_plan.md 打勾,進下一個任務
3. 中斷後說「continue task loop」即可從第一個未勾選任務接續。

## 切換 implementer / 模型

編輯 `.claude/cowork.md` 的 `implementer` 與 `implementer_model`
欄位即可,下一次委派生效:

- **只換模型**(如 opus → sonnet、gpt-5.6 → gpt-5.5):改
  `implementer_model` 一行就好。codex 後端若 `/codex:rescue` 不支援
  指定模型,要同步改 `.codex/config.toml` 的 `model`。
- **codex / opencode → claude**:直接改,AGENTS.md 的協定段落留著
  無害。
- **claude → codex / opencode**:改完要重跑 `/cowork:init` 讓
  AGENTS.md 補上協定段落(task-loop 委派前會檢查,缺了會擋下來
  提示)。
- 任務進行到一半也能換——協定狀態都在 `artifacts/task-<id>/` 檔案
  裡,新實作端從 plan.md / implement.md 接手;但舊實作端的 thread
  記憶帶不過去,建議在任務邊界切換。

## 結構

```
.claude-plugin/     plugin + marketplace manifest
skills/init/        初始化 skill 與檔案模板(templates/)
skills/task-loop/   任務迴圈 skill(含 implementer adapter)
agents/             branch-setup、committer、file-search(sonnet)
                    implementer(implementer=claude 時用,預設
                    opus,委派時依 implementer_model 覆寫)
```

## 升級

skills 與 agents 隨 plugin 更新自動生效。寫進各 repo 的
`WORKFLOW.md` / `AGENTS.md` 協定段落是帶版本標記
(`cowork-kit-version`)的 snapshot——plugin 改版後在該 repo 重跑
`/cowork:init` 即可升級,init 會保留原參數、只換協定內容。
