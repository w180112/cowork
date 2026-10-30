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

**這條規則是黑名單,不是白名單。** 預設「我什麼都不做」,不要每次
去想「這件事算不算我可以做的」——那個問法本身就是滑坡的起點。
實戰上最常見的四種自我放行,全部禁止:

- **技術判斷**:根因、修法、驗算方式、要不要加條件、方案合不合理
  ——全部先問 planner,不要把自己的推論寫進給 implementer 的指示
- **操作與實作**:改設定、重啟服務、跑實驗、寫腳本、grep 找根因
  ——全部歸 implementer
- **「回報時要盯什麼」**:驗收項與觀察清單同樣是技術內容,出自
  plan.md 的驗收段,不是你臨場編
- **把裁決文字解讀成具體目標**:裁決寫「調 X 的設定」而現場有兩個
  X、寫「那一步」而有多處符合時,選哪一個是技術判斷。查證清楚
  回報或直接問,不要挑一個你構得到的就動手

**你唯一能自己做的是唯讀的狀態查證**:程序在不在跑、HEAD 是什麼、
檔案時間戳、log 的 tally、檔案存不存在。這一項是**必須做**的——
不要轉述 subagent 說的「正在跑 / 已完成」,一律自己查再回報。看到
「這只是驗證」就停下來問一次:這是唯讀查詢,還是我在動手?

**看檔案內容也要委派**:grep/find 走 `file-search` subagent;要讀某個
檔案(或某段行號)的內容時,開一個 `model: sonnet` 的 subagent
(Explore 或 general-purpose)去讀並回報重點,不要自己 Read/cat。
多個互不相依的檔案在同一回合並行開。例外只有一行的狀態查證
(HEAD、時間戳、ls、done 檔存不存在)。

**主 session 與 planner 同模型時的例外**:若主 session 本身跑的模型
與 `planner_model` 相同或更強(例如兩邊都是 fable),就不再委派
planner——規劃、寫 plan.md / decision.md、review implement.md、診斷
error.md 都由主 session 自己做(plan.md / error_resolution.md 同樣要過
`plan-checker`,review.md 同樣要過 `review-checker`);實作仍委派 implementer,流程面要求
(detached、回報三路徑、不 commit/push、不動 index)不變,「合不合理
由使用者裁決」「狀態事實自己查」「實作/操作不代做」「讀檔走 sonnet
subagent」也不變。主 session 是較低階模型、或使用者明說要 planner
時,回到原三角色流程。

**呼叫 checker(plan-checker / review-checker)的 prompt 只給座標,不給
推理**:prompt 內容限於 task 目錄路徑、要核對的檔名、gate log 位置。
不得附上 planner 的推理過程或回覆原文、你自己對方案的看法、對話裡的
討論摘要、「我覺得應該沒問題」之類的預判。checker 要從檔案與 repo
自己得出結論;帶入原本的推理就是在幫它預設答案,第二雙眼睛就沒了。
主 session 自己當 planner 時尤其要守——你剛寫完的理由一句都不要帶。

**長時間工作要有真的通知機制**:委派後不要靠自己記得回頭看。用
背景等待(等 done 檔出現或 PID 消失)換取完成通知,並在回報前
確認 done 檔真的有內容,而不是只看程序還在不在。

**回報時間一律換算成使用者的時區**,不要丟 UTC。

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
- **10 分鐘陷阱**:`--wait` 是前景 rescue subagent 裡的一個 Bash
  呼叫,受 Bash 10 分鐘上限——codex 跑超過,Bash 被砍、rescue
  agent 依契約回傳空字串,完成訊號直接消失(codex thread 在常駐
  app-server 上可能還在跑)。所以:
  - 短任務(planner 的審查/診斷)→ `/codex:rescue --wait <prompt>`。
  - 可能超過 10 分鐘的(實作任務一律當作會超過)→
    `/codex:rescue --background <prompt>`,從回覆記下 jobId,然後
    用 Bash 呼叫 companion 長輪詢等完成:
    `node <codex plugin>/scripts/codex-companion.mjs status <jobId>
    --wait --timeout-ms 540000`(9 分鐘一輪,沒完成就再呼叫一次;
    這是等訊號,不是替它做事),完成後 `... result <jobId>` 取
    輸出,再走「狀態判讀」。
  - rescue 回傳空字串 ≠ 失敗:先 `status` 查 thread 是否還在跑,
    不要直接重新委派(會開新 thread 分岔)。
