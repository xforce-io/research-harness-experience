---
zone: active
tags: []
pin: false
score: 0.4063829787234043
dwell: 1
---
# SKILL.state: Scalable Long-Horizon Agent Skills

> Topic integration note derived from Library read artifact `.researcher-workspace/library/papers/paper_arxiv_2608_26263/reads/read_paper_arxiv_2608_26263.md`.

## Library read

---
kind: library-read-identity
authors: ["Sanket Badhe","Priyanka Tiwari","Jonghyun Chung"]
source_id: "arxiv:2608.26263"
source_url: "https://arxiv.org/abs/2608.26263"
pdf_url: "https://arxiv.org/pdf/2608.26263"
---
先读取常驻技能与完整提示，再按 Library 深读规范产出证据卡。接着读完整提示里的输出约定与论文正文。继续读取输出模板与论文全文关键部分。继续读实验与结果，并对照一篇既有 Library read 的写法。已掌握机制、评测与边界，下面按 Library 规范只产出证据卡正文。

# SKILL.state: Scalable Long-Horizon Agent Skills

> 把推理/动作/观测一路追加进 transcript 导致 prompt 随步数膨胀、过时事实污染决策 → 每步只喂不可变技能说明 \(P\)、结构化执行状态 \(\Sigma_t\)、最新观测 \(O_t\)，推理用完即丢、只合并校验后的状态补丁 → 单步上下文近似 \(O(1)\)、累计 token 近似 \(O(T)\)。

## Essence

**问题**  
长程 procedural skill 的主流 runtime 仍是对话式：历史推理、动作与观测不断追加，prompt 随 horizon 单调增长；过时观测与废弃推理留在上下文里，模型每步都要从文本历史里“重建当前世界”。

**做法**  
SKILL.state 把执行改成显式状态转移：每步输入 \(A_t=(P,\Sigma_t,O_t)\)；模型产出 \((R_t,\Delta\Sigma_t,a_t)\)；runtime 校验 \(\Delta\Sigma_t\) 后做字典合并 \(\Sigma_{t+1}=\Sigma_t\oplus\Delta\Sigma_t\) 并执行 \(a_t\)；\(R_t\) 与历史观测/动作永不进入后续 prompt。Schema 按域一次性编写（如 InterCode CTF 100 题共用 5 个字段），状态只保留未来决策所需信息。

**证据**  
Warehouse、Gemini-3-Flash、\(T=100\)：SKILL.state 准确率 \(0.94\)、累计约 \(65\mathrm{k}\) tokens；Stateful（LangGraph 式，状态+全文 transcript）约 \(1.06\mathrm{M}\) tokens（约 \(16.2\times\)），准确率 \(0.91\) [Table 1]。

**边界**  
这不是“更强压缩器”或通用长期记忆：硬依赖是——域内可预先写出足以充当充分统计量的结构化 schema，且任务目标不要求保留完整交互轨迹本身（审计/溯源类任务不适用）。

## Claims

- 每步条件化为 \(A_t=(P,\Sigma_t,O_t)\)：模型不再接收既往观测、动作或推理；步内可保留多步 CoT，但校验并应用状态转移后，推理迹 \(R_t\) 被永久丢弃 [§3, §3.2]。
- 状态更新是结构化补丁 \(\Delta\Sigma_t\)（键级变更/删除的 JSON），经 runtime 确定性校验后按带 null 删除语义的字典合并写入 \(\Sigma_{t+1}\)；非法补丁不污染持久状态，触发回滚重试 [§3.2, §7]。
- 对话式 runtime 的单步上下文 \(|C_t|=O(t)\)、累计 token \(O(T^2)\)；SKILL.state 单步 \(|P_t|=O(|P|+|\Sigma|+|O|)\) 与已执行步数无关，累计 \(O(T)\) [§3.3]。
- Schema 按域而非按任务编写；InterCode CTF 100 个挑战复用同一静态 5 字段 schema（`discovered_flags` / `tested_hypotheses` / `active_files` / `working_dir` / `cmd_summary`）[§3.1]。
- 噪声鲁棒性来自机制而非更强注意力：干扰遥测在生成状态补丁时被过滤，不会进入后续 prompt；Warehouse \(T=50\)、高噪声下 Prompt 准确率从 \(0.68\) 降到 \(0.53\)，SKILL.state 维持 \(\geq 0.97\) [§5.3, Table 2]。
- 外部环境静默漂移时，历史基线因旧事实压过新观测需 \(5\)–\(8\) 步才恢复；SKILL.state 依赖当前结构化状态，报告为 \(0\) 恢复步 [§5.4, Table 3]。
- 在约 \(1{,}800\) token 预算对齐下，截断窗口 / Summary-capped / ReAct+LLMLingua 在 Warehouse \(T=100\) 分别掉到 \(0.18\) / \(0.52\) / \(0.22\)，SKILL.state 为 \(0.94\)——增益不能只归因于“更短 prompt”，而依赖结构化状态保留关系依赖 [§5.6, Table 5]。
- 开源模型主要失败模式是结构化输出遵从而非“不会推理”：Gemma-4-31B、\(T=100\) 错误中 \(68\%\) 为过早覆盖/删除既有键，\(20\%\) 类型/schema 不一致，\(12\%\) JSON 语法 [§5.7]。
- Warehouse 长程缩放（Gemini-3-Flash）：\(T=200\) 时 SKILL.state 准确率 \(0.94\)、累计约 \(122\mathrm{k}\) tokens；Memory 基线累计约 \(6.18\mathrm{M}\) tokens、准确率 \(0.84\)；Prompt 准确率降至 \(0.74\) [Table 1]。
- InterCode CTF：pass@1 \(54.2\%\)（最强基线 Memory \(46.4\%\)，Stateful \(41.8\%\)），相对 ReAct / Stateful 分别少约 \(60.4\%\) / \(65.9\%\) 总 token [Table 4]。
- Sierra \(\tau\)-Bench：Retail pass \(58.3\%\)（Stateful \(51.7\%\)）；Airline pass \(32.4\%\)（Stateful \(28.1\%\)），平均 prompt 约 \(2{,}800\) vs 基线峰值更高、总 token 相对 ReAct / Stateful 约省 \(40.5\%\) / \(45.4\%\) [Table 4]。

