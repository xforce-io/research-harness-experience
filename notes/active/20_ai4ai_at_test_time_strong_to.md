---
zone: active
tags: [update_harness, evaluation_protocol]
pin: false
score: 0.8
dwell: 2
---
# AI4AI at Test-Time: Strong-to-Weak Capability Transfer via Harnesses

> Topic integration note derived from Library read artifact `.researcher-workspace/library/papers/paper_arxiv_2608_12307/reads/read_paper_arxiv_2608_12307.md`.

## Library read

---
kind: library-read-identity
authors: ["Cheng Qian","Wenting Zhao","Liangwei Yang","Heng Wang","Jielin Qiu","Heng Ji","Silvio Savarese","Huan Wang","Shelby Heinecke"]
source_id: "arxiv:2608.12307"
source_url: "https://arxiv.org/abs/2608.12307"
pdf_url: "https://arxiv.org/pdf/2608.12307"
---
> 不训练弱模型：强 builder 在 5% 验证集上迭代编写含 benchmark 路由、确定性求解器与格式强制的推理时 harness，把可编译的任务结构从弱模型的推理中卸载进代码，GPT-5.4-mini 宏平均准确率 0.488→0.763（最佳 0.912）。

## Essence

**问题**：强→弱能力迁移的主流路线（蒸馏、on-policy distillation）都要更新弱模型参数；"推理时 harness"这条不改权重的路线缺乏系统性实证——为什么有效、哪些设计真正起作用、何时失效，此前没有受控研究。

**做法**：把强 builder 模型放进 agentic coding 平台（Cursor / Claude Code / GPT Codex），初始工作区只有三样东西：任务规则文件、目标模型调用示例、5% 带标签验证集。builder 循环执行"写/改 scaffold → 让弱目标模型在验证集上跑分 → 把错误样本回灌工作区"，收敛后导出一个可执行 entry point，由人工在 3900 条隐藏测试集上运行。scaffold 形态完全自由：路由、提示模板、确定性求解器、格式强制、few-shot 任意组合。

**证据**：72 次运行中 GPT-5.4-mini 从 0.488 升至均值 0.763（100% 运行超基线，最佳 0.912）；收益与 determinism fraction（完全由代码/规则作答的题目占比）相关 r=0.72，而与更长推理、更多验证采样基本无关。

**边界**：结论限于 4 个 ToM 文本选择题基准；收益随基准"可编译性"变化极大（BigToM 近乎全卸载，MuMA-ToM 仍主要靠模型），且 benchmark 路由依赖"输入属于哪个基准已知"这一评测前提。

## Claims

**机制类**

1. 测试时能力迁移成立：builder 全程不可见测试集，+0.275 的均值提升表明 scaffold 捕获的是跨样本可复用的任务结构，而非实例级记忆 [§5.1]。
2. 收益机制是认知负荷卸载而非激发更多推理：determinism fraction 与最终准确率 Pearson r=0.72，而 scaffold 代码量与准确率仅 r≈0.22——关键是"卸载对的东西"而非"写更多代码" [§5.9, Table 4]。
3. 有效 scaffold 是两层结构：近乎普适的"可靠性地板"（format enforcement 100%、greedy decoding 98%、benchmark routing 95%）不解释 run 间差异；拉开差距的是任务结构编译——polarity/negation logic (+0.090)、structured extraction (+0.055)、hybrid fallback (+0.040) [§5.4 Fig 6; §5.8 Fig 9]。
4. 最佳 scaffold 的 item 级改善高度不对称：修复 1717 个基线错误、仅破坏 105 个基线正确项（配对 McNemar χ²=1424.4，p<10⁻⁴）[§5.8, Fig 10a]。
5. Headroom 定律：realized uplift 由目标模型在该基准上的 headroom（1−baseline）预测，r=0.75——scaffolding 本质是"能力恢复机制"，帮助最多的地方是可纠正错误最多的地方 [§5.6, Fig 8b]。
6. 对已强的目标，scaffolding 会反噬：Gemini-3.5-flash 上每个 builder 都在至少一个基准回退（9/20 情形），集中于其已近饱和的 Hi-ToM（−0.04）与 MuMA-ToM（−0.02）[§5.6]。
7. 弱模型自我 scaffolding 也能 +0.17~0.22，但只有更强 builder 才能进入高性能区间（GPT Codex 上强 builder +0.314 vs 自建 +0.168）[§5.8, Fig 10b]。

**消融/规律类**

