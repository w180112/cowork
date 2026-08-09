---
name: task-loop
description: 依 root_plan.md 執行三角色任務迴圈——主 session 只做
  調度(挑任務、開分支、委派、轉發檔案、取得核准),規劃/審查/錯誤
  診斷委派給 planner,實作委派給 implementer(後端都在
  .claude/cowork/config.md 設定)。當使用者說「開始跑 task loop」「處理
  下一個任務」「continue task loop」或提到 root_plan.md 時觸發。
---

執行前務必先讀取`.claude/cowork/WORKFLOW.md` 與 `.claude/cowork/config.md`,
協定細節以 WORKFLOW.md 為準、專案參數(main_branch、planner、
implementer、gates、approval、語言)以 config.md 為準。兩者缺一,
請使用者先跑 `/cowork:init`。

**你(主 session)是調度端,不是思考端,也不是執行端。** 你只做:
挑任務、開分支、委派 planner / implementer、在兩者與使用者之間
轉發檔案內容、取得核准、比對 version 與 STATUS 字串。設計、code
review、error 診斷一律委派給 planner;跑測試/gate、改 code、git
操作一律由對應 subagent 自己執行——即使你覺得自己代跑比較快也
不要代勞。subagent 被權限或 sandbox 擋住時,解法是**排除障礙後讓
它自己重跑**(修 sandbox 設定、請使用者放行權限或手動執行環境層
修改),不是你接手執行。這個工作流的前提是主 session 可能跑在
低階模型上,你代跑的結果不可信。

## 初次執行(root_plan.md 不存在時)

把使用者提供的工作範圍委派給 planner(規劃階段,不帶 task id),
請它產生 `.claude/cowork/root_plan.md`(下文簡稱 root_plan.md),格式:

```markdown
# Root Plan

## Memory / Policy 摘要
(相關的專案慣例、記憶重點)

## 任務清單
- [ ] task-1: <一句話描述>
- [ ] task-2: <一句話描述>
```

產生後,把任務清單列出來請使用者確認拆分是否合理,**停在這裡等
使用者明確回覆要開始才進入下面「每一輪的執行步驟」**。使用者要調整
就把意見轉給 planner 修改。除非使用者一開始就明確要求「不用確認、
直接開始跑」,才可以跳過這個停頓點。

## 後端 adapter(planner 與 implementer 共用)

planner 依 cowork.md 的 `planner` / `planner_model` 選後端,
implementer 依 `implementer` / `implementer_model`,委派與續接方式
相同,只有 prompt 內容不同。兩個欄位都可隨時改,下一次委派生效——
協定狀態都在 `.claude/cowork/artifacts/task-<id>/` 的檔案裡,新後端從檔案接手
即可,但舊後端的工作記憶(thread context)不會跟過去,建議在任務
邊界切換。

外部 CLI 後端(codex、opencode)委派前,先確認 AGENTS.md 含有
cowork 協定段落(認 `cowork-kit-version` 標記)——缺了就請使用者跑
`/cowork:init` 補上。**planner 與 implementer 不要同時設 codex**:
`/codex:rescue --resume` 只會續接最後一個 thread,兩個角色會互相
搶線(init doctor 會擋)。

**完成訊號(三個後端通用)**:委派或續接送出後,若沒有同步拿到
結果(工具回報的是背景 task),就結束這一輪回覆、等 task 完成
通知——不要輪詢、不要當作已完成往下走、也不要在等待時先做後面的
步驟。被喚醒後一律先走「狀態判讀」,以 artifacts 檔案為準,不要
只信 agent 回覆的文字。

**後端 claude**
- 委派:Agent tool 呼叫對應 subagent(planner 角色用 `planner`、
  implementer 角色用 `implementer`),`run_in_background: false`,
  `model` 參數帶入該角色的 model 欄位,prompt 附上階段說明與 task
  路徑 `.claude/cowork/artifacts/task-<id>/`。協定指令在 agent 定義裡,不用重述。
- 續接:對同一個 agent 用 SendMessage(非同步,走「完成訊號」
  規則),保留其既有 context;不要另開新 agent。

**後端 codex**
- 委派:`/codex:rescue --wait <prompt>`(嚴格序列執行;底層是
  codex-rescue subagent,可能以背景 task 執行——完成與否依上面
  「完成訊號」規則,不要假設呼叫回來就是做完了)。planner 角色的
  prompt 要指明階段(規劃/審查/錯誤診斷)並指向 WORKFLOW.md 的
  格式與檢查清單。
- 續接:`/codex:rescue --resume --wait <prompt>`,同一個 thread
  續作。
- 模型:若 /codex:rescue 支援指定模型就帶入該角色的 model 欄位;
  不支援則以 `~/.codex/config.toml` 的 `model` 為準,發現兩者不
  一致時提醒使用者。
- Sandbox:companion 開新 thread 會硬編 `workspace-write`,repo
  `.codex/config.toml` 的設定無效。任務需要 sandbox 外能力時,開
  新 thread 要帶 `CODEX_COMPANION_SANDBOX=danger-full-access`
  (前提:companion script 已 patch 支援;thread 權限開啟時固定,
  resume 不能升級,所以要在**第一次委派**就帶上)。委派連續失敗在
  權限類錯誤(socket bind、git 寫入、網路不通)時,先懷疑 sandbox
  而不是程式碼,對照 init doctor 的 sandbox 檢查項。
- **可寫範圍 = companion 啟動時的 cwd**:第一次委派前先確認目前
  工作目錄就是目標 repo 根目錄(必要時 `cd` 過去再委派),否則
  codex 會連 repo 檔案都寫不了。
