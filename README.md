# 参考文献与引用管理 Skill（citation-consistency-manager）

一个把"稿件 + 文献库"转成"可核验引用成稿"的 Agent Skill。核心：引用定位台账 + 编号同步 + 去重归并判据 + 来源核验映射 + 格式转换记录 + 改写核对分离。

## 目录结构
- `SKILL.md` —— Skill 本体（含 YAML frontmatter、CRC 工作流、输出契约）
- `references/` —— 方法细则、自检清单、字段字典、引用规则、目标规范档
  - `method.md` —— CRC 四机制细则与反例库
  - `checklist.md` —— 覆盖"挂得对/编得顺/收得净/对得上"的分组自检清单
  - `schema.json` —— result.json 的 JSON Schema（draft-07）
  - `citation_rules.md` —— 引用形式、去重分层判据、字段映射表、术语规范
  - `format_profiles.md` —— GB/T 7714—2015、APA 7 等目标规范配置档
- `assets/manuscript_template.md` —— 可编辑稿件与文末列表模板
- `examples/` —— 最小调用示例（已标注为示例，非真实案例）
- `GITHUB_URL.txt` —— 提交前填 GitHub 公开链接

## 使用方式
1. 将本目录作为 Skill 安装到 Agent（平台约定路径 `/skills/weekly-skill`）。
2. 输入 `/input/case.json`（稿件、文献库、目标规范、任务说明）与 `task.md`，自然语言下指令如"整理引用并重排编号"。
3. 按 SKILL.md 的 SOP（Step 0~7）执行，输出到 `/app/output/result.json`、`manuscript_with_citations.md` 与 `reference_list.md`。

## 依赖
仅需文本处理能力；无外部运行时依赖。引用形式、去重判据、目标规范映射内置于 SKILL.md 与 `references/`，无需联网即可按材料工作；联网核验仅在材料允许且可用时执行，失败须记为 `unverified`。

## 适用与限制
- 适用：中英文科研稿件的引用插入定位、编号重排、去重归并、BibTeX/RIS/EndNote/CNKI 与 GB/T 7714/APA 格式转换、来源核验与正文—文末一致性检查。
- 限制：材料缺失、DOI 无法解析、条目歧义、稿件不可编辑时会标记 `partial`/`blocked`，不编造条目、DOI、数值或 benchmark 分数；不替代作者/领域专家对论断价值的最终判断。

## 与评分维度的对应
- 实际效果(40)：SOP Step 2~6 分别落实定位、编号、去重、格式、核验六类动作；输出契约的各类台账提供逐条回溯（见 SKILL.md 第 3、4 节）。
- 使用增益(25)：以"四账一表"结构化产出替代逐条手工核对，减少返工；增益对照由平台白板给出，本作品不自带分数（见第 2、7 节）。
- 创新增益(15)：编号连锁更新映射、正文—列表双向同步、论断级引用匹配、可解释去重判据为新增能力（见第 2 节与 `references/method.md`）。
- 可靠体验(10)：异常处理表覆盖 DOI 失败、条目歧义、编号冲突、格式不支持等情形（见第 6 节）。
- 生态贡献(10)：完整规则文件、规范配置档、模板、示例与反例库可供复用（见第 8 节与 `references/`）。

## Runtime contract
平台加载 Skill 到 `/skills/weekly-skill`，输入在 `/input/case.json` 与 `task.md`，输出到 `/app/output/result.json`（含全部输出契约字段与 `case_id`）。资源最多 50 个 UTF-8 文本文件、单文件 ≤500KB、总计 ≤2MB、无软链。执行预算 64 次模型请求 / 900 秒 / 每次 16384 token。
