# cowork task-loop 專案設定

<!-- cowork-kit-version: 2 -->

task-loop skill 每輪開始時讀取本檔案。修改設定直接改這裡即可,
不需要重跑 /cowork:init(重跑 init 用於升級協定版本或重新問答)。
換 implementer 後端 / 模型可用 `/cowork:model`。

- main_branch: {{MAIN_BRANCH}}
- implementer: {{IMPLEMENTER}}         <!-- claude | codex | opencode;隨時可改,下次委派生效。
                                            從 claude 改成 codex/opencode 時請重跑
                                            /cowork:init 讓 AGENTS.md 補上協定段落 -->
- implementer_model: {{IMPLEMENTER_MODEL}}   <!-- claude: opus | sonnet
                                                  codex: gpt-5.6 | gpt-5.5(依 codex 設定支援為準)
                                                  opencode: 完整 provider/model,如 zai/glm-5.2 -->
- approval: {{APPROVAL}}         <!-- ask:branch/commit/PR 都先問使用者 | auto:全自動 -->
- commit_language: {{COMMIT_LANGUAGE}}
- chat_language: {{CHAT_LANGUAGE}}

## Commit gates(依序執行,全部通過才能 PASS)

{{GATES}}

## Gate 注意事項

{{GATE_NOTES}}

## 專案特有審查提醒

{{REVIEW_NOTES}}
