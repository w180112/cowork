# cowork task-loop 專案設定

<!-- cowork-kit-version: 4 -->

task-loop skill 每輪開始時讀取本檔案。修改設定直接改這裡即可,
不需要重跑 /cowork:init(重跑 init 用於升級協定版本或重新問答)。
換 planner / implementer 的後端或模型可用 `/cowork:model`。

- main_branch: {{MAIN_BRANCH}}
- planner: {{PLANNER}}                 <!-- claude | codex | opencode;規劃/審查/錯誤診斷角色。
                                            隨時可改,下次委派生效;planner 與 implementer
                                            不可同時為 codex(--resume 單 thread 會搶線)。
                                            從 claude 改成 codex/opencode 時請重跑
                                            /cowork:init 讓 AGENTS.md 補上協定段落 -->
- planner_model: {{PLANNER_MODEL}}     <!-- claude: fable | opus | sonnet
                                            codex: gpt-5.6 | gpt-5.5(依 codex 設定支援為準)
                                            opencode: 完整 provider/model,如 zai/glm-5.2 -->
- implementer: {{IMPLEMENTER}}         <!-- claude | codex | opencode;實作角色,規則同上 -->
- implementer_model: {{IMPLEMENTER_MODEL}}   <!-- claude: opus | sonnet
                                                  codex / opencode:同 planner_model 格式 -->
- approval: {{APPROVAL}}         <!-- ask:branch/commit/PR 都先問使用者 | auto:全自動 -->
- commit_language: {{COMMIT_LANGUAGE}}
- chat_language: {{CHAT_LANGUAGE}}

## Commit gates(依序執行,全部通過才能 PASS)

<!-- 所有 gate 一律由實作端自己跑。gate 需要 sandbox 外資源
     (裝置檔、特權網路等)時,解法是放寬實作端的 sandbox,
     不是主 session 代跑(見 WORKFLOW.md「Gate 被 sandbox
     擋住時」)。 -->

{{GATES}}

## Gate 注意事項

{{GATE_NOTES}}

## 專案特有審查提醒

{{REVIEW_NOTES}}
