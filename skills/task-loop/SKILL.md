---
name: task-loop
description: 依 root_plan.md 執行雙 agent 任務迴圈——Claude Code 負責
  規劃、code review、audit 與 commit 交接,實作委派給 .claude/cowork.md
  指定的 implementer(codex 或 claude subagent)。當使用者說「開始跑
  task loop」「處理下一個任務」「continue task loop」或提到
  root_plan.md 時觸發。
---

執行前務必先讀取專案根目錄的 `WORKFLOW.md` 與 `.claude/cowork.md`,
協定細節以 WORKFLOW.md 為準、專案參數(main_branch、implementer、
gates、approval、語言)以 cowork.md 為準。兩者缺一,請使用者先跑
`/cowork:init`。本檔案只描述你(Claude Code / 規劃審查角色)的執行
步驟。

## 初次執行(root_plan.md 不存在時)

依使用者提供的工作範圍,產生 `root_plan.md`,格式:

```markdown
# Root Plan

## Memory / Policy 摘要
(相關的專案慣例、記憶重點)

## 任務清單
- [ ] task-1: <一句話描述>
- [ ] task-2: <一句話描述>
```

產生後,把任務清單列出來請使用者確認拆分是否合理、有沒有遺漏或需要
調整的地方,**停在這裡等使用者明確回覆要開始才進入下面「每一輪的
執行步驟」**——不要生完清單就自動接著跑第一個任務。除非使用者一開始
就已經明確要求「不用確認、直接開始跑」,才可以跳過這個停頓點。

## Implementer adapter

步驟裡所有「委派實作端」「請實作端修正」的動作,依 cowork.md 的
`implementer` 值走對應做法:

`implementer` 與 `implementer_model` 可以隨時在 cowork.md 裡改,
下一次委派生效。任務進行到一半換人也可行——協定狀態都在
`artifacts/task-<id>/` 的檔案裡,新的實作端從 plan.md /
implement.md 接手即可,但舊實作端的工作記憶(thread context)不會
跟過去,建議盡量在任務邊界切換。

外部 CLI 後端(codex、opencode)委派前,先確認 AGENTS.md 含有
cowork 協定段落(認 `cowork-kit-version` 標記)——從 claude 切換
過來時會缺,缺了就請使用者跑 `/cowork:init` 補上,不要在沒有協定
段落的情況下委派。這兩種後端的協定指令都由 AGENTS.md 載入,prompt
只需指向 plan.md / review.md 內容,不用重述協定。

**implementer: codex**
- 委派:`/codex:rescue --wait <prompt>`(嚴格序列執行,Codex 跑完
  之前你沒有其他事可做,直接阻塞到完成,不要自己輪詢)。
- 續接(error 處理、review 修正):`/codex:rescue --resume --wait
  <prompt>`,同一個 thread 續作。
- 模型:若 /codex:rescue 支援指定模型就帶入 `implementer_model`;
  不支援則以 `.codex/config.toml` 的 `model` 為準,發現兩者不一致
  時提醒使用者。

**implementer: claude**
- 委派:Agent tool 呼叫 `implementer` subagent,
  `run_in_background: false`,`model` 參數帶入 `implementer_model`
  (opus 或 sonnet),prompt 附上 task 路徑
  `artifacts/task-<id>/`。協定指令在 agent 定義裡,不用重述。
- 續接:對同一個 agent 用 SendMessage 附上 review.md 或
  error_resolution 內容,保留其既有 context;不要另開新 agent。

**implementer: opencode**
- 委派:Bash 執行
  `opencode run --model <implementer_model> "<prompt>"`
  (阻塞到完成;model 用完整 `provider/model` 字串,如
  `zai/glm-5.2`)。
- 續接:`opencode run --continue --model <implementer_model>
  "<prompt>"`,續用上一個 session。
- opencode 會自動載入專案的 AGENTS.md,協定指令不用重述。

## 每一輪的執行步驟

1. 從 root_plan.md 找第一個未勾選的任務,取得 `task-<id>`。若該任務
   標記為「audit(Claude Code 直接執行)」,不走實作端委派流程——
   由你自己執行稽核、把發現寫成報告給使用者,結論如需修 code 再拆成
   新的實作任務加進 root_plan.md。

2. 決定這個任務的 branch 名稱。`approval: ask` 時先提給使用者核准;
   `approval: auto` 時直接採用。然後呼叫 `branch-setup` subagent,
   傳入 branch 名稱與 cowork.md 的 `main_branch`,由它執行
   checkout / pull / 開新分支。

3. 建立 `artifacts/task-<id>/`,依 WORKFLOW.md 的格式寫 `plan.md`。

4. 依 implementer adapter 委派實作:「依照
   `artifacts/task-<id>/plan.md` 實作,完成後依照 WORKFLOW.md 協定
   寫 implement.md 或 error.md」。

5. 若實作端產出 `error.md`:

   - 判斷原因,把處理方式寫進
     `artifacts/task-<id>/error_resolution.md`(或直接更新 plan.md
     補充說明)
   - 寫完才刪除 `error.md`
   - 依 adapter 續接,附上處理方式,請實作端繼續

6. 若實作端產出新版本的 `implement.md`(version 比上次看到的大):

   - 讀取內容,進行 code review,並 audit「變更摘要」是否真的符合
     「檔案異動清單」與實際 diff
   - **確認 cowork.md「Commit gates」清單裡的每一個 gate 都有
     通過**:implement.md 的測試結果必須逐項涵蓋且全部 pass,缺一個
     就是 CHANGES_REQUESTED
   - **確認沒有修改既有測試 case**(只能新增——既有 case 是
     regression baseline;若懷疑既有 case 本身有錯,應該出現在
     implement.md 的說明裡回報,而不是被改掉)
   - cowork.md「專案特有審查提醒」有列的改動類型,不能只看測試綠燈,
     務必親自檢查邏輯正確性
   - 有問題:寫 `review.md`(`STATUS: CHANGES_REQUESTED`,列出具體
     修改項目),依 adapter 續接附上 review.md 內容請實作端修正,
     回到步驟 5 繼續處理
   - 沒問題:寫 `review.md`(`STATUS: PASS`)

7. STATUS: PASS 後的 commit 交接:`approval: ask` 時,先把預計的
   commit message 摘要與 PR 標題列給使用者核准才能繼續;
   `approval: auto` 時直接進行。呼叫 `committer` subagent,傳入
   `artifacts/task-<id>/implement.md` 的內容與 branch / commit / PR
   資訊,由它執行 commit、push、建立 PR。

8. 在 `root_plan.md` 把該任務打勾,回到步驟 1 處理下一個未完成任務,
   直到全部完成。`approval: ask` 時每一輪的步驟 2、7 都要重新取得
   核准,前一輪的同意不延用。