- gate 或 git stash 被 sandbox 擋住(error.md 註明權限類障礙)時,
  **不要代跑**:對照 init doctor 的 sandbox 檢查項排除障礙(通常是
  `CODEX_COMPANION_SANDBOX=danger-full-access` 開新 thread 重新
  委派——sandbox 等級 resume 升不了級),然後讓實作端自己重跑。
- 帶 `CODEX_COMPANION_SANDBOX` 這類環境變數的委派、或寫入
  `~/.codex/config.toml`,可能被 Claude Code 權限系統擋下——被擋
  時不要重試,把要執行的指令原文列給使用者,請使用者手動執行或
  在權限設定放行。

**後端 opencode**
- 委派:Bash 執行
  `opencode run --model <該角色的 model> "<prompt>"`(阻塞到完成;
  model 用完整 `provider/model` 字串,如 `zai/glm-5.2`)。planner
  角色的 prompt 要指明階段並指向 WORKFLOW.md。
- 續接:`opencode run --continue --model <該角色的 model>
  "<prompt>"`,續用上一個 session。planner 與 implementer 都用
  opencode 時,`--continue` 只會接最後一個 session,續接前用
  `opencode session list` 確認、必要時帶 session id。
- opencode 會自動載入專案的 AGENTS.md,實作端協定指令不用重述。

## 狀態判讀(被喚醒或不確定進度時一律先做)

subagent 跑完的訊號可能漏接(task 通知、使用者隔了一陣子才回來、
context 被摘要過)。**協定的唯一事實來源是檔案,不是你的記憶或
agent 的回覆**——每次收到 task 完成通知、使用者說「繼續」、或你
不確定現在進行到哪裡時,先讀 `.claude/cowork/artifacts/task-<id>/`
再決定下一步:

- `error.md` 存在 → 步驟 7
- `implement.md` 的 version 比 root_plan / 上次紀錄的還大 → 步驟 8
- `decision.md` 是 `STATUS: WAITING` 且「回答」未填 → 步驟 4
- `plan.md` 已存在但還沒問過使用者是否進實作 → 步驟 5
- 檔案都沒變 → agent 可能還在跑或已結束卻沒寫檔:用 TaskOutput
  查該 agent 的狀態;已結束但協定檔沒產出 → 續接它,要求依
  WORKFLOW.md 把該寫的檔案補寫完成,不要自己代寫。

## 每一輪的執行步驟

1. 從 root_plan.md 找第一個未勾選的任務,取得 `task-<id>`。若該
   任務標記為「audit」,委派 planner 執行稽核並寫報告,把報告轉給
   使用者;結論如需修 code,請 planner 拆成新任務加進 root_plan.md,
   本輪到此為止。

2. 決定這個任務的 branch 名稱。`approval: ask` 時先提給使用者核准;
   `approval: auto` 時直接採用。然後呼叫 `branch-setup` subagent,
   傳入 branch 名稱與 cowork.md 的 `main_branch`。

3. 建立 `.claude/cowork/artifacts/task-<id>/`,委派 planner(規劃階段):prompt 附
   任務描述與 task 路徑,請它產出 plan.md 或 decision.md。

4. 若 planner 產出 `decision.md`(`STATUS: WAITING`):
   - 用 AskUserQuestion 把每個未回答的問題轉給使用者,選項與
     planner 的建議照檔案內容呈現
   - 把使用者的決定填進各題「回答」欄,STATUS 改成 `ANSWERED`
   - 續接 planner,回到本步驟開頭(可能來回多次)

5. planner 產出 `plan.md` 後,把它的設計摘要轉述給使用者,**問是否
   進入實作**——每個任務都要問,不受 `approval: auto` 影響。使用者
   有意見就轉給 planner 修改,回到步驟 4;同意才往下。

6. 委派 implementer:「依照 `.claude/cowork/artifacts/task-<id>/plan.md` 實作,
   完成後依照 WORKFLOW.md 協定寫 implement.md 或 error.md」。

7. 若實作端產出 `error.md`:
   - 續接 planner(錯誤診斷階段),請它寫 `error_resolution.md`
   - planner 寫完後,刪除 `error.md`
   - 續接 implementer,附上處理方式,請它繼續

8. 若實作端產出新版本的 `implement.md`(version 比上次看到的大):
   - 續接 planner(審查階段),請它依 WORKFLOW.md 檢查清單審查、
     寫 `review.md`
   - 你只看 review.md 的 `STATUS` 字串:
     - `CHANGES_REQUESTED` → 續接 implementer 附上 review.md
       內容請它修正,回到步驟 7 繼續處理
     - `PASS` → 進入步驟 9

9. commit 交接:`approval: ask` 時,先把預計的 commit message 摘要
   與 PR 標題列給使用者核准才能繼續;`approval: auto` 時直接進行。
   呼叫 `committer` subagent,傳入 `.claude/cowork/artifacts/task-<id>/implement.md`
   的內容與 branch / commit / PR 資訊。committer 看不到你與使用者
   的對話——需要敏感操作(amend、force push、改既有 PR)時,把
   「使用者已核准 <操作>」明確寫進委派 prompt,讓 committer 自己
   執行,不要因為它拒絕就由你代跑 git 指令。

10. 在 `root_plan.md` 把該任務打勾,回到步驟 1 處理下一個任務,
    直到全部完成。步驟 5 的「是否進入實作」每一輪都要問;
    `approval: ask` 時每一輪的步驟 2、9 也都要重新取得核准,前一輪
    的同意不延用。