8. Builder reasoning effort 是单调杠杆：Opus-4.7 从 low 到 extra-high，0.711→0.793→0.807→0.856（Spearman ρ=0.77；x-high vs high p=0.013），无过度工程迹象；scaffold 代码量随之从 ~510–650 LOC 增至 ~1000–1300 LOC [§5.7, Table 3]。
9. 平台效应是二阶条件因子：原生平台平均优势仅 +0.013（8 格中 5 胜，p=0.484）；平台×effort 交互更实质——原生平台优势只在高 reasoning effort 下出现 [§5.5, Fig 7]。
10. 验证高效：builder 中位数只用 5 次验证评估（范围 2–15）；最佳验证分与全集分 r=0.96、平均乐观 gap 仅 0.021；但迭代次数与最终成绩 r=0.17——限制因素是 builder 假设质量而非反馈量 [§5.3, Fig 5]。
11. 可复现但不完全确定：组内 SD 均值 0.036（约为主提升的 1/10），最宽 repeat 跨度 0.201；大方差集中于确定性求解器策略——单条规则逻辑错误可在 1000+ 题基准上移位数十点 [§5.2, Fig 4]。
12. 不同强 scaffold 互补：顶部 scaffold 修复集的并集覆盖 97% 的基线错误，超过任何单一 scaffold [§5.8, Fig 10c]。

**基准记分**

13. 主结果：57 个 GPT-5.4-mini 目标 run 均值 0.763（+0.275），100% 超基线；最佳单次 0.912（GPT-5.5 × GPT Codex，+0.423 / 86.7% 相对提升），在全部四个基准上超过无 scaffold 的 GPT-5.4（0.619）与 GPT-OSS-120B [§5.1, Table 1, Fig 3]。
14. 与人类设计 harness（UserHarness，0.939）相比任务依赖：BigToM 上 1.00 反超 0.95，但 MMToM-QA 0.84 vs 0.98、Hi-ToM 0.80 vs 0.87、MuMA-ToM 0.88 vs 0.96，差距集中在不可编译部分 [§5.1, Fig 3b]。
15. 残余错误标记可编译性边界：Hi-ToM 递归深度 0→4 准确率 0.999→0.700（deception 再降至 0.772）；MMToM qtype 2.1 低至 0.680；顶部 8 个 scaffold 平均修复 83% 基线错误、仅破坏 7% 基线正确项 [§5.10, Table 5, Fig 12]。

## Assumptions

1. 5% 固定随机种子验证切片对隐藏测试集有代表性；对程序化生成的基准（BigToM 源自因果模板），切片足以恢复生成模板 [§3, §4]。
2. 所有任务为文本格式选择题（binary / 3-choice），答案空间可被确定性解析——format enforcement 的收益预设了这一点 [§4]。
3. 输入的基准身份可被路由识别：聚合评测中 scaffold 按基准/子类型分流成立，混合部署场景未论证 [§5.4]。
4. 任务指令明确要求 builder "降低目标模型认知负荷"，确定性卸载行为部分是被指令诱导而非完全自发涌现 [§5.9]。
5. 12 技术分类法由作者阅读 scaffold 代码人工编码，未报告 inter-annotator 一致性 [§5.4]。
6. 目标模型 API 行为在构建与评测期间稳定，导出的 entry point 可直接执行无需适配 [§3, Algorithm 1]。

## Method

与传统路线的对比：

| | 训练时蒸馏 | 本文（测试时 scaffolding） |
|---|---|---|
| 改什么 | 弱模型参数 | 弱模型周围的推理环境 |
| 载体 | 权重更新 | 代码化 harness（路由/模板/求解器/格式约束） |
| 何时发生 | 训练期 | 推理期，目标模型冻结 |

形式化（Algorithm 1）：

- **输入**：builder 模型 M_build 置于 agentic coding 平台 H_build；初始工作区 W₀ = {规则文件 R, 目标模型调用示例 C_demo, 5% 验证集 V}；隐藏全集 T 对 builder 不可见。
- **循环**：(1) M_build 读取工作区 → (2) 提出或修订 scaffold S_k（任意推理时过程：prompt 模板、benchmark 路由、确定性前/后处理、答案格式强制、验证 pass、few-shot 检索、符号求解器）→ (3) 在 V 上调用目标模型执行 S_k，得验证准确率 a_k → (4) 错误集 E_k（预测≠标签的验证样本）并入 W_{k+1} → 重复直至 builder 提交。
- **输出**：可执行 entry point f_Ŝ(x; M_tar)，由人工评估者在 T 上运行并返回全集准确率。
- 概念上 builder 逼近 argmax_S Acc(S, M_tar; T)，但只能以 V 为代理，因此成功 scaffold 必须把验证切片中的结构泛化到隐藏集。

## Eval

- **数据**：4 个 ToM 基准聚合为 3900 条隐藏测试集——BigToM 1200（binary belief/goal/action）、Hi-ToM 1200（递归 0–4 阶嵌套信念 + 欺骗）、MMToM-QA 600（Bayesian goal/belief）、MuMA-ToM 900（3-choice 多智能体社交目标）；195 条 5% 验证集（固定随机种子）；仅用文本格式题目。
- **变量矩阵**：11 个 builder 配置（GPT-5.5、Opus-4.7 × 4 个 effort 档、Sonnet-4.6、GPT-5.4-mini、Codex-5.3、Gemini-3.1-Pro、Gemini-3.5-flash、Grok-0.1）× 3 个平台（Cursor / Claude Code / GPT Codex）× 2 个目标（GPT-5.4-mini 为主，Gemini-3.5-flash 做目标对照）× 3 次重复 = 72 runs。
- **基线**：vanilla 直调（GPT-5.4-mini 0.488、Gemini-3.5-flash 0.761）；UserHarness 人类设计 harness（0.939 / 0.941）；无 scaffold 的 GPT-5.4（0.619）与 GPT-OSS-120B 作跨模型参照。
- **指标**：4 基准全集准确率的非加权宏平均（主）；验证评估使用次数（次，效率）。统计工具：配对 McNemar 检验、permutation test、Pearson/Spearman 相关。

