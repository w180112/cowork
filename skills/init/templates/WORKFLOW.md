# Task-loop 協定規格

<!-- cowork-kit-version: 6(由 /cowork:init 生成,升級協定請重跑 init)-->

本文件是 task-loop 三個角色之間交接的唯一事實來源。實際的檔案格式、
狀態欄位、版本規則以本文件為準。

- **主 session(調度端)**:挑任務、開分支、委派下面兩個角色、轉發
  檔案、向使用者取得核准。不做設計、不做 review、不診斷錯誤。
- **planner(規劃審查端)**:設計方案、寫 plan.md 與 decision.md、
  review implement.md、診斷 error.md。後端與模型見
  `.claude/cowork/config.md` 的 planner 設定。
- **實作端(implementer)**:依 plan.md 改 code、跑 gates、寫
  implement.md 與 error.md。後端與模型見 cowork.md 的 implementer
  設定。

## 目錄結構

每個任務對應一個獨立資料夾,避免不同任務互相覆蓋、也留得住歷史紀錄:

```
.claude/cowork/artifacts/
└── task-<id>/
    ├── plan.md               # planner 產生,實作端的實作規格
    ├── decision.md           # planner 產生問題,主 session 填回答,保留當紀錄
    ├── implement.md          # 實作端產生,每次修改後版本號 +1
    ├── review.md             # planner 產生,一次性訊息,讀完即刪
    ├── error.md               # 實作端產生,主 session 處理完才刪除
    └── error_resolution.md    # planner 產生,error.md 的處理紀錄
```

`task-<id>` 使用 `.claude/cowork/root_plan.md` 任務清單裡的序號,例如 `task-3`。整個
`.claude/cowork/artifacts/` 目錄不進版控(/cowork:init 已加入 .gitignore)。

## plan.md 格式

planner 撰寫,內容至少包含:

- 目標與背景(為什麼要做這個任務)
- 涉及的檔案路徑清單
- 明確的驗收條件——具體到測試名稱或可驗證的行為描述,不要寫
  「功能正常」這種無法驗證的敘述
- 已知的邊界情況或限制

## decision.md 格式

planner 規劃途中需要使用者做決定時撰寫(方案取捨、規格不明確等;
能自行判斷的小事不要問)。主 session 把問題轉給使用者、把回答填回
「回答」欄後續接 planner。檔案保留不刪,作為設計決策紀錄;同一任務
再有新問題就往下附加新的 Q 區塊。

```markdown
# decision.md
STATUS: WAITING | ANSWERED

## Q1: <問題一句話>
- 選項與各自利弊
- planner 建議:<哪個選項、為什麼>

### 回答
(主 session 填入使用者的決定;planner 未拿到回答前不往下做)
```

`STATUS` 是機器可讀欄位:planner 寫入新問題時設 `WAITING`,主
session 填完所有未回答的問題後改成 `ANSWERED` 再續接 planner。

## review 檢查清單

planner 審查 implement.md 時逐項檢查(親自看 diff,不要只信
implement.md 的敘述),缺一項就是 CHANGES_REQUESTED:

- audit「變更摘要」是否真的符合「檔案異動清單」與實際 diff
- cowork.md「Commit gates」清單裡的每一個 gate 都有涵蓋且 pass
- 沒有修改既有測試 case(只能新增——既有 case 是 regression
  baseline;懷疑既有 case 有錯應該在 implement.md 回報,不是改掉)
- comment 品質:改到的程式碼周邊 comment 有跟著更新;comment 只
  描述最終程式碼的行為,沒有寫成修改過程
- 文件類產出(implement.md、README 等)白話易讀
- cowork.md「專案特有審查提醒」列的改動類型,不能只看測試綠燈,
  務必親自檢查邏輯正確性
- 下節「工程通則」逐條符合

## 工程通則

跨專案適用,planner 規劃時遵守、審查時逐條檢查,實作端動手前先讀。

**測試**
- 既有測試 case 只能新增不能改;要測 static 函式,把領域邏輯拆成
  正式匯出的窄函式,不要用巨集或 `#ifdef` 在產品程式碼開洞
- **測試要測到產品路徑本身**。為測試複製一份平行的邏輯,等於只驗
  了那份副本——把共用部分抽出來,兩邊呼叫同一個
- 樣本/fixture 必須實測抓得到它針對的缺陷:把被測邏輯暫時改壞、
  確認測試真的變紅。全綠卻抓不到目標缺陷的樣本等於沒寫
- 不要自加 gate 以外的驗證步驟,照專案既有 pipeline 走

**破壞性測試(sabotage / fault injection)**
- 注入只能改成 scratch 路徑或非破壞性的目標值;**嚴禁**把路徑變數
  指向 `/dev/*`、`/proc`、`/sys`、`/etc` 等系統路徑
- 凡是可注入的變數會流進 `rm`、`>`、`truncate`、`dd`、`mv` 等破壞性
  指令,那個指令**一律改用硬編常數** —— 同一個變數身兼「寫入目標」
  與「刪除目標」是災難的結構前提
