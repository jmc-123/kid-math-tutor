# 模式 4：build（图谱扩展）

本模式只在本地仓库内读写文件，不进行任何网络操作（不联网访问 GitHub，不自动 clone/push）。

## 执行步骤

**Step 1 — 获取教材结构**

用户提供以下任一：

- 教材 PDF 路径（用 Python fitz 提取目录）
- 手动粘贴的章节列表
- 章节名称列表

如果是 PDF，执行：

```python
import fitz
doc = fitz.open("教材路径.pdf")
toc = doc.get_toc()
for level, title, page in toc:
    print("  " * (level-1) + f"第{page}页: {title}")
```

**Step 2 — 生成 topics.json**

字段定义参照 `../schema/topics.schema.json`（路径相对于本文件所在目录）。对每个知识点，生成完整记录：

```json
{
  "id": "mt_chXX_00N",          // 章号+序号，延续已有编号
  "type": "CONCEPTUAL|PROCEDURAL|REPRESENTATIONAL",
  "subject": "Mathematics",
  "domain": "所属大领域",
  "name": "知识点名称",
  "description": "2-3句话描述核心内容，含关键公式/定理/方法",
  "ageRangeStart": 12,
  "ageRangeEnd": 15,
  "centrality": 0.0-1.0,        // 在该领域的核心程度
  "evidence": ["能做X", "能做Y", "理解Z"],
  "assessmentPrompt": "含{{name}}占位符的诊断问题",
  "standards": ["pep-math-2022:X册.章.节"]
}
```

**Step 3 — 生成 dependencies.json**

字段定义参照 `../schema/dependencies.schema.json`。分析知识点间的前后依赖，生成：

```json
{
  "topicId": "mt_chXX_00N",
  "prerequisiteId": "mt_chYY_00M", // 可以是跨册的
  "strength": "hard|soft",
  "reason": "一句话说明为什么需要这个前置"
}
```

**Step 4 — 生成 clusters + curriculum-standards**

按领域聚合知识点，对齐课标，字段定义参照 `../schema/clusters.schema.json` 与 `../schema/curriculum-standards.schema.json`（参考已有文件的格式）。

**Step 5 — 生成 manifest.json**

统计 topics/dependencies/clusters/standards 数量，列出各章信息。

**Step 6 — 校验**

执行以下检查：

1. 所有 topicId 在 topics 中存在（跨册引用除外）
2. DAG 无环（Kahn 算法拓扑排序，visited == len(local_topics)）
3. id 无重复

校验通过后，将生成的文件写入 `../data/{grade}/` 目录即可（路径相对于本文件所在目录）。是否提交或推送到远程仓库由用户自行决定并手动操作，本 skill 不执行任何网络/Git 推送操作。