## Assumptions

- 执行状态可被做成未来决策的充分统计量：凡影响后续动作的信息都能在首次知悉时投影进 schema；否则丢弃历史不是无损的 [§7]。
- 相关状态结构可预先按域固定；评测不覆盖“执行中动态发现 schema”的设定 [§3.1, §7]。
- 模型能可靠产出可校验的结构化补丁；强模型（如 Gemini-3-Flash）下该假设大致成立，小开源模型上大量失败来自合并/类型/JSON 遵从 [§5.7, §7]。
- 主实验解码固定 temperature \(0.0\)、top-p \(1.0\)，把方差压到可复现，但不代表采样解码下的稳健性 [§5.1]。
- “Stateful / LangGraph-style”基线定义为：结构化状态块 + 完整滚动对话 transcript；据此主张 SKILL.state 替换的是推理底物而非否定图编排本身 [§2.2, §5.1]。
- SkillExecBench 的 Warehouse（500 独立货架槽位）与 Software Repository 图状态，与“显式槽位/图式状态”高度同构——合成诊断床偏向状态中心执行 [§4.1] `[low]`。

## Method

**前后对比（执行底物）**

| | 对话式 / ReAct | Memory（摘要） | Stateful（LangGraph 式） | SKILL.state |
|---|---|---|---|---|
| 每步主要条件 | \(P\) + 全历史 transcript | \(P\) + NL 摘要 + 近 3 步 | \(P\) + 结构化状态 + 全历史 | 仅 \((P,\Sigma_t,O_t)\) |
| 推理迹去向 | 追加进历史 | 进入摘要/窗口 | 追加进历史 | 投影进 \(\Delta\Sigma_t\) 后丢弃 |
| 上下文随 \(T\) | \(O(T)\) / 累计 \(O(T^2)\) | 仍随摘要与窗口增长 | 仍随 transcript 增长 | 单步近似 \(O(1)\) |

**执行环（Algorithm 1）**

1. **输入**：不可变 procedural specification \(P\)；当前结构化状态 \(\Sigma_t\)；环境最新观测 \(O_t\)。
2. **模型计算**：生成 \((R_t,\Delta\Sigma_t,a_t)\)——步内可多步推理；\(\Delta\Sigma_t\) 为 JSON 补丁，`null` 表示删除键。
3. **Runtime**：确定性校验 \(\Delta\Sigma_t\) → \(\Sigma_{t+1}=\Sigma_t\oplus\Delta\Sigma_t\) → 执行 \(a_t\)；非法补丁回滚重试，不写入 \(\Sigma\)。
4. **输出到下一步**：只保留更新后的 \(\Sigma_{t+1}\) 与新观测；\(R_t\)、旧观测与旧动作不再出现在 prompt。

**设计取舍（正文明确）**  
相对 DST：DST 在准静态对话旁挂槽位且仍保留全文；SKILL.state 把结构化状态当作充分统计量并丢掉对话历史。相对压缩：不做“先堆历史再压缩”，而是从机制上阻止历史累积。

## Eval

