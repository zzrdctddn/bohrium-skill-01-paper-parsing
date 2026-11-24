# 文献文档解析自检清单（逐项打钩）

> 以"审阅者"身份独立执行，不与解析阶段合并。每项不通过须回退对应步骤并在 `change_log` 记录。

## 提得全（Layout Block Inventory）
- [ ] 标题、作者、单位、摘要逐字提取
- [ ] 正文逐字提取，无漏行、无丢段
- [ ] 图表题注逐字提取并与图/表对应
- [ ] 脚注逐字提取，独立归类
- [ ] 参考文献逐条提取，独立归类
- [ ] 页眉页脚单独归类，**未混入正文**
- [ ] 每个区块有 block_id / page / bbox / confidence / status
- [ ] 缺字、乱码、失败区块已登记，未静默省略

## 排得对（Reading Order Reconstruction）
- [ ] 双栏按"左栏到底→右栏"排序，未串栏
- [ ] 多栏按栏分组、栏内 y 升序
- [ ] 通栏标题/表格/公式插在正确顺序点
- [ ] 跨页段落未拼接成一句，保留段落边界
- [ ] 每块有 reading_order_index，顺序连续可复核

## 拆得清（Structure Reconstruction）
- [ ] 章节层级、编号、parent 指针正确
- [ ] 表格行列数、单元格归属逐格核对
- [ ] 合并单元格用 rowspan/colspan 表达，未改写成重复文本
- [ ] 多层表头用 header_rows 标记
- [ ] 跨页续表用 continuation 链接并各自保留页码
- [ ] 公式上下标、希腊字母、编号与原文逐字一致
- [ ] 符号存疑标 symbol_uncertain，未解读科学含义

## 核得回（Per-Block Provenance）
- [ ] 每块有 source_anchor（页码+bbox+basis+confidence）
- [ ] 任务要求的关键结论均有 source_anchor_map 回链
- [ ] 读不出内容标 unreadable，进 human_review_items
- [ ] 无法判断/冲突内容标 low confidence，进 human_review_items
- [ ] 未生成"看似合理"的替代文字

## 结构与边界
- [ ] artifact 含 title/authors/affiliations/abstract/sections/figures/tables/reading_order
- [ ] status 诚实（success/partial/blocked）
- [ ] partial/blocked 写明原因与缺口
- [ ] 无材料外新增文字、数值、benchmark 分数或软件状态
- [ ] 以真实 PDF 内容为唯一事实依据，不据领域标签补造数据
- [ ] 独立核对完成，所有改动均在 change_log
