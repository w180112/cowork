---
description: cowork 日常入口——把後面的話當任務丟進 task-loop;未初始化的專案會提示先跑 /cowork:init
---

cowork 任務入口。使用者附加的參數:$ARGUMENTS

分流規則:

1. `.claude/cowork/config.md` 不存在 → 不要代跑初始化,告訴使用者這個
   專案還沒設定,請先執行 `/cowork:init`——問答會讓他明確選擇
   planner / implementer 的後端與模型,到此結束。
2. 已初始化 → 讀取並依照
   `${CLAUDE_PLUGIN_ROOT}/skills/task-loop/SKILL.md` 執行任務迴圈,
   $ARGUMENTS 就是工作範圍或指示(如「開始任務 <描述>」「繼續」
   「處理下一個任務」);沒帶參數且 `.claude/cowork/root_plan.md`
   已存在 → 視同 continue,從第一個未勾選任務接續;沒帶參數也沒有
   root_plan.md → 問使用者這次要做什麼。