- 產品/測試檔案裡不留注入鉤:注入用函式覆寫,cleanup 重新 source 還原

**Comment 與文件**
- comment 三秒原則:一句話、80 字元內;只寫程式碼看不出來的約束
  (鎖序、caller 契約、記憶體序),不寫比較句與指路句;放不進的細節移到
  文件。`@brief` 一句話講完做什麼,設計理由歸 plan/decision.md
- comment 只描述最終行為,不寫修改過程
- commit message、PR 標題內文一律英文;**comment 與 commit 都不得
  出現內部任務編號**(task-N、增量 X 等),送 review 前 grep diff 確認

**除錯與收尾**
- 停止程序一律 graceful(SIGTERM);SIGKILL 會跳過清理路徑
- 修復手段不得與事故共享失敗模式——例如「檔案不見了」的事故,
  修法就不能有「檔案暫時不存在」的中間狀態
- 遇紅先保全證據再動任何東西;重跑會覆蓋現場
- 弱化斷言(恆真化、`|| true`、catch-all、拿掉 control)不是實作端
  能自裁的修法,一律回 planner

**提選項時只提治本的**
- 列給使用者裁的選項,**每一個都必須是治本的做法**。治標的、遮蔽症狀
  的、把問題挪到別處的,不要列進去 —— 列出來就是在邀請對方選它
- 常見的治標形狀:縮小發生窗口而不關閉它、把測試移出會出問題的路徑、
  改設定讓症狀不出現、加重試掩蓋不穩定。這些即使「改動最小」也不算
  選項
- 若治本的做法只有一個,就只列一個。**選項數量不是目標**
- 若治本的代價使用者可能無法接受,照實說代價,由他決定要不要做 ——
  但不要因此端出一個治標的替代品

**完成的定義**
- **每一個 gate 都要綠,任務才算完成。** gate 紅著不能收尾,**即使歸因是
  既有缺陷、與本任務的改動無關** —— 「不是我弄壞的」不是放行的理由,
  紅的 gate 就是紅的
- 調查中挖到而併進本任務的問題,**要修完**才算完成;登記起來不等於處理過
- 帶著已知的紅去 commit,等於把「這裡壞掉」寫進歷史,下一個人會以為
  那是可接受的狀態

**範圍**
- 調查中挖到的相鄰問題,預設併進當前任務,不要每項開新編號
- 既有程式碼與本任務無關的部分不順手改,那只會撐大 diff

## implement.md 格式

實作端撰寫,**每次改動都必須整份重寫**(不是附加內容),格式:

```markdown
# implement.md
version: <整數,從 1 開始,每次因應 review 修改後 +1>
timestamp: <ISO 8601>

## 變更摘要
(修改了什麼、為什麼這樣改)

## 檔案異動清單
- path/to/file.ts: 新增 / 修改 / 刪除,一行說明

## 測試結果
(.claude/cowork/config.md「Commit gates」清單裡的每一個 gate 各一行:
 pass/fail、指令與輸出摘要,缺一個 gate 就不算完成)
```

`version` 欄位是主 session 判斷「有沒有新內容需要送審」的依據。
**版本號沒有增加,就不會重新審查**——這是避免 review 迴圈
卡死的關鍵規則,實作端每次因應意見修正完,一定要把這個數字加一並整份
重寫這個檔案。

### Gate 被 sandbox 擋住時

所有 gate 一律由實作端自己跑,主 session 不代跑。實作端跑在
sandbox 裡(如 codex 的 workspace-write)而 gate 需要 sandbox 外的
資源(裝置檔、特權網路、workspace 外路徑)時:

1. 實作端寫 `error.md` 註明是 sandbox 權限擋住哪個 gate(不算
   自己造成的錯誤,不用 stash 診斷)。
2. 主 session 對照 init doctor 的 sandbox 檢查項排除障礙——放寬
   sandbox(`CODEX_COMPANION_SANDBOX=danger-full-access` 開新
   thread、`network_access = true` 等)或請使用者手動調整。
3. 障礙排除後續接(sandbox 等級 resume 升不了級的,得開新 thread
   重新委派),讓實作端**自己重跑**全部 gate、自己寫 implement.md。

## review.md 格式

planner 撰寫。**檢查清單是必填表格,不是提醒**:每一條都要有「證據」
欄(看了哪個 diff 區段、哪個 log 的哪一行、跑了什麼指令),空著就
不是合法的 review;`gate 紅過的 step` 不是 `none` 時不得寫 `PASS`。

