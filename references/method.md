# LASP 方法细则与反例库

## 四机制对应关系
LASP 的四机制不是并列技巧，而是有依赖关系的流水线；前一机制的产出是后一机制的输入：

- **提得全（Layout Block Inventory）→ 排得对（Reading Order Reconstruction）**：只有先把页面切成带坐标的区块，才谈得上按几何重排顺序。
- **排得对 → 拆得清（Structure Reconstruction）**：章节、表格、公式只有在正确阅读顺序下还原，结构才不会错位。
- **拆得清 → 核得回（Per-Block Provenance）**：结构还原后，每个结构单元才能回链到其来源区块与坐标。

跳过前置机制会级联失败：区块没切好 → 顺序必错 → 表格串列 → 溯源指错页。

## 机制一：提得全 —— 版面区块清单
- 区块类型枚举见 `layout_rules.md`：title / author / affiliation / abstract / body / section_heading / figure_caption / table_caption / footnote / reference / header / footer / equation。
- 逐字提取，保留原始标点与断行；页眉页脚、脚注、参考文献必须独立类型，**不得并入 body**。
- 每个区块必带 `block_id`、`page`、`bbox=[x0,y0,x1,y1]`、`confidence`、`status`。
- 硬规则：宁可标 `unreadable` 也不填猜测文字；漏一个区块比多标一个 uncertain 更严重。

## 机制二：排得对 —— 阅读顺序重建
判定优先级（由高到低）：
1. **通栏优先**：跨整页宽的标题、通栏表格/公式，插入其 y 位置对应的顺序点。
2. **栏内自上而下**：同一栏内按 y 坐标升序。
3. **多栏先左后右**：双栏先扫左栏到底，再扫右栏；三栏类似。
4. **跨页续接**：段落在页末未完，下一页首块标 `continuation_of=上一块 block_id`，**不拼成一句**（保留段落边界与各自页码）。

反直觉点：页末的左栏底部文字，其顺序在右栏顶部文字**之前**；按纯 y 坐标排序会把两栏交错，是典型串栏错误。

## 机制三：拆得清 —— 结构还原
- **章节树**：用编号（如 2.1）与标题样式推断 level；每节点存 `parent`、`page`；编号跳跃要标 `numbering_uncertain`。
- **表格**：先定网格（行列数），再逐格归属；合并单元格用 `rowspan/colspan` 表达，不改写成重复文本；多层表头用 `header_rows` 数组；跨页续表用 `continuation` 链接且**各自保留页码**。
- **公式**：按结构元素登记 `eq_id`、编号、原文对照；上下标、希腊字母、特殊符号逐字保真；无法确证的符号标 `symbol_uncertain`，不解读公式含义。

## 机制四：核得回 —— 逐块溯源
- `source_anchor` 五要素：`block_id` + `page` + `bbox` + `basis`（提取依据）+ `confidence`。
- `source_anchor_map` 把任务要求的关键结论逐条回链；"关键结论"由 `task.md` 的证据范围决定。
- 读不出 → `status=unreadable`；与原文冲突或无法判断 → `confidence=low` 并进 `human_review_items`；两者都不得生成替代文字。

## 提取与核对分离
解析阶段禁止自我核对；解析完成后以"审阅者"身份独立执行 checklist，发现错误回退对应步骤并写 `change_log`。两遍合并会漏掉自己引入的串栏与错格。

## 反例库
| 反例 | 为什么错 | 改法 |
|---|---|---|
| 先左右栏按 y 混合排序输出 | 双栏交错串栏，左栏末行接到右栏首行 | 先按栏分组、栏内按 y、左栏优先 |
| 页眉页码写进正文首行 | 污染正文，逐字比对失败 | 页眉页脚归独立类型，不进 body |
| 合并单元格拆成多格重复文本 | 破坏表格结构，无法复用比对 | 用 rowspan/colspan 表达 |
| 跨页段落拼接成一句 | 丢失段落边界与出处页码 | 标 continuation_of，各段保留页码 |
| 希腊字母/上下标拍平成普通字符 | 公式失真，方法描述不可读 | 逐字保真，存疑标 symbol_uncertain |
| 扫描件读不出就编一段"完整正文" | 编造事实，违反红线 | 标 unreadable + uncertainties，进人工确认 |
| 溯源只写"第 3 页" | 粒度太粗，无法复核具体区块 | 写 page + bbox + basis + block_id |
| 无可核验记录却自称"质量提升" | 自述不能替代证据 | 用 self_check 与对照记录证明，不编 benchmark |
