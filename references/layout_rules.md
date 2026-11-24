# 版面规则：区块类型、阅读顺序与锚点格式

本文件是 LASP 机制一、二、四的字段级细则，供 Agent 执行时查表。

## 1. 版面区块类型（type 枚举与归属规则）
| 类型 | 含义 | 是否进正文 | 提取要点 |
|---|---|---|---|
| title | 论文标题 | 否（单列） | 逐字，保留大小写与标点 |
| author | 作者 | 否（单列） | 逐字，保留上标编号含义 |
| affiliation | 单位 | 否（单列） | 逐字 |
| abstract | 摘要 | 否（单列） | 逐字，整段不拆 |
| body | 正文段落 | 是 | 逐字，保留段落边界 |
| section_heading | 章节标题 | 否（入 section_tree） | 保留编号 |
| figure_caption | 图题注 | 否（入 figures） | 与图号一一对应 |
| table_caption | 表题注 | 否（入 tables） | 与表号一一对应 |
| footnote | 脚注 | 否（单列） | 逐字，标所在页 |
| reference | 参考文献条目 | 否（单列） | 逐条，保留编号 |
| header / footer | 页眉/页脚 | 否 | 单列，**禁止并入 body** |
| equation | 公式 | 否（入 equation_registry） | 按结构元素，保真符号 |

## 2. bbox 约定
- 格式 `[x0, y0, x1, y1]`，原点为页面左上角，单位为点（pt）或平台提取工具给出的原始单位；同一文档内单位必须一致。
- 区块 bbox 不得跨页；跨页内容拆成多块，用 `continuation_of` 链接。

## 3. 阅读顺序判定（reading_order_index / reading_order_map）
1. 通栏元素（宽度跨过页宽约 90% 以上）优先，按 y 插入顺序点。
2. 其余按栏分组；双栏 `column=left` 全扫完再 `column=right`；栏内 y 升序。
3. `reading_order_index` 全文档连续递增，从 1 起。
4. 跨页续接：后一块 `continuation_of=前一块 block_id`，两者各自保留页码与 bbox，不合并文本。
5. 判定存疑写 `note="order_uncertain: <原因>"`，并进 `human_review_items`。

## 4. 表格结构字段字典（table_structures）
- `table_id`：与题注表号一致，如 T1。
- `rows` / `cols`：网格行列数（含表头行）。
- `header_rows`：多层表头的行索引数组，如 `[0,1]`。
- `cells[]`：`{row, col, rowspan, colspan, text}`；行列索引从 0 起；合并单元格只在主格出现一次，`rowspan/colspan>1`，不重复填充被合并位置。
- `continuation`：`{continued_from, continued_on_page}`；跨页续表各页各建一条 `table_structures`，由该字段互链。

## 5. 公式字段字典（equation_registry）
- `eq_id` / `number`：如 E3 / `(3)`，与原文编号一致。
- `text`：原文对照（保真上下标、希腊字母、特殊符号；无法用纯文本表达时保留可读近似并标 `symbol_uncertain=true`）。
- `symbols[]`：出现的特殊符号清单。
- 不解读公式科学含义；仅做结构登记与定位。

## 6. 锚点格式（source_anchor / source_anchor_map）
- 单块锚点五要素：`block_id` + `page` + `bbox` + `basis` + `confidence`。
- `basis` 取值示例："文本层逐字提取" / "OCR 后处理校验" / "题注+表格几何推断" / "人工可见但低清"。
- `source_anchor_map` 的 `claim` 必须来自 `task.md` 要求的证据范围；一条结论一个锚点，不笼统写"第 X 页"。
- 锚点粒度与可核验性匹配：关键结论精确到 bbox；一般字段精确到 block_id 即可。