- **数据 / 环境**：SkillExecBench（Warehouse 库存；Software Repository 分支/PR/CI 图）；InterCode CTF（100 个 Linux bash CTF）；Sierra \(\tau\)-Bench Retail / Airline（工具–用户–策略约束客服）[§4]。
- **主基线**：Prompt（ReAct 式全量追加）；Memory（滚动 3 步 + 周期性 NL 摘要，对标 MemGPT 式思路）；Stateful（结构化状态 + 全量 transcript，对标 LangGraph 式）[§5.1]。
- **预算对齐对照**：Truncated sliding window；Summary-capped；ReAct + LLMLingua；目标预算约与 SKILL.state 单步 footprint 对齐（\(\sim 1{,}800\) tokens）[§5.6, Table 5/11]。
- **模型**：Gemini-3-Flash（主表）；Gemma-4-31B-it、Qwen3-8B-it（附录缩放）；temperature \(0.0\) [§5.1, Table 7–8]。
- **指标**：任务准确率 / pass@1 / 官方 programmatic pass；平均 prompt tokens；全轨迹累计 tokens；噪声下 score；状态恢复步数 [§4.3]。
- **统计**：合成实验 5 个 procedural seed，报 mean±SD；文称 \(T\geq 50\) 时相对基线 paired t-test \(p<0.01\) [§5.1]。
- **主结果锚点**：Table 1（Warehouse 缩放）、Table 2–3（噪声/恢复）、Table 4（公开基准）、Table 5（预算对齐）、附录 Table 6–11（Software 缩放、开源模型、噪声/恢复扩展）。

## Weaknesses

- SkillExecBench 由作者构造且状态几乎就是“应被记住的槽位/图”，与方法假设同构；缺少“状态结构与自然语言目标不对齐”的压力测试，长程增益可能被诊断床放大 [§4.1, Table 1]。
- Memory 基线在 Warehouse \(T=200\) 累计 token（\(\sim 6.18\mathrm{M}\)）反而高于无界 Prompt（\(\sim 2.61\mathrm{M}\)），说明摘要实现未形成有效封顶；用它代表“压缩/记忆路线”会夸大相对优势，也削弱与生产级 MemGPT/Mem0 的可比性 [Table 1, §5.1]。
- \(\tau\)-Bench Retail 上 SKILL.state 平均 prompt（\(3{,}325\)）高于 ReAct（\(2{,}819}\)）与 Memory（\(2{,}737\)），与“处处更小上下文”的叙事不完全一致；总 token 更低可能混杂回合更短/更早成功，文中未分解 [Table 4]。
- 开源模型上机制红利变脆：Gemma \(T=100\) 时 SKILL.state 与 Stateful 同为 \(0.42\)；错误 taxonomy 指向格式遵从，但正文未报告 grammar-constrained decoding 的对照结果，却把“结构化输出”写成可部署前提 [§5.7, Table 7, §7]。
- Schema 质量与字段选择是未消融的自由参数：InterCode 的 `tested_hypotheses` / `cmd_summary` 实质是压缩后的历史备忘，接近“把 transcript 手工搬进状态”，与“无历史”主张存在张力，文中未量化字段设计对分数的贡献 [§3.1]。
- 缺少与强检索/分页记忆（完整 MemGPT OS、向量 episodic memory）及真实 LangGraph 生产配置的对照；现有 Stateful 基线仍强制携带全量 transcript，可能不是该生态最强状态用法 [§2.2, §5.1]。

## Relations

- builds-on ReAct 式逐步“推理–动作–观测”循环 [high]：仍逐步产出动作并与环境交互，但把条件化底物从追加 transcript 换成 \((P,\Sigma_t,O_t)\)；正文以 Yao et al. 2022 为 Prompt 基线 [§5.1]。
- competes-with MemGPT / 摘要式记忆 runtime [med]：同打长程上下文膨胀，路径相反——Memory 基线保留摘要+近窗对话语义，SKILL.state 丢弃对话历史、只留可变结构化状态；正文引用 Packer et al. 2023 作为 Memory 范式 [§2.2, §5.1]。
- competes-with LLMLingua 等统计式 prompt 压缩 [high]：Experiment 5 在同预算下直接对比，压缩毁掉关键槽位标识而结构化状态保留关系依赖 [§5.6, Table 5]。
- extends Dialogue State Tracking（DST）的结构化槽位思想 [med]：§2.3 明确对比——DST 在全文对话旁维护辅助状态，SKILL.state 把状态当作充分统计量并丢掉对话历史 [§2.3]。
- competes-with LangGraph 式“辅助结构化状态 + 对话 transcript”编排 [med]：Stateful 基线即该设定；论文主张真正替换的是推理底物，而非否定图工作流编排 [§2.2, §5.1]。
- orthogonal Recuris（Experiential–Working Memory 演化）[low]：两者都强调长程任务需要紧凑可信的工作状态，但 Recuris 面向技能记忆跨任务演化与门控补丁，SKILL.state 面向单技能执行时丢弃推理、固定 schema 的 \(O(1)\) prompt；问题邻域重叠、机制目标不同。

## Takeaway

- **可借用**：把“步内推理”与“跨步持久状态”拆开——推理只服务本步补丁与动作，持久层用可校验的结构化 merge，而不是继续追加自然语言历史。
- **需复核**：合成诊断床与 schema 设计可能过度偏袒方法；Retail 上平均 prompt 并不总更短；小模型结构化遵从未用受限解码实证收口前，不要把 \(O(1)\) footprint 当成与模型能力无关的纯系统收益。
- **只记一句**：长程 skill 执行的可扩展底物不是更会摘要的聊天记录，而是每步只看见 \(P+\Sigma_t+O_t\)、并把推理投影进可合并状态后立刻丢掉。