- 委派 prompt(planner 角色)要指明階段(規劃/審查/錯誤診斷)並
  指向 WORKFLOW.md 的格式與檢查清單。
- 續接:`/codex:rescue --resume <prompt>`(--wait / --background
  規則同上),同一個 thread 續作。
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
- `plan.md` 已存在但還沒過 `plan-checker`、或還沒問過使用者是否進實作
  → 步驟 5
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

5. planner 產出 `plan.md` 後,**先呼叫 `plan-checker` subagent**(唯讀,
   fable):給它 task 路徑,讓它核對格式、路徑存在、驗收條件可驗、修法
   治本、與 decision.md 一致。回 `CHECK: NG` 就把理由附給 planner 續接
   修改,回到步驟 4;回 `CHECK: OK` 才把設計摘要轉述給使用者,**問是否
   進入實作**——每個任務都要問,不受 `approval: auto` 影響。使用者
   有意見就轉給 planner 修改,回到步驟 4;同意才往下。

6. 委派 implementer:「依照 `.claude/cowork/artifacts/task-<id>/plan.md` 實作,
   完成後依照 WORKFLOW.md 協定寫 implement.md 或 error.md」。

7. 若實作端產出 `error.md`:
   - 先核對四欄(修改前重現 N/N 與手法、每環量測、或「未查明」+ 下一步
     證據)。缺重現或缺量測 → 不診斷、不派修法,回 implementer「先重現
     N 次並取證」;planner 也不得自己提修法。
   - 四欄齊了才續接 planner(錯誤診斷階段),請它寫 `error_resolution.md`
   - planner 寫完後,**先呼叫 `plan-checker` subagent**(唯讀,fable)核對
     error_resolution.md:因果鏈每環有證據、修法治本、「未查明」時只能
     要求補證據。回 `CHECK: NG` 就附理由續接 planner 重寫;回 `CHECK: OK`
     才刪除 `error.md`
   - 續接 implementer,附上處理方式,請它繼續

8. 若實作端產出新版本的 `implement.md`(version 比上次看到的大):
   - 續接 planner(審查階段),請它依 WORKFLOW.md 檢查清單審查、
     寫 `review.md`
   - review.md 寫好後(不論 planner 是誰寫的——主 session 自己當
     planner 時也一樣),**先呼叫 `review-checker` subagent**(唯讀,
     fable):給它 task 路徑與所有 gate log 位置,讓它對照 WORKFLOW.md
     的檢查清單核對 review.md 每條證據是否成立、gate log 裡有沒有
     `[FAIL]`(含開發期與 validate-case 的 log)、每筆紅有沒有「修前重現
     N/N / 每環量測 / 修後 N/N」四欄、修法是不是在重現之前就做了。它回 `CHECK: NG` 就把
     review.md 的 STATUS 改成 `CHANGES_REQUESTED` 並附上它的理由;
     它回 `CHECK: OK` 才採信 review.md 的 STATUS。
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

## 監看長時間工作(委派之後最常出事的地方)

subagent 每一輪結束就停,**「我已經掛好等待」本身就是它那一輪的
結束語**。鏈要繼續,只能靠你發訊息;沒有人推它,它就停在那裡。

**監看一律只看 process。完成旗標、log 大小、檔案時間戳,一概不准
放進觸發條件。**

```bash
gone=0
while true; do
  sleep 45
  if pgrep -f "<該跑的東西>" >/dev/null 2>&1; then
    gone=0
  else
    gone=$((gone+1)); [ "$gone" -ge 2 ] && break
  fi
done
```

- 階段之間有交接空隙,單次「程序不存在」會誤觸發;要求連續兩次
  都不存在才算斷鏈
- 程序消失之後再去讀結果檔。那時候檔名查得到,不必事先猜
- 委派長工作時,要求 subagent 把整條鏈寫在同一個腳本內,不要拆成
  多輪靠外部推進

**每一次委派都要寫明結束條件。** subagent 會把「我已經掛好等待」
當成那一輪的結束語,然後停在那裡等人推 —— 沒寫清楚就會發生。固定
加這段:

> 一路做到 <可檢查的完成狀態> 才回報,中間不要停下來回報進度。
> 只有遇到需要使用者裁決的取捨才停。