## Weaknesses

1. 摘要头条 "0.49→0.91" 用的是 72 次 run 中的最佳单次（best-of-72），均值是 0.763；且最佳 run 的提升显著依赖 BigToM 的模板编译（近乎免费的 observed/unobserved 表面线索），把最佳情形当作代表性结果表述 [Abstract vs §5.1 Table 1]。
2. Benchmark routing 是评测构造的产物：scaffold 按基准子类型分流的前提是知道输入来自哪个基准；真实混合流量中该信号不存在，"reusable skills" 的迁移声明未在此条件下测试 [§5.4 Fig 6b]。
3. 全部任务为选择题，format enforcement + greedy decoding 的"可靠性地板"收益在自由生成任务上未必成立，但结论以未加限定的 "scaffolding" 一般口吻陈述 [§4, §6]。
4. 技术归因自认 "associational rather than strictly causal" 且技术共现混杂，但结论小节仍以因果口吻断言 "the core mechanism is cognitive-load reduction"；表中方差极大或对照组仅 1–3 个 run 的条目（如 greedy −0.110、self-consistency +0.038）与充分对照条目并列，易被误读为效应量 [§5.8, §6, Fig 9]。
5. Headroom 定律与 over-scaffolding 结论仅基于 2 个目标模型；Gemini 侧 9/20 的回退也可能由格式干扰等混杂解释，样本量不足以支撑一般化原则 [§5.6]。
6. 无任何成本/延迟报告：高 effort 构建的一次性开销、scaffold（路由 + 确定性代码 + 目标调用）相对 vanilla 直调的推理时开销均未量化，"complement to distillation" 的实用性主张缺经济学维度 [§5.7, §6]。
7. Scaffold 代码与平台配置未发布，12 技术编码为主观人工判断且无一致性检验，72 runs 依赖特定平台版本，可复现性受限 [§4, §5.4]。
8. 组内仅 3 次重复，SD=0.036 的稳定性估计本身不确定；平台效应检验（8 个配对格、p=0.484）功效不足，"平台是二阶因素"可能是未检出而非证伪 [§5.2, §5.5]。
9. 验证-全集 r=0.96 部分是机械的：两者测同一程序化生成分布，模板化基准上 5% 切片天然足以恢复模板；该结论不能外推到异质真实数据 [§5.3]。

## Relations

- builds-on ADAS (Hu et al., 2025) [high]: §2 明确沿用 ADAS 的"agentic 系统即代码工件、由模型自动发现"框架，本文把该搜索约束到强 builder→弱目标的迁移 regime 并做受控变量分析。
- builds-on UserHarness (Qian et al., 2026) [high]: 直接用作 human-inspired 基线（0.939），且第一作者相同——本文相当于把其人工 ToM harness 的构建过程交给 builder 模型自动化。
- extends PAL / Program-of-Thoughts (Gao et al., 2023; Chen et al., 2022) [high]: §2 明言沿用"把脆弱认知工作移入可执行表示"的动机；确定性求解器作为高收益技术从 builder 设计中涌现，将该范式从单模型提示推广到跨模型 harness 编译。
- competes-with 训练时蒸馏 (Hsieh et al., 2023; Agarwal et al., 2024) [med]: 同样针对强→弱能力迁移，主张以推理时结构替代权重更新，但论文未做任何蒸馏对照实验（性能与成本均无 head-to-head），"complement" 定位缺乏实证支撑。
- extends weak-to-strong generalization (Burns et al., 2023) [med]: 方向反转（弱监督强模型 → 强模型为弱模型构建环境）且把迁移载体从训练信号移到推理环境；论文 §2 一句带过该对比但未实验。
- orthogonal Harness-Bench (Yao et al., 2026) / Meta-Harness (Lee et al., 2026) [med]: 三者同属 harness 工程文献，但后两者测同一模型的 harness 配置效应或搜索 harness 代码，本文隔离的是跨模型能力迁移量，测量对象不同。

## Takeaway

- 方法论可借鉴：**"验证错误回灌工作区 + 导出入口函数在隐藏集评估"** 是一个干净的 harness 构建评测协议；determinism fraction（代码完全作答的题目占比）是可移植的认知卸载量化指标。
- 需要复核：头条数字是 best-of-72 而非均值；收益与基准可编译性强耦合（BigToM 0.94 vs MuMA-ToM 0.36 的 determinism），选择题格式放大了 format enforcement 的收益；benchmark 路由依赖评测特有信号。
- 若只记一件事：scaffolding 不替代推理能力，而是重新分配它——builder 一次性把可编译结构写进 harness，弱模型只剩不可编译的残余；编译上限由任务的可编译性决定。
