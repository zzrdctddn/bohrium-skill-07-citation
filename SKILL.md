---
name: citation-consistency-manager
description: 把"一份稿件 + 一个文献库"转成"引用挂得准、编号编得顺、重复收得净、来源对得上"的可复核引用管理成稿。当用户要求"整理参考文献""插入/重排引用编号""去重合并文献条目""GB/T 7714 与 APA 格式转换""核验引用真实性""检查正文与文末一致性"时使用。输出含引用定位台账、编号映射表、去重决策表、来源核验表、格式转换记录、可编辑稿件与自检结果。
version: 1.0.0
license: MIT
---

# 参考文献与引用管理（Citation-Renumbering & Consistency）

> 面向真实稿件与真实文献库，把"整理引用"从逐条手工核对变成**可回溯的工程流程**。核心不是排版出漂亮的文献表，而是让每一条引用都能回答三个问题："挂对位置了吗？编号同步了吗？来源真实且匹配吗？"

## 1. 用途与适用边界

**解决什么**：基础 Agent 做引用管理的常见失败——引用堆在段尾或图表注附近、增删调序后正文编号与文末列表不同步、同一文献的中英文/预印本正式版/多库导出重复成多条、编造或错配元数据、正文论断与所引文献主题对不上。本 Skill 用"四账一表（定位台账 / 编号映射 / 去重决策 / 来源核验 / 格式转换记录）+ 核对与改写分离"消除这些失败。

**触发场景**：为稿件插入引用并选位置与形式；增删/调序/合并章节后重排编号；合并重复文献条目；在 BibTeX/RIS/EndNote/CNKI 导出与目标规范（GB/T 7714—2015、指定 APA 版本、顺序编码、著者—出版年、脚注）之间转换；核验条目真实性与元数据；逐句判定正文论断与所引文献是否匹配。

**不该用**：稿件与文献库都为空且不允许核验时（应报告材料缺口，不凭标签补造条目）；用户只要改排版样式而不要引用逻辑核对时（转用排版 Skill）；无 DOI/原文且任务禁止联网核验却要求"确认真实性"时（只能标 `unverified` 进人工确认，不得宣称已核实）。

**输入**：`/input/case.json`（含 `case_id`、任务说明、稿件文本、文献库输入、目标引用规范与约束）与 `task.md`。**输出**：`/app/output/result.json`（字段见第 4 节）+ 可编辑的 `manuscript_with_citations.md` 与 `reference_list.md`。

## 2. 核心方法（CRC）

本 Skill 的方法名为 **CRC = Citation-Renumbering & Consistency（引用定位—重排—一致性管理）**，四个不可跳过的机制，分别对应题面四类关键问题：

1. **挂得对 — 引用定位台账（Citation Placement Ledger）**：为稿件中每个"需要来源支撑的论断句"分配 `claim_id`，把引用绑定到**该句句末/句内**而非段尾或图表注；标注引用形式（顺序编码 / 著者—出版年 / 脚注）与一句多引、多句一引的处理规则。任何"看起来需要出处却没绑定来源"的句子标 `needs_source`，绝不静默略过。
2. **编得顺 — 编号同步引擎（Renumbering Sync）**：维护一张"旧编号 → 新编号"映射表；任何插入、删除、调序、合并之后，按首次出现顺序重排，并**同步刷新**正文标记、文末列表序号、图表注、附录引用与交叉引用，保证编号连续、上标样式统一、无孤儿引用、无未被引用条目（除非任务允许）。
3. **收得净 — 去重归并判据（Dedup & Merge Criteria）**：对同一文献的中英文题名、预印本与正式版、期刊与会议版、不同库导出字段差异，按**分层判据**（DOI 完全一致 → 题名+首作者+年份一致 → 题名模糊+作者+年份人工确认）给出可解释的合并依据；对同名作者的不同文献设"禁止误合并"护栏；每次合并写 `merge_rationale`。
4. **对得上 — 来源核验映射（Source Verification Map）**：逐条核验引用真实存在、元数据（作者/年份/题名/卷期页/DOI）准确；逐句判定正文论断的主题与结论是否真的落在所引文献上；证据不足的句子标 `insufficient_evidence` 并进 `human_review_items`。

另有两条贯穿铁律：**改写与核对分离**（先完成改写，再用全新视角独立执行第 7 节核对，两遍不可合并）；**不编造事实与 benchmark**（材料没有的条目、DOI、数值一律不补；核验失败如实记为 `unverified`/`blocked`）。

## 3. 工作流（SOP）