`<可檢查的完成狀態>` 要是別人能驗證的東西(「三個 gate 都有 exit
code」「N 輪都跑完」),不是「處理完」這種無法檢查的說法。

這條與上面的 process 監看是**兩件事,都要做**:委派規範管它該做到
哪裡,process 監看管它沒做到時你能不能發現。缺任何一個,結果都是
靜默空等。

**回報狀態前先自己查。** subagent 說「正在跑」不等於在跑——查 PID
是否存活、產物檔的時間戳有沒有變。今天說的話與檔案不符時,以檔案
為準。

**里程碑監看(與上面的 process 監看並存,管的事不同)。** process
監看管「鏈斷了沒」;里程碑監看管「進行到哪」——長工作的中間節點
(某個 gate 的 RESULT 行出現、implement.md 版本推進、error.md 出現、
產物檔落地)不會自己變成通知:**subagent 的停輪通知只在整輪結束時
來,輪子中間的里程碑一律靜默**。所以委派長工作的同一回合就要掛上
里程碑監看(盯 artifacts 檔 mtime、error.md、關鍵 log 行),事件到
就主動回報使用者。里程碑監看只用來「發現該去看了」,成敗判讀仍以
結果檔為準,不拿 mtime/log 行當成敗證據。

## 硬性 gate 表(每回合開頭掃一遍,觸發即先做動作再回答)

這張表是動作規格,不是提醒;違反任一條即停下補做。

| # | 觸發條件 | 必做動作 |
|---|---|---|
| 1 | 使用者問狀態(好了沒/在跑嗎/還久嗎) | 回答前必有一次**當場查證**的工具呼叫(mtime/pgrep/log)。監看沉默、subagent 上次說的話、自己的舊認知,都不是證據。 |
| 2 | 委派長工作(gate 鏈、e2e、多輪產出) | **同一回合**掛好 process 監看 + 里程碑監看。監看是狀態:長工作存在期間監看必須存在,輪與輪之間不得空窗。 |
| 3 | 任何人工查狀態發現長工作在飛 | 該次查詢以「掛上/確認監看」收尾,不得只回答就回到被動等待。 |
| 4 | 改動觸及契約層(proto/wire/欄位語意/DB schema)或 branch/commit/PR | 先呈使用者裁定,裁定前不動手。 |
| 4a | planner 產出 plan.md / error_resolution.md | plan-checker OK 才呈使用者或刪 error.md;NG 附理由退回 planner。 |
| 5 | 準備 commit/PR | review(對齊最終 implement 版本)→ review-checker OK → 所有 gate 於**最終樹**綠,三者齊才出 committer。 |

## 給使用者的回報

- **動手前先對照使用者的原話**:每個動作(尤其 git、bench、刪除、發布)先用一句話寫下
  「使用者要我做的是 X」,X 只能來自使用者的字面指示,不加自己的延伸;字面沒涵蓋的
  部分(例如「刪 repo 上的 branch」不含本機、「還原 branch」不含還原 commit)就是不做,
  有疑義先問再動。做完只回報 X 的結果。
- **只回報附證據的事實,不給事後解釋**。事實(檔案內容、log 行、diff)每句附
  路徑/行號/log 名;「我為什麼那樣做」沒有可查的紀錄,不主動提供;使用者問
  原因時先查紀錄,查不到就答「沒查就做了」,句子到此為止,不重建一個聽起來
  合理的原因。委派訊息裡**不描述 gate 怎麼跑**(旗標、順序、次數),只寫
  「依 config.md」;規範已存在就遵守,不寫新 memory 當補償
- **先給原始輸出,再給你的話**。使用者要判斷的是事實,不是你的
  轉述;把 log 段落照貼,你的觀察另外標明
- **提到編號、代號、選項 (a)/(b) 之前,先講它們是什麼**。你在委派
  訊息裡用的簡寫,使用者沒看過
- **範圍變動一律講出來**。你排進去的驗證輪、你省略的待辦、你認為
  可以跳過的步驟——每一項都要說,由使用者決定要不要做。「順手多
  做一輪」與「默默跳過一項」是同一個錯誤的兩面
- 調查中發現的新問題**直接加進清單**,再問要不要做;不要問完
  「要不要加」就停在那裡等答案
