---
name: layout-aware-doc-parser
description: 把科研 PDF（含双栏论文、复杂表格、公式、扫描件）解析成可继续处理、逐块可溯源的结构化结果，并按原版面顺序还原阅读次序。当用户要求"解析这篇 PDF""提取正文/图表题注/脚注/参考文献""读双栏论文别串栏""还原表格行列与合并单元格""保留公式上下标与希腊字母""扫描件文字识别与校验""每个结果要标页码出处"时使用。输出含版面区块表、阅读顺序映射、章节树、表格结构、公式登记、证据锚点与自检结果。
version: 1.0.0
license: MIT
---

# 文献文档解析（Layout-Aware Structured Parsing）

> 面对真实科研 PDF，把"解析文档"从"吐出一段看起来像全文的文本"变成**可复核的版面工程**。核心不是提取更多字，而是让每一个字符、每一个单元格、每一个符号都能回答："它来自哪一页、哪个区块、按什么顺序？读不出来的是什么？"

## 1. 用途与适用边界

**解决什么**：基础 Agent 解析 PDF 的典型失败——双栏串栏（左栏末行接到右栏首行）、跨页错序、表格丢列或把合并单元格拆散、公式上下标拍平、希腊字母变问号、页眉页脚混进正文、扫描件直接编文字、读不出也硬凑成"完整全文"。本 Skill 用"**先定位版面 → 再重建顺序 → 再还原结构 → 最后逐块溯源核验**"四段流水线消除这些失败，读不出的如实标注，绝不靠猜补齐。

**触发场景**：解析/重解析科研 PDF；按版面顺序提取正文与题注；双栏多栏论文的阅读顺序修正；跨行跨列/多层表头/跨页续表的结构还原；公式编号与符号保真；章节层级识别；扫描版 PDF 的 OCR 与后处理校验；要求"每段每表指回页码区块"的可溯源任务。

**不该用**：只要"总结这篇论文讲了什么"而不要逐字结构与出处时（转用文献摘要/笔记 Skill）；只要语言翻译时（转用文献翻译 Skill）；PDF 加密且无解密权限时（应报告不可解析，不得绕过权限）。图表与公式按**结构元素**识别与定位，不要求理解其科学含义——本 Skill 不做科学语义解读。

**输入**：`/input/case.json`（含 `case_id`、任务说明、PDF 路径或材料清单、聚焦方向、证据范围）与 `task.md`。真实 PDF 以实际文件为准，不依据文件名或领域标签臆测内容。**输出**：`/app/output/result.json`（字段见第 4 节）+ 可编辑的 `parsed_document.md`。

## 2. 核心方法（LASP）

本 Skill 的方法名为 **LASP = Layout-Aware Structured Parsing（版面感知的可溯源解析）**，四个不可跳过的机制，一一对应题面"提得全 / 排得对 / 拆得清 / 核得回"：

1. **提得全 —— 版面区块清单（Layout Block Inventory）**：先把页面切成带坐标的区块（`block_id` + 页码 + bbox + 类型），类型覆盖标题/作者/单位/摘要/正文/章节标题/图表题注/脚注/参考文献/页眉页脚。逐字提取每个区块，页眉页脚单独归类**不进正文**。任何区块缺字、乱码、识别失败必须登记，不得静默省略。
2. **排得对 —— 阅读顺序重建（Reading Order Reconstruction）**：按版面几何判定栏序（双栏先左后右、跨栏通栏优先、跨页续接），为每个区块分配 `reading_order_index`。规则：同一页先按栏、再按 y 坐标；跨栏元素（通栏标题、跨栏表格、双栏公式）在其跨越位置插入；跨页段落用 `continuation_of` 链接，不拼成一句。串栏与错行是硬错误。
3. **拆得清 —— 结构还原（Structure Reconstruction）**：章节还原多级层级与编号（parent 指针）；表格还原行、列、单元格归属与 `rowspan/colspan`、多层表头、跨页续表；公式还原上下标、希腊字母、编号与原文对照（按结构元素登记，不解读含义）。三者分别输出 `section_tree`、`table_structures`、`equation_registry`。
4. **核得回 —— 逐块溯源（Per-Block Provenance）**：每个输出块携带 `source_anchor`（页码 + 区块坐标 + 提取依据 + 置信度）；`source_anchor_map` 把关键结论映射到原文位置。OCR 不可辨、冲突或缺失的内容标 `uncertain`/`unreadable` 并进 `human_review_items`，**不生成"看似合理"的替代文字**。

