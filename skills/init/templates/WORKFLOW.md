# Task-loop 協定規格

<!-- cowork-kit-version: 3(由 /cowork:init 生成,升級協定請重跑 init)-->

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

## review.md 格式

planner 撰寫,格式:

```markdown
# review.md
in_response_to_version: <對應的 implement.md version>
STATUS: PASS | CHANGES_REQUESTED

## 意見
(STATUS 為 CHANGES_REQUESTED 時列出具體修改項目;PASS 時可留白)
```

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
