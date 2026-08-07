# cowork

Claude Code plugin:task-loop 三角色工作流。主 session 只做
**調度**(挑任務、開分支、委派、轉發、要核准),所以可以跑在低階
模型上;思考工作交給兩個可替換後端的角色:

- **planner** — 設計方案、寫 plan.md、code review、error 診斷
- **implementer** — 依 plan.md 實作、跑 commit gates

兩個角色的後端與模型都在 init 時選、之後可隨時改:

| 後端 | planner 模型 | implementer 模型 | 委派方式 |
|---|---|---|---|
| `claude` | fable(預設)/ opus / sonnet | opus(預設)/ sonnet | plugin 內建 subagent |
| `codex` | gpt-5.6(預設)/ gpt-5.5 | 同左 | openai-codex plugin `/codex:rescue` |
| `opencode` | 任意 `provider/model`,如 `zai/glm-5.2` | 同左 | opencode CLI `opencode run` |

限制:planner 與 implementer 不可同時用 codex(`/codex:rescue
--resume` 只有一條 thread,兩角色會搶線;init doctor 會擋)。

源自 fastrg-node 的 task-loop 協定,抽掉專案特定內容後參數化。

## 安裝

```
/plugin marketplace add /root/cowork
/plugin install cowork@cowork-marketplace
```

前提:

- 任一角色要用 codex 的話,需先安裝 openai-codex plugin;Codex
  sandbox 若需放寬(`.codex/config.toml` 等)屬環境層設定,init 只
  提醒不代管。
- 任一角色要用 opencode 的話,需先安裝 opencode CLI,且所選
  provider 已 `opencode auth login`。
- 建 PR 需要 `gh` 已登入;純本地 repo 也能用,committer 會跳過
  push/PR 並註明原因。

## 快速開始:`/cowork`

首次使用先跑 `/cowork:init`(問答選 planner / implementer 的後端
與模型)。之後 `/cowork <你要做的事>` 就是日常入口:

```
/cowork 幫我把登入改成 OAuth    # 生任務清單開跑(未 init 會提示先跑 /cowork:init)
/cowork 繼續                    # 從 root_plan.md 第一個未勾選任務接續
/cowork                         # 有 root_plan.md 視同繼續,沒有就問你要做什麼
```

## 首次初始化:`/cowork:init`

在目標專案下執行,問答收集參數:

| 參數 | 說明 |
|---|---|
| `main_branch` | 主分支名(自動偵測後請你確認) |
| `planner` + `planner_model` | 規劃/審查角色的後端與模型(見上表) |
| `implementer` + `implementer_model` | 實作角色的後端與模型(見上表) |
| commit gates | 依序執行的測試指令(如 `make test`),全過才能 commit |
| gate 注意事項 | gate 之間的坑(如「unit test 後要重編 production binary」) |
| `approval` | `ask`:branch/commit/PR 都先問你(預設)/ `auto`:全自動 |
| 語言 | commit/PR 語言、對話語言 |
| 審查提醒 | review 時要人工特別檢查的改動類型(如 lock-free) |

init 生成的檔案都集中在 `.claude/cowork/`:

- `.claude/cowork/config.md` — 專案參數,**之後改參數直接編輯它即可**
- `.claude/cowork/WORKFLOW.md` — 協定規格 snapshot(檔案格式、
  STATUS 欄位、版本規則、review 檢查清單)
- 執行期工作檔 `root_plan.md` 與 `artifacts/` 也放這裡,init 會把
  它們加進 `.gitignore`(config 與 WORKFLOW 建議進版控)
- 唯一例外:任一角色選 codex 或 opencode 時,協定段落合併進根目錄
  `AGENTS.md`(append,不覆寫既有內容)——外部 CLI 只會自動載入
  根目錄的 AGENTS.md,搬不進去;兩個角色都選 claude 時不需要

最後跑 doctor 檢查(git repo、gh 登入、codex plugin、codex 衝突)。

## 日常使用:`/cowork`

1. **首次**:提供工作範圍,planner 生成 `.claude/cowork/root_plan.md` 任務清單,
   停下來等你確認拆分合理才開跑。
2. **每一輪**(一個任務):
   - 提 branch 名(`approval: ask` 時等你核准)→ branch-setup 開分支
   - 委派 planner 設計;planner 需要你決定的問題會寫進
     `.claude/cowork/artifacts/task-<id>/decision.md`,主 session 轉給你選、把答案
     寫回去再讓 planner 繼續
   - planner 寫好 `plan.md` → 主 session 給你看設計摘要,**問你
     是否進入實作**(每個任務都問,`auto` 也不跳過)
   - 委派 implementer 實作 + 跑 gates
   - planner review(依 WORKFLOW.md 檢查清單),有問題退回修正,
     直到 `STATUS: PASS`
   - `approval: ask` 時列 commit message / PR 標題等你核准 →
     committer 執行 commit / push / PR
   - root_plan.md 打勾,進下一個任務
3. 中斷後說「continue task loop」即可從第一個未勾選任務接續。

## 切換角色後端 / 模型:`/cowork:model`

```
/cowork:model                        # 不帶參數:問答選角色、後端、模型
/cowork:model planner opus           # planner 換模型(claude 後端 fable → opus)
/cowork:model sonnet                 # 沒指明角色 → implementer,只換模型
/cowork:model implementer codex gpt-5.5   # implementer 換後端 + 模型
```

改的就是 `.claude/cowork/config.md` 的 `planner(_model)` /
`implementer(_model)` 欄位(手動編輯也一樣),下一次委派生效:

- **只換模型**:改 model 欄位一行就好。codex 後端若
  `/codex:rescue` 不支援指定模型,要同步改 `.codex/config.toml`
  的 `model`。
- **codex / opencode → claude**:直接改,AGENTS.md 的協定段落留著
  無害。
- **claude → codex / opencode**:改完要重跑 `/cowork:init` 讓
  AGENTS.md 補上協定段落(task-loop 委派前會檢查,缺了會擋下來
  提示)。
- 任務進行到一半也能換——協定狀態都在 `.claude/cowork/artifacts/task-<id>/` 檔案
  裡,新後端從 plan.md / implement.md / decision.md 接手;但舊後端
  的 thread 記憶帶不過去,建議在任務邊界切換。

## 結構

```
.claude-plugin/     plugin + marketplace manifest
skills/init/        初始化 skill 與檔案模板(templates/)
skills/task-loop/   任務迴圈 skill(含 planner/implementer 共用 adapter)
agents/             branch-setup、committer、file-search(sonnet)
                    planner(planner=claude 時用,預設 fable)
                    implementer(implementer=claude 時用,預設 opus)
                    兩者委派時都依 cowork.md 的 model 欄位覆寫
```

## 升級

skills 與 agents 隨 plugin 更新自動生效。寫進各 repo 的
`.claude/cowork/` 內的檔案與 `AGENTS.md` 協定段落是帶版本標記
(`cowork-kit-version`)的 snapshot——plugin 改版後在該 repo 重跑
`/cowork:init` 即可升級,init 會保留原參數、只換協定內容。