贯穿一条铁律：**提取与核对分离**——先完成解析草稿，再以"审阅者"身份独立执行第 5 节自检，两遍不可合并；每次修正写入 `change_log`。

## 3. 工作流（SOP）

严格按顺序执行，每步都有检查点；检查点不过则回退该步，不得带病推进。

**Step 0 · 输入解析与边界确认**
- 读 `case.json` 与 `task.md`，提取：PDF 路径、聚焦方向（多栏/表格/公式/扫描件等）、需要的结构字段、证据范围、明确禁止事项。
- 打开真实 PDF，确认是否加密、是否有文字层、页数、语言、栏数初判。
- 检查点：能否读取真实文件？加密/无权限则 `status=blocked` 说明，不得伪造内容；缺口登记到 `uncertainties`。

**Step 1 · 页面切分与版面区块清单（提得全）**
- 逐页切分版面，产出带 bbox 的区块清单，标注类型与逐字文本；页眉页脚、脚注、参考文献分区存放。
- 检查点：正文/题注/脚注/参考文献是否分别归位？页眉页脚是否未混入正文？有无漏行丢段？缺失区块登记 `uncertainties`。

**Step 2 · 阅读顺序重建（排得对）**
- 依据栏几何与跨栏元素判定顺序，为每块写 `reading_order_index`；跨页段落写 `continuation_of`。
- 检查点：左栏末行是否未接右栏首行？跨页是否未拼句？通栏标题/表格是否插在正确位置？错位即回退本步。

**Step 3 · 结构还原（拆得清）**
- 3.1 章节树：标题按编号与层级建 `section_tree`（level/parent/page）。
- 3.2 表格：逐表还原行列、单元格、`rowspan/colspan`、多层表头、跨页续表（`continuation`）。
- 3.3 公式：逐式登记编号、上下标、希腊字母、原文对照；符号存疑标 `symbol_uncertain`。
- 检查点：表格单元格归属是否逐格核对？公式符号是否与原文逐字一致？有无把页眉数字当表头？

**Step 4 · 逐块溯源映射（核得回）**
- 为每块生成 `source_anchor`，汇总 `source_anchor_map`；把任务要求的关键结论逐条回链页码区块与提取依据。
- 检查点：每条关键结论是否有可定位锚点？读不出/冲突内容是否已标 `uncertain` 并进 `human_review_items`？

**Step 5 · 独立核对（与解析分离）**
- 以审阅者身份逐项走 `references/checklist.md`；重点复核四类关键问题是否全部达标；发现错误回退对应步骤并在 `change_log` 记录。
- 检查点：自检表全部通过，或未通过项均有 `change_log` 与处置说明。

**Step 6 · 输出**
- 写 `parsed_document.md`（可编辑结构化解析）+ `result.json`（结构化结果，含 `case_id` 与全部输出契约字段）。

## 4. 输出契约（result.json）

```json
{
  "case_id": "<原样带回>",
  "status": "success | partial | blocked",
  "artifact": {
    "title": "<标题逐字文本>",
    "authors": ["<作者>"],
    "affiliations": ["<单位>"],
    "abstract": "<摘要逐字文本>",
    "sections": [{"section_id": "S2", "level": 2, "number": "2", "title": "<章节标题>", "page": 3, "text": "<正文>"}],
    "figures": [{"fig_id": "Fig1", "caption": "<题注逐字>", "page": 4}],
    "tables": [{"table_id": "T1", "caption": "<题注逐字>", "page": 5}],
    "reading_order": ["B001", "B002", "..."]
  },
  "block_provenance": [
    {"block_id": "B001", "type": "body", "text": "...", "page": 1, "bbox": [x0,y0,x1,y1], "reading_order_index": 3, "confidence": "high|medium|low", "status": "ok|uncertain|unreadable"}
  ],
  "reading_order_map": [{"page": 1, "column": "left", "sequence": 1, "block_id": "B001", "note": "左栏首块"}],
  "section_tree": [{"section_id": "S2", "level": 2, "title": "...", "number": "2", "parent": "S1", "page": 3}],
  "table_structures": [
    {"table_id": "T1", "page": 5, "rows": 4, "cols": 6, "header_rows": [0],
     "cells": [{"row": 0, "col": 0, "rowspan": 1, "colspan": 1, "text": "..."}],
     "continuation": {"continued_from": null, "continued_on_page": null}}
  ],
  "equation_registry": [{"eq_id": "E3", "page": 6, "number": "(3)", "text": "<原文对照>", "symbols": ["α","β"], "symbol_uncertain": false}],
  "source_anchor_map": [{"claim": "<关键结论/字段>", "block_id": "B001", "page": 1, "bbox": [x0,y0,x1,y1], "basis": "<提取依据>"}],
  "uncertainties": ["..."],
  "human_review_items": ["..."],
  "change_log": [{"step": 2, "action": "修正左栏末块顺序", "reason": "原顺序串到右栏"}],
  "self_check": {"all_extracted": true, "order_correct": true, "structures_restored": true, "provenance_complete": true}
}
```

