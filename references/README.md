# References

> 本目录是研究 agent 工作的**外部资料索引**——KWeaver 内部资料经脱敏后存档于此，学术论文以 arXiv ID 索引（PDF 不入库）。
> 目的：让研究 agent 在干净启动时（无 `~/Downloads` 上下文）也能完整推进研究工作。

---

## 文件清单

### KWeaver 项目资料（脱敏后入库）

| 文件 | 类型 | 内容 |
|------|------|------|
| `01_overcoming_ontology.md` | 方向性纲领 | KWeaver 的 AI 工程能力如何超越本体论（飞轮、三层壁垒、Agent-first）|
| `02_engineering_roadmap.md` | 工程路线图设计稿 | 飞轮内功 / 标杆 / 主轴三层结构，Scaling 审视表，组件现状与差距 |
| `03_kweaver_core_overview.md` | 产品架构概览 | 整体架构、核心组件、BKN Lang、VEGA、ISF、TraceAI 的功能定位 |
| `04_kweaver_core_deep_analysis.md` | 技术深度分析 | 非结构化数据问答的可靠性挑战 + 145 样本消融实验 + 业界对比 |
| `05_harness_engineering_position.md` | 业界趋势分析 | Prompt → Context → Harness 三阶段演进，KWeaver 在生态中的位置 |

### 学术资料

| 文件 | 内容 |
|------|------|
| `papers/INDEX.md` | 9 篇相关论文 arXiv ID + 一句话定位 + 与飞轮环节的映射 |
| `papers/*.pdf` | **不入库**（gitignored）。由**外部的 researcher 项目**负责 fetch / 同步——本项目只维护索引与笔记，不负责拉取动作 |

### 详细笔记

每篇学术论文的 7-Block 深读笔记在 `../notes/` 目录下，文件名以编号开头（`01_signals_*` 至 `08_tide_trace_*`）。

---

## 脱敏规则（Desensitization Rules）

KWeaver 是开源项目（GitHub repo: `~/dev/github/kweaver/`），技术性内容本来就要公开传播。脱敏只针对**公司层面**的敏感信息：

### 必须移除

- 财务 / 估值 / ARR / GMV / 定价信息
- 客户名称、Logo、合同金额
- 内部团队规模、汇报线、个人姓名（保留作者署名 OK）
- 内部 Q3/Q4 时间承诺
- 商业合作的具体条款（保留"已公开宣布"的部分）
- 商务向 sales 表述（"投资建议"、"积极关注"等）

### 必须保留

- **全部技术架构、组件设计、接口**（这是开源项目本来就要公开的）
- 飞轮 / Scaling / Bitter Lesson 等方法论
- 性能 benchmark（已经在公开材料中露出）
- 路线图里的方向性内容（删具体日期）
- 与其他产品（LangSmith、Palantir 等）的技术性对比
- 公开宣传的优化指标 talking points（如"Token 降 30%+"、"准确率 92%+"）

### 灰色地带（按需判断）

- 未公开的客户案例细节 → 抽象为"某金融客户"或删除
- 未公开的具体性能数字 → 模糊化为"数量级"描述
- 内部组织架构细节 → 删除
- 路线图中的"现状与差距" → 工程信息可公开（开源代码本来可见）

### 应用流程

新材料入库时：
1. 复制原文件到 `references/` 暂存目录
2. 按上述规则逐段过滤
3. 在文件头加入"脱敏处理"说明（"无脱敏"、"移除 X"、"软化 Y" 等）
4. PR review 时另一人 sanity check
5. 通过后入主目录

---

## 与原始材料的关系

每个 `references/*.md` 文件的"来源"字段标明原始位置（如 `~/Downloads/...`），**保持原始文件不入库**。当原始文件更新时，重新执行入库流程，**不在 references/ 中维护版本历史**——版本历史仍以原始文件为准。

---

## 给后续 research agent 的指引

如果你是后续接手研究的 agent：

1. **先读 `01_overcoming_ontology.md`**——理解 KWeaver 的核心定位（飞轮、Agent-first、超越本体论）
2. **再读 `02_engineering_roadmap.md`**——理解当前差距和路线图
3. **需要技术细节时**读 `03` / `04`
4. **需要对外位置感时**读 `05`
5. **学术论文**：用 `papers/INDEX.md` 找论文，配合 `notes/` 目录下的 7-Block 深读笔记
6. **核心产出**：项目根目录 `report.md` 是综合产出（v2.0），`notes/00_research_landscape.md` 是研究地图

**重要约束**：
- 引用 references 内容时使用相对路径（如 `references/02_engineering_roadmap.md#层-1-内功`）
- **不要把 references 内容大段复制到对外产出中**——用引用 + 关键事实摘录的方式
- 如果发现 references 与 KWeaver 实际产品演进有偏差，标注出来并请求人工 sanity check
- 写技术报告时**优先使用 KWeaver 自己的术语**（双轨 Trace、BKN、ISF、Triage Agent 等），不要回退到泛学术术语

---

## 项目边界

**本项目（research-agent-triage）的职责：**
- 维护 `references/` 索引和脱敏后的 KWeaver 项目资料
- 维护 `notes/` 学术论文深读笔记
- 维护 `report.md` 综合产出

**不属于本项目职责（由外部 researcher 项目负责）：**
- 自动 fetch 学术 PDF（按 `papers/INDEX.md` 中的 arXiv ID）
- 从 `~/Downloads` 同步原始 KWeaver 材料
- 定期巡检学术界新论文并更新索引

外部 researcher 项目会向本项目提供 `papers/*.pdf`（落在 gitignored 路径）和原始材料（落在 `refs-staging/`，本地脱敏处理后再入 `references/`）。

## 可选 follow-up（本项目内）

- [ ] PR template：入库新 reference 时的 checklist
- [ ] 自动化检查：CI 检查 references/ 中是否有意外 leak 的敏感信息（如金额/客户名 regex）