```markdown
# review.md
in_response_to_version: <對應的 implement.md version>
STATUS: PASS | CHANGES_REQUESTED

## 檢查清單(每條必填證據)
| 項目 | 結果 | 證據 |
|---|---|---|
| 變更摘要 = 檔案異動清單 = 實際 diff | ok / ng | <diff 區段 / 指令> |
| 每個 commit gate 都有跑且 pass | ok / ng | <各 gate 的結果行> |
| 既有測試 case 未修改 | ok / ng | <diff 中測試檔的變動性質> |
| comment 更新且只描述最終行為 | ok / ng | <看過的區段> |
| 文件白話 | ok / ng | |
| 專案特有審查提醒逐項 | ok / ng / n.a. | <哪幾項、怎麼查> |
| 工程通則逐項(含「完成的定義」「只提治本的」) | ok / ng | |
| gate 紅過的 step | none / <step 清單> | <哪個 gate 的哪個 log> |
| 紅過的 step 根因已查清且修復後再綠 | ok / n.a. | <根因一句話 + 重跑結果> |

## 意見
(STATUS 為 CHANGES_REQUESTED 時列出具體修改項目;PASS 時可留白)
```

`gate 紅過的 step` 的定義:**本任務任何一次 gate 執行**出現過的 FAIL,
不論後來重跑有沒有綠、不論 stash 基準是否也紅、不論是不是本次改動
造成。「既有 flake」「重跑就綠」「不是我弄壞的」都不是填 `none` 的理由;
有紅過就併進本任務查到根因、修到綠,才能 PASS(見工程通則「完成的定義」)。

`STATUS` 欄位是唯一的機器可讀判斷依據,雙方都用字串比對
`STATUS: PASS` 是否存在,不要用語意理解取代這個判斷。review.md 讀完
即可刪除,它不是需要保留版本歷史的檔案(implement.md 才是)。

## error.md 格式

實作端撰寫,僅在**確認不是自己造成**的錯誤時才寫,判斷方式:

```markdown
# error.md
version_at_error: <當時的 implement.md version>

## 錯誤描述

## 已排除的可能性
(說明為何判斷不是自己造成——附上 git stash 基準測試的結果)
```

**自我診斷規則**:實作端發現測試失敗時,先用 `git stash` 把自己的
改動擋掉、對乾淨基準跑一次同樣的測試。

- 基準(stash 後)也失敗 → 不是自己造成的,寫 `error.md`。
- 基準通過、只有改動後失敗 → 是自己造成的,`git stash pop` 還原後
  直接修正,不寫 `error.md`。
- sandbox 擋住 `.git` 寫入、stash 跑不了 → 在 error.md 註明
  「基準未驗證(sandbox 擋 stash)」,不要卡在原地重試;主 session
  會放寬 sandbox 後讓你自己重跑診斷。

`error.md` 由 planner 診斷,**必須先把處理方式寫進
`error_resolution.md` 或更新 `plan.md`**,主 session 才能刪除
`error.md`——刪除動作本身不承載任何資訊,純粹是「已讀,可以繼續」
的訊號。

## 角色職責對照

| 檔案 | 誰寫 | 誰刪 | 誰用來判斷下一步 |
|---|---|---|---|
| plan.md | planner | — | 實作端 |
| decision.md | planner(問題)/ 主 session(回答) | 保留不刪 | planner |
| implement.md | 實作端(每次重寫) | — | 主 session(比對 version)/ planner(審查) |
| review.md | planner | 實作端(讀完即刪) | 主 session(STATUS)/ 實作端 |
| error.md | 實作端 | 主 session(處理完才刪) | planner |
| error_resolution.md | planner | — | 實作端(下次實作時參考) |

## 診斷與量測(查 bug 時最常自己騙自己的地方)

**間接跡象不是證明。** 「規則消失了」「log 有 cleanup 字樣」這類
旁證,可能來自完全無關的正常流程。判準要是那件事本身——修法生效
會留下的、只有它會留下的那個訊號。硬判準的作用是「沒出現就一定
失敗」,不是「出現就一定成功」:單一判準達成不等於目標達成,其餘
的觀察值同樣要看。

**量測工具會製造假結論。** 實際踩過的:

- `pgrep -f <pattern>` 會命中自己的命令列;查程序用 `pgrep -x` 比對
  程序名,或掃 `/proc/*/comm`
- `ps -eo args | grep -c` 數的是 argv,而 forked 子 shell 沿用父程序
  的 argv——「還有 N 個程序」可能全是遺留子程序,主體早就死了
- bash 的函式層級重導在函式執行期間生效,`/proc/<pid>/fd` 讀到的
  是「當下停在哪個函式」的結果,不是腳本平時的樣子
- 取樣間隔會造出不存在的現象:間隔 15 秒只能得到「+15 還在、+30
  不在」,不能推出耗時 18 秒
- 為了量測而加的包裝層(`bash -c ... ; echo $?`)會改變 ppid、訊號
  處置等被測條件,讓被測物在量具下表現得不一樣
- 離線最小重現與真實環境的差異(job control 開關、fd 拓撲、前面
  幾十個步驟留下的繼承狀態)會讓結論反向

**沒有證據就說沒有。** 「機制成立 + 症狀吻合」是相容性論證,不是
證據。與其給一個聽起來合理的因果,不如說「這三種可能目前分不出
來,要分辨需要 X」——後者能讓下一步做對,前者會讓人往錯的方向
再走一輪。

**先取證再實作。** 依尚未證實的因果去改 code,改完發現沒用時,
你會同時不確定「假說錯了」還是「改法錯了」。
