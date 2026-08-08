---
name: init
description: 初始化 cowork task-loop 工作流——問答收集專案參數(主分支、
  commit gates、planner、implementer、核准政策、語言),生成
  .claude/cowork/config.md 設定檔與 WORKFLOW.md,任一角色選 codex/opencode
  時把協定段落合併進 AGENTS.md,最後跑環境檢查。當使用者說
  「cowork init」「初始化 task-loop」或首次在專案使用 cowork 時觸發。
---

本 skill 目錄下的 `templates/` 是所有生成檔案的來源,填入參數時只做
`{{PLACEHOLDER}}` 替換,不要改寫模板其他內容。

## 步驟 0:偵測既有狀態

- 若 `.claude/cowork/config.md` 已存在:讀取其 `cowork-kit-version` 標記。
  - 版本與本 plugin 模板相同 → 告知使用者已初始化過,問是否要重新
    問答覆寫設定,不要就結束。
  - 版本較舊 → 這是升級:保留使用者原本的參數值,用新模板重新生成
    設定檔與 WORKFLOW.md(以及 AGENTS.md 協定段落),列出協定變更
    給使用者確認後才寫入。
- 若只有**舊版位置**的檔案(`.claude/cowork.md`、根目錄
  `WORKFLOW.md` / `root_plan.md` / `artifacts/`,kit-version ≤ 2
  的舊配置):視為升級,參數照上一條處理,並把 root_plan.md 與
  artifacts/ 搬進 `.claude/cowork/`、刪除根目錄的 WORKFLOW.md 與
  舊 `.claude/cowork.md`,`.gitignore` 裡的舊 `artifacts/` 條目
  換成新路徑。
- 若不在 git repo 內,先告知並詢問是否 `git init`。

## 步驟 1:收集參數(AskUserQuestion)

1. **main_branch**:先用 `git symbolic-ref refs/remotes/origin/HEAD`
   或現有分支偵測,偵測結果當預設值請使用者確認。
2. **planner(後端 + 模型)**:規劃/審查/錯誤診斷角色,兩段式
   問答——
   - 後端:`claude`(plugin 內建 planner subagent)、`codex`
     (透過 /codex:rescue 委派)、`opencode`(透過 opencode CLI
     委派)。
   - 模型(依後端給選項):claude → `fable`(預設)、`opus` 或
     `sonnet`;codex → `gpt-5.6`(預設)或 `gpt-5.5`;opencode →
     完整 `provider/model` 字串。
   寫進 `planner` 與 `planner_model` 欄位。
3. **implementer(後端 + 模型)**:實作角色,同上兩段式問答——
   後端選項相同(claude 用 plugin 內建 implementer subagent);
   模型:claude → `opus`(預設)或 `sonnet`;codex / opencode 同
   planner 的格式。寫進 `implementer` 與 `implementer_model` 欄位。
   **planner 與 implementer 不可同時選 codex**(/codex:rescue 的
   --resume 只有一條 thread,兩角色會互相搶線),使用者這樣選時
   說明原因並請改其中一個。
4. **commit gates**:請使用者提供依序執行的測試/檢查指令(至少一個,
   例如 `make test`、`npm test`)。同時詢問有沒有 gate 之間的注意
   事項(例如「跑完 unit test 要重編 production binary」「e2e 失敗
   先跑某恢復程序再重試」),沒有就填「(無)」。
5. **approval**:`ask`(branch 名/commit/push/PR 都先經使用者核准,
   預設)或 `auto`(全自動,task-loop 不停下來問;planner 的
   decision.md 問題與「是否進入實作」不受此影響,一律會問)。
6. **commit_language / chat_language**:預設 commit/PR 英文、對話
   繁體中文,請使用者確認。
7. **專案特有審查提醒**:review 時需要人工特別檢查的改動類型(例如
   lock-free、加密邏輯),沒有就填「(無)」。

## 步驟 2:生成檔案

1. `templates/cowork-config.md` 填入參數 → 寫到 `.claude/cowork/config.md`。
   gates 以 markdown 有序清單呈現(`1. <指令> — <一句話說明>`)。
2. `templates/WORKFLOW.md` → 複製到 `.claude/cowork/WORKFLOW.md`。
   已存在且非 cowork 生成(沒有版本標記)時,先問過使用者才覆寫。
3. 把 `.claude/cowork/artifacts/` 與 `.claude/cowork/root_plan.md`
   加進 `.gitignore`(已有就跳過)。這是任務執行的工作檔;config.md
   與 WORKFLOW.md 則建議進版控,團隊成員 clone 後不用重新 init。
