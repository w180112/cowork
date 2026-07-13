<!-- cowork-kit-version: 1 — 本段由 /cowork:init 生成,升級協定請重跑 init -->
# 實作端職責(task-loop 協定)

本段給外部 CLI 實作端(Codex、OpenCode 等透過 AGENTS.md 載入指令的
agent)遵循;Claude Code 是規劃/審查端,不適用本段的執行步驟。

完整協定規格(檔案格式、STATUS 欄位、版本規則)見 ./WORKFLOW.md,
專案參數(commit gates、分支名等)見 ./.claude/cowork.md。開始任何
實作前請先讀取這兩份檔案。

## 執行步驟

1. 讀取 `artifacts/task-<id>/plan.md`,理解目標與驗收條件。

2. 依 plan.md 實作,依循本檔案的程式碼慣例,完成後**依
   `.claude/cowork.md`「Commit gates」清單的順序跑完全部 gate**,
   並遵守「Gate 注意事項」。

3. 測試失敗時,先執行以下自我診斷再決定下一步:

   ```
   git stash
   <重跑失敗的那個 gate>
   git stash pop
   ```

   - 基準(stash 後)也失敗 → 不是你造成的,寫
     `artifacts/task-<id>/error.md`(格式見 WORKFLOW.md),結束這一輪。
   - 基準通過、只有你的改動後失敗 → 是你造成的,直接修正後重跑,
     不要寫 error.md。

4. 測試全部通過後,寫入 `artifacts/task-<id>/implement.md`(格式見
   WORKFLOW.md,注意 version 欄位)。「測試結果」必須涵蓋 gates
   清單裡的每一項。若 `.claude/cowork.md`「專案特有審查提醒」有列
   需要特別標註的改動類型,在「變更摘要」開頭明確標註。

5. 若之後收到針對這次 implement.md 的修正指令(會附上 review.md 的
   內容):
   - 依照 review.md 的意見修改
   - 重新跑過一次完整的 gates
   - 整份重寫 implement.md,version 欄位 +1——不要只回覆修改了什麼,
     一定要重寫這個檔案,這是 Claude Code 判斷「可以重新審查了」的
     唯一依據

6. 全程不要自己判斷「這樣應該可以了」就直接 commit / push——commit
   與 PR 一律由 Claude Code 端(透過 committer subagent)處理,你只
   負責 `artifacts/task-<id>/` 底下的 `implement.md` 與 `error.md`。
