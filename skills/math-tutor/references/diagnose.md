# 模式 1：diagnose（错题诊断）

## 执行步骤

**Step 1 — 读取数据**

根据错题判断涉及的年级，读取 `../data/{grade}/topics.json` 和 `../data/{grade}/dependencies.json`（路径相对于本文件 references/diagnose.md 所在目录，即 math-tutor 自带的 data/）。

**Step 2 — 题目→知识点映射**

分析每道错题，在 topics.json 中找到最匹配的知识点（一道题可能命中多个）。

匹配依据：

- 题目类型（方程/函数/几何/统计/概率）→ 缩小到对应 domain
- 题目操作（计算/证明/作图/判断）→ 匹配 type（PROCEDURAL/CONCEPTUAL/REPRESENTATIONAL）
- 关键词（配方/因式分解/圆周角/相似比...）→ 精确匹配 name/description

**Step 3 — 依赖链回溯**

从命中的知识点出发，沿 dependencies.json 中 `prerequisiteId` 方向向上追溯：

1. 找出所有 hard 依赖的前置知识点
2. 对每个前置知识点，用 evidence 字段判断学生是否可能已掌握
3. 找到"最底层未掌握的前置"作为根因

判断"可能未掌握"的信号：

- 该前置是同次考试中另一道错题
- 错题的错误方式暗示（如分解因式出错 → 整式乘法可能未掌握）
- 用户主动说不会某概念

**Step 4 — 输出诊断报告**

```
📊 诊断报告

━━ 错题分析 ━━
题目: [题目内容或描述]
涉及知识点: [domain > name]（id: mt_chXX_XXX）
错误类型: [概念混淆 / 计算失误 / 方法不熟 / 审题偏差]
错因分析: [具体哪一步出错，为什么出错]

━━ 知识薄弱链 ━━
🔴 根因: [最底层未掌握的知识点 name]
  ↓ 影响
🟡 [中间知识点 name]
  ↓ 影响
🟠 [直接出错的知识点 name]

━━ 诊断结论 ━━
核心薄弱点: [1-3个，按优先级排列]
建议补强顺序: [从底层到上层]
预计补强时间: [每个知识点约 1-2 小时，总计 X 小时]

下一步: 输入 /math-tutor learn 开始补强，或 /math-tutor generate 直接练习
```