字段原则：`status` 诚实反映完成情况；`partial` 必须写清 `uncertainties` 与缺口；`blocked` 说明拦截原因（如加密无权限）；`unreadable` 块不得填猜测文字；不得编造 benchmark 分数、页数或软件状态。证据映射由 `source_anchor_map` 承担，逐条可定位到页码与 bbox。

## 5. 质量自检清单

执行前完整读取 `references/checklist.md`，逐项打钩。关键硬项：

- [ ] 正文、图表题注、脚注、参考文献逐字提取，无漏行丢段
- [ ] 页眉页脚单独归类，未混入正文
- [ ] 阅读顺序按原版面，双栏/多栏不串栏、跨页不错行不拼句
- [ ] 章节层级与编号、parent 指针正确
- [ ] 表格行列、单元格归属、合并单元格、多层表头、跨页续表准确
- [ ] 公式上下标、希腊字母、编号与原文逐字一致
- [ ] 每块有 `source_anchor`（页码+bbox+依据+置信度）
- [ ] 读不出/冲突内容标 `uncertain`/`unreadable` 并进 `human_review_items`
- [ ] 无编造文字、数值、benchmark 分数或软件状态
- [ ] 独立核对完成，改动均在 `change_log`

## 6. 异常处理与边界

| 情况 | 处理 |
|---|---|
| PDF 加密/无权限 | `status=blocked`，说明权限拦截，不绕过、不伪造内容 |
| 无文字层扫描件 | 走 OCR 流程并做后处理校验；低置信度块标 `uncertain`，进人工确认，不硬识别 |
| 图片型表格/公式 | 按结构元素登记并标 `image_based`，无法确证的内容不填猜测值 |
| 乱码字体/符号失真 | 保留原文形状描述，标 `symbol_uncertain`，进 `human_review_items` |
| 多栏错序/跨页续接 | 回退 Step 2 依几何重排；仍不确定则标 `order_uncertain` 并说明 |
| 页码错乱/缺页 | 按实际页面记录，缺页登记 `uncertainties`，不补造页面内容 |
| 材料中无某结构（如无表格） | 该字段留空数组并在 `uncertainties` 说明"原文未含"，不虚构内容 |
| 与领域标签不符 | 以实际 PDF 内容为唯一事实依据，不据标签补造数据 |

## 7. 最小调用示例（示意，非真实案例）

**输入**（`case.json` 片段）：
```json
{"case_id":"demo-001","task":"按原版面顺序解析这份双栏 PDF，还原表格结构与公式，每段标页码出处","focus":["multi_column_order","table_structure","provenance"],"pdf":"/input/materials/paper.pdf"}
```
**调用**：加载本 Skill 后，用户说"用 layout-aware-doc-parser 解析这篇论文，别串栏、表格要能复用、每段标页码"。

**示意输出**（节选）：
```json
{"case_id":"demo-001","status":"success","artifact":{"reading_order":["B001","B002"],"tables":[{"table_id":"T1","page":3}]},"block_provenance":[{"block_id":"B001","type":"body","page":1,"bbox":[60,80,300,200],"reading_order_index":1,"status":"ok"}],"table_structures":[{"table_id":"T1","page":3,"rows":3,"cols":4,"cells":[]}],"source_anchor_map":[{"claim":"表1列出3组参数","block_id":"B014","page":3,"bbox":[70,400,540,560],"basis":"题注+表格几何"}]}
```
> 上述为演示接口用的虚构示例（bbox 与文本均为占位示意），不代表任何真实案例结果。

## 8. 资源清单

- `references/method.md` —— LASP 四机制细则与反例库
- `references/checklist.md` —— 覆盖四类关键问题的逐项自检清单
- `references/schema.json` —— `result.json` 字段字典（JSON Schema draft-07）
- `references/layout_rules.md` —— 版面区块类型、阅读顺序判定与锚点格式规则
- `assets/parsed_document_template.md` —— 可编辑结构化解析模板
- `examples/` —— 最小调用示例（已标注为示意，非真实案例）
- `README.md` —— 目录结构、使用方式、依赖与评分对应说明
- `GITHUB_URL.txt` —— 提交前替换占位链接
