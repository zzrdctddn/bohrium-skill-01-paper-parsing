# 示例说明（示意，非真实案例）

本目录提供一组最小调用示例，仅用于说明输入输出接口与完整使用流程，不构成任何真实案例结果或正式答案。

- `example_input.json`：虚构输入，含 `case_id` 与 `note`（注明"演示接口用，非官方案例"）。
- `example_output.json`：虚构输出，含 `case_id`、`status`、主产物 `artifact`、逐块溯源 `block_provenance`、证据映射 `source_anchor_map`、`uncertainties`、`human_review_items`、`change_log`、`self_check`。其中 bbox、页码、文本与结论均为占位示意。

## 调用流程
1. 加载 `layout-aware-doc-parser` Skill。
2. 按 `example_input.json` 的形态准备 `/input/case.json` 与 `task.md`（真实运行时 PDF 为实际文件）。
3. 自然语言下指令，如"解析这篇双栏论文，别串栏、表格要能复用、每段标页码"。
4. 按 SKILL.md 的 SOP 执行，产出 `result.json` 与 `parsed_document.md`。

## 复用提示
- 示例中的 `focus`、`need_fields`、`evidence_scope` 可替换为你自己的任务需求；字段字典见 `references/layout_rules.md`。
- 不要把示例中的虚构数值、bbox 或结论当作真实解析结果；正式结果必须来自对真实 PDF 的实际读取与核对。