4. **僅當 planner 或 implementer 任一為 codex 或 opencode**(外部
   CLI 只讀得到 repo 內的檔案):
   - `templates/AGENTS-section.md` 合併進專案根目錄 `AGENTS.md`:
     檔案不存在就直接建立;已存在則 **append 到檔尾**,不可覆寫既有
     內容;已有舊版 cowork 段落(認版本標記)則只替換該段落。
   - 確認 `CLAUDE.md` 引用了 `@AGENTS.md`:沒有 CLAUDE.md 就建立
     一行套殼(`@AGENTS.md`);已有 CLAUDE.md 但沒引用就在開頭加上。
   - 兩個角色都是 claude 時完全跳過本步驟,不動 AGENTS.md。

## 步驟 3:環境檢查(doctor)

逐項檢查並回報,失敗不中止、列出修法:

- 在 git repo 內、`main_branch` 分支存在。
- `gh auth status` 通過(committer 建 PR 需要;無 remote 的純本地
  repo 可標記為「略過,之後要建 PR 再處理」)。
- planner 為 claude 且 planner_model 為 fable 時:提醒使用者確認
  訂閱方案有 fable 可用,沒有就改 opus。
- planner 與 implementer 同時為 codex 時:直接回報錯誤請使用者改
  其中一個(--resume 單 thread 衝突,見步驟 1)。
- 任一角色為 codex 時:openai-codex plugin 已安裝
  (`/codex:rescue` 可用)。若 /codex:rescue 不支援指定模型,
  該角色的 model 需自行設定在 `~/.codex/config.toml` 的
  `model` 欄位,init 提醒使用者確認兩者一致。
- 任一角色為 codex 時,額外檢查以下 **sandbox 已知陷阱**
  (實測踩過,不是假設性提醒),逐項回報現況:
  1. **companion 每輪強制 `sandbox=workspace-write`**——plugin 的
     `codex-companion.mjs` 開 thread 時硬編傳入 sandbox 參數,repo
     `.codex/config.toml` 的 `danger-full-access` 會被蓋掉、無效。
     若任務需要 sandbox 外的能力(Unix socket bind、`.git` 寫入、
     SSH 等),companion script 需 patch 成支援
     `CODEX_COMPANION_SANDBOX` 環境變數覆寫,且 **plugin 更新後
     patch 會被蓋掉,要重新套用**。檢查方式:grep companion script
     裡有沒有 `CODEX_COMPANION_SANDBOX`。
  2. **workspace-write 下預設沒有網路**——若 gate 需要對外連線
     (loopback、測試容器),確認 `~/.codex/config.toml` 有:
     ```toml
     [sandbox_workspace_write]
     network_access = true
     ```
  3. **codex app-server 是常駐進程,config 只在啟動時讀一次**——
     改過 `~/.codex/config.toml`(包含上面兩項)之後,要 kill 掉
     broker(`app-server-broker.mjs`)與 `codex app-server` 進程,
     下一輪委派才會拉到新設定。
  4. **可寫範圍 = companion 啟動時的 cwd**——委派前工作目錄必須是
     目標 repo 根目錄,否則 codex 連 repo 檔案都寫不了。
  以上涉及 `~/.codex/config.toml` 的修正,**doctor 不要代寫**
  (home 目錄設定檔常被權限系統擋,擋下後重試只是浪費一輪)——把
  要加的內容原文列出,請使用者手動加或明確核准後再寫。
- 任一角色為 opencode 時:`opencode` CLI 已安裝(`opencode
  --version`),且該角色 model 的 provider 已完成認證
  (`opencode auth login`)。
- **權限預放行**:task-loop 執行中常見指令若沒放行,每輪都會跳
  permission prompt 或被 auto 模式擋下(委派 codex/opencode 的
  Bash 呼叫、gh 唯讀查詢如 `gh pr view` / `gh pr checks` 等)。
  列出建議的 `permissions.allow` 清單,詢問使用者要不要寫進專案
  `.claude/settings.local.json`(要才寫,不要代決定)。

## 步驟 4:總結

列出生成/修改的檔案清單、選定的參數,並提示:
- 之後改參數直接編輯 `.claude/cowork/config.md`,不用重跑 init;只要換
  planner / implementer 的後端或模型可以用 `/cowork:model`。
- plugin 升級後重跑 `/cowork:init` 可升級 repo 內的協定 snapshot
  (WORKFLOW.md 與 AGENTS.md 段落)。
- 開始跑任務用 `/cowork <工作範圍>`,接續用 `/cowork 繼續`。