严格按顺序执行，每步都有检查点；检查点不过则回退该步或转入异常处理。

**Step 0 · 输入解析与边界确认**
- 读 `case.json` 与 `task.md`，提取：任务要求的动作（插入定位 / 编号更新 / 去重 / 格式转换 / 来源核验 / 一致性检查）、稿件与文献库材料、目标规范、明确禁止事项。
- 检查点：列出本案例需要执行的动作清单；任一必需材料缺失则登记 `uncertainties`，不要凭领域标签补造。

**Step 1 · 稿件与文献库快照（Snapshot）**
- 保存原稿与文献库原始快照（用于回溯 diff）；为稿件建立**句子索引**（段号-句号）；为文献条目建立主键清单（DOI、题名、首作者、年份、类型、来源库）。
- 检查点：原文与文献库逐字保留可回溯；条目主键缺失的标 `missing_key` 并进候选核验。

**Step 2 · 引用定位（挂得对 / Placement Ledger）**
- 扫描每个论断句，判断是否"需要来源支撑"（他人结论、具体数值、事实主张）；需要的分配 `claim_id` 并绑定候选来源，确定插入位置与引用形式；处理一句多引与多句一引。
- 检查点：无引用堆在段尾/图表注；所有"需要来源"的句子都有 `claim_id`；无来源的进 `needs_source` 而非编造。

**Step 3 · 编号同步（编得顺 / Renumbering Sync）**
- 建立引用编号索引；按任务指定的编号制（顺序编码按首次出现、著者—出版年按字母/拼音）编排；发生增删调序时生成"旧→新"映射，并同步正文标记、文末列表、图表注、附录与交叉引用。
- 检查点：编号连续无跳号；正文与文末一一对应；无孤儿引用与未引用条目（除非允许）；上标/括号样式统一。

**Step 4 · 去重归并（收得净 / Dedup Criteria）**
- 按分层判据扫描重复：DOI 一致先并；题名+首作者+年份一致再并；疑似重复（预印本/正式版、中英文题名、会议/期刊版）列候选并人工确认。记录每条合并的判据与理由。
- 检查点：无同名作者不同文献被误并；每条合并有 `merge_rationale`；合并后编号已在 Step 3 基础上重新同步。

**Step 5 · 格式转换与字段补齐（Format Map）**
- 在来源格式与目标规范间做字段映射：保留 DOI、卷、期、页、文献类型标识、中英文标点差异；缺失字段标 `missing_field` 而非臆造；按目标规范统一作者名、年份、标点、著录项顺序。
- 检查点：目标规范必填项齐全；关键字段无丢失；中英文混排标点符合规范。

**Step 6 · 来源核验（对得上 / Verification Map）**
- 通过 DOI / 数据库 / 原文比对核验条目真实性与元数据；逐句判定正文论断与所引文献主题、结论是否匹配；对证据不足或无法核验的标 `unverified`/`insufficient_evidence` 并进人工确认。
- 检查点：每条引用有真实性结论；每个论断—来源匹配有依据；不确定的绝不写成"已核实"。

**Step 7 · 独立核对与输出**
- 以审阅者身份逐项走 `references/checklist.md`；生成 `manuscript_with_citations.md`、`reference_list.md` 与 `result.json`；所有改动写入 `change_log`，未通过项写清原因。
- 检查点：自检表全过或每项未过均有 `change_log`；`status` 与完成情况一致；示例文本明确为示意。

## 4. 输出契约（result.json）

```json
{
  "case_id": "<原样带回>",
  "status": "success | partial | blocked",
  "note": "示意文本请标注，非真实案例",
  "artifact": {
    "manuscript": "<带引用的可编辑稿件正文>",
    "reference_list": "<按目标规范排好的文末条目>",
    "citation_style": "<numeric | author-year | footnote>",
    "target_spec": "<GB/T 7714-2015 | APA7 | ...>"
  },
  "citation_placement_ledger": [
    {"claim_id":"C1","sentence_loc":"§2 p1 s3","claim_text":"...","needs_source":true,"position":"句末","form":"numeric","sources":["REF-2"],"status":"placed|needs_source"}
  ],
  "renumber_map": [
    {"old":"[7]","new":"[3]","scope":"正文+文末+图注","reason":"删除原[3]后顺延"}
  ],
  "dedup_decisions": [
    {"group_id":"G1","members":["REF-4","REF-9"],"merge_key":"10.1000/xyz","criteria":"DOI完全一致","merge_rationale":"同一DOI，题名中英差异，合并为一条","action":"merged"}
  ],
  "source_verification": [
    {"claim_id":"C1","entry_key":"REF-2","exists":"yes|no|unknown","metadata_accuracy":"match|mismatch","claim_match":"match|partial|mismatch","basis":"DOI解析命中/原文p3","status":"verified|unverified"}
  ],
  "format_conversion_log": [
    {"entry_key":"REF-2","from":"BibTeX","to":"GB/T 7714-2015","fields_preserved":["DOI","卷期页"],"fields_missing":[]}
  ],
  "fact_source_map": [
    {"item":"REF-2 卷期页","value":"12(3):45-52","source_location":"input refs/ref2.bib"}
  ],
  "uncertainties": ["..."],
  "human_review_items": ["..."],
  "change_log": [{"step":4,"action":"合并REF-4与REF-9","reason":"DOI一致"}],
  "self_check": {
    "placement_correct": true,
    "numbering_synced": true,
    "dedup_clean": true,
    "sources_verified": true,
    "no_fabrication": true
  }
}
```

