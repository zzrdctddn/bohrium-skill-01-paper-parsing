# 文献文档解析 Skill（layout-aware-doc-parser）

一个把科研 PDF 转成"可溯源结构化解析结果"的 Agent Skill。核心：先定位版面 → 重建阅读顺序 → 还原章节/表格/公式结构 → 逐块溯源核验，提得全、排得对、拆得清、核得回。

## 目录结构
- `SKILL.md` —— Skill 本体（YAML frontmatter、LASP 方法、SOP、输出契约）
- `references/` —— 方法细则与反例库、自检清单、字段字典、版面规则
- `assets/` —— 可编辑结构化解析模板
- `examples/` —— 最小调用示例（已标注为示意，非真实案例）
- `GITHUB_URL.txt` —— 提交前填 GitHub 公开链接（当前为 REPLACE_ME 占位）

## 使用方式
1. 将本目录作为 Skill 安装到 Agent（平台约定路径 `/skills/weekly-skill`）。
2. 输入 `/input/case.json`（含 `case_id`、任务、PDF 路径、聚焦方向）与 `task.md`。
3. 自然语言下指令，如"解析这篇双栏论文，别串栏，表格要能复用，每段标页码"。
4. 按 SKILL.md 的 SOP 执行，输出到 `/app/output/result.json` 与 `parsed_document.md`。

## 依赖
- 平台预装的 PDF 文本/版面提取工具与 OCR 能力（若可用）；无文字层时启用 OCR 并做后处理校验。
- 解析规则、锚点格式、字段字典均内置于 SKILL.md 与 `references/`，无需外部词表。
- 注：Runtime contract 规定预算为 64 次模型请求 / 900 秒 / 每次 16384 token；资源≤50 个 UTF-8 文本文件、单文件≤500KB、总计≤2MB、无软链。真实 PDF 以 `/input` 实际文件为准，不依据文件名或领域标签臆测内容。

## 适用与限制
- 适用：双栏/多栏科研论文、含复杂表格与公式的 PDF、扫描件 OCR 后处理，以及一切要求"每个结果指回页码区块"的可溯源解析。
- 限制：加密且无权限的 PDF 标 `blocked`；图片型表格/公式、乱码符号、低清扫描件标 `uncertain` 并进人工确认；图表与公式按结构元素识别定位，不解读科学含义。材料缺失、无法判断、运行失败分别说明，不编造事实与 benchmark 分数。

## 评分对应
实际效果(40)、使用增益(25)、创新增益(15)、可靠体验(10)、生态贡献(10) 的支撑材料分别见 SKILL.md 第 3/2/2/6/8 节与 `references/`：
- 实际效果：SOP Step 1~4（提得全/排得对/拆得清/核得回）+ `block_provenance`/`source_anchor_map`。
- 使用增益：LASP 四机制带来的额外结构（带坐标区块表、阅读顺序映射、表格结构与公式登记）+ 增益可对照的 `self_check`。
- 创新增益：多栏顺序重建、表格结构重建、逐块溯源为基线难稳定实现的能力，见 `references/method.md`。
- 可靠体验：Step 0 边界确认 + 第 6 节异常处理表。
- 生态贡献：字段字典、锚点规则、可复用模板与示例见 `references/`、`assets/`、`examples/`。
