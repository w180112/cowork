# Task-loop 協定規格

<!-- cowork-kit-version: 2(由 /cowork:init 生成,升級協定請重跑 init)-->

本文件是 Claude Code(規劃 / 審查 / 整合)與實作端(Codex 或 Claude
subagent,見 `.claude/cowork.md` 的 implementer 設定)之間交接的唯一
事實來源。實際的檔案格式、狀態欄位、版本規則以本文件為準。

## 目錄結構

每個任務對應一個獨立資料夾,避免不同任務互相覆蓋、也留得住歷史紀錄:

```
artifacts/
└── task-<id>/
    ├── plan.md               # Claude Code 產生,實作端的實作規格
    ├── implement.md          # 實作端產生,每次修改後版本號 +1
    ├── review.md             # Claude Code 產生,一次性訊息,讀完即刪
    ├── error.md               # 實作端產生,Claude Code 處理完才刪除
    └── error_resolution.md    # Claude Code 產生,error.md 的處理紀錄
```

`task-<id>` 使用 root_plan.md 任務清單裡的序號,例如 `task-3`。整個
`artifacts/` 目錄不進版控(/cowork:init 已加入 .gitignore)。

## plan.md 格式

Claude Code 撰寫,內容至少包含:

- 目標與背景(為什麼要做這個任務)
- 涉及的檔案路徑清單
- 明確的驗收條件——具體到測試名稱或可驗證的行為描述,不要寫
  「功能正常」這種無法驗證的敘述
- 已知的邊界情況或限制

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
(.claude/cowork.md「Commit gates」清單裡的每一個 gate 各一行:
 pass/fail、指令與輸出摘要,缺一個 gate 就不算完成)
```

`version` 欄位是 Claude Code 判斷「有沒有新內容需要重新審查」的依據。
**版本號沒有增加,Claude Code 就不會重新審查**——這是避免 review 迴圈
卡死的關鍵規則,實作端每次因應意見修正完,一定要把這個數字加一並整份
重寫這個檔案。

## review.md 格式

Claude Code 撰寫,格式:

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

Claude Code 處理完 `error.md` 後,**必須先把處理方式寫進
`error_resolution.md` 或更新 `plan.md`**,才能刪除 `error.md`——刪除
動作本身不承載任何資訊,純粹是「已讀,可以繼續」的訊號。

## 角色職責對照

| 檔案 | 誰寫 | 誰刪 | 誰用來判斷下一步 |
|---|---|---|---|
| plan.md | Claude Code | — | 實作端 |
| implement.md | 實作端(每次重寫) | — | Claude Code(比對 version) |
| review.md | Claude Code | 實作端(讀完即刪) | 實作端 |
| error.md | 實作端 | Claude Code(處理完才刪) | Claude Code |
| error_resolution.md | Claude Code | — | 實作端(下次實作時參考) |