字段原则：`status` 诚实反映完成情况；`partial`/`blocked` 必须写清 `uncertainties`；核验失败标 `unverified` 而非"已核实"；不得编造 DOI、条目或 benchmark 分数；示例一律标注为示意。

## 5. 质量自检清单

执行前完整读取 `references/checklist.md`，逐项打钩。关键硬项：

- [ ] 每个需来源支撑的论断句都有 `claim_id` 且绑定具体位置，无段尾堆砌
- [ ] 编号连续、正文与文末一一对应、图表注/附录同步、样式统一
- [ ] 重复条目按分层判据合并，每条有理由，无误并同名作者不同文献
- [ ] 每条引用真实性有结论，元数据准确，论断—来源主题匹配
- [ ] 目标规范必填项齐全，关键字段（DOI/卷期页/类型）无丢失
- [ ] 无材料外新增条目/DOI/数值；不确定项进 `human_review_items`
- [ ] 改写与核对分离，所有改动有 `change_log`；`status` 诚实

## 6. 异常处理与边界

| 情况 | 处理 |
|---|---|
| 文献库为空 / 稿件无论断句 | `status=blocked`，登记缺口，不硬造条目或引用 |
| DOI 解析失败 / 无法联网 | 标 `unverified`，进 `human_review_items`，成稿用中性表述，不宣称已核实 |
| 条目元数据歧义（同名作者） | 不自动合并，列候选交人工，写清歧义点 |
| 编号冲突 / 跳号 | 重建编号映射表，回退 Step 3 重新同步，写 `change_log` |
| 目标规范未知 | 采用材料/任务指定规范；无则默认顺序编码 + GB/T 7714—2015，并在 `uncertainties` 说明 |
| 稿件格式不支持（扫描件/损坏） | 报告格式问题，`status=blocked`，说明需要可编辑文本 |
| 预印本与正式版并存 | 按 Step 4 分层判据列候选，默认保留正式版并说明，不确定交人工 |

## 7. 最小调用示例（示意，非真实案例）

**输入**（`case.json` 片段）：
```json
{"case_id":"demo-007","task":"为稿件插入引用并按 GB/T 7714-2015 整理文末列表","params":{"cite_style":"numeric","target_spec":"GB/T 7714-2015"},"materials":{"manuscript":"manuscript.md","library":["refs.bib"]}}
```
**调用**：加载本 Skill 后，用户说"用 citation-consistency-manager 整理这篇稿件的引用"。

**示意输出**（节选）：
```json
{"case_id":"demo-007","status":"success","artifact":{"citation_style":"numeric","target_spec":"GB/T 7714-2015"},"citation_placement_ledger":[{"claim_id":"C1","position":"句末","sources":["REF-2"],"status":"placed"}],"renumber_map":[{"old":"[5]","new":"[2]","reason":"去重后顺延"}],"self_check":{"numbering_synced":true,"no_fabrication":true}}
```
> 上述为演示接口用的虚构示例，不代表任何真实案例结果。

## 8. 资源清单

- `references/method.md` —— CRC 方法细则与反例库
- `references/checklist.md` —— 覆盖四类关键问题的逐项自检清单
- `references/schema.json` —— 输出字段字典（JSON Schema）
- `references/citation_rules.md` —— 引用形式、去重判据、字段映射领域规则
- `references/format_profiles.md` —— GB/T 7714、APA 等目标规范配置档
- `assets/manuscript_template.md` —— 可编辑稿件与文末列表模板
- `examples/` —— 最小调用示例（标注为示例，非真实案例）
