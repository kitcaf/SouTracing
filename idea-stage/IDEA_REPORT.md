# Idea Discovery Report

**方向**：网络传播图上的传播源定位（source localization / rumor source detection），重点研究如何将带用户、内容和时间信息的真实传播数据转化为严格、非平凡、可复现的源定位任务  
**日期**：2026-08-21  
**Run ID**：`source-tracing-datasets-2026-08-21`  
**推荐主题**：**OpenTrace：面向真实传播快照的无泄漏基准与特征条件传播反演**

## Executive Summary

你的观察抓住了一个真实问题：经典传播源定位长期依赖“真实拓扑 + IC/LT/SIR 模拟扩散”或只包含感染状态的旧级联，用户画像、内容和真实传播动力学被大量丢弃。

但截至 2026 年，不能再把论文贡献表述为“首次把富属性传播数据用于源定位”。最接近的工作已经包括：

- **NFSL（CIKM 2024）**：面向真实传播场景的 user-centric source localization；
- **DSLF（IJCAI 2024）**：利用用户画像和传播特征进行跨平台联合源定位；
- **CRSLL（IJCAI 2025）**：结合用户画像、评论信息与 LLM 的跨平台源定位；
- **TSR-RSD（2026）**：在 GossipCop、PolitiFact、PHEME 等富内容数据上使用 GNN 与语义推理；
- **PDSL（arXiv 2026）**：进一步强调真实传播动力学和跨网络泛化。

因此，**“数据更丰富 + 拼一个 feature-aware GNN”本身创新性不足**。真正尚未被解决、且更容易形成高质量论文的缺口是：

1. 富属性数据转换后经常泄漏源点：传播树根、转发方向、精确时间戳、source-only 文本或记录类型会让任务退化为“找 root”；
2. 用户特征常混入事件发生后的数据，产生 future leakage；
3. 随机级联划分允许同一 source user 同时出现在训练和测试中，模型可能只是在记忆“惯常源点”；
4. PHEME/UPFD 等通常提供传播树而非事件发生前的完整社会/交互图，和标准 source-localization 输入并不一致；
5. 目前缺少一个完全开放、时间切片严格、同时提供真实级联、环境图、内容和历史用户特征的统一基准；
6. 现有方法多直接做判别式分类，较少显式学习“消息—用户—边”条件下的真实传播核，再解决逆问题。

**最终建议**：以 2024–2025 年公开的 Bluesky 全量/纵向数据为主，构建 `OpenTrace-BSky`；定义严格的 causal cutoff、source-disjoint split、泄漏探针和 sim2real 协议；提出 **CausalHazard-SL**，学习真实的 feature-conditioned propagation hazard，并用 forward-consistency 约束反演源点。

---

## 1. 问题定义与需要澄清的“源”

### 1.1 建议采用的任务

对一条真实传播事件 \(c\)，给定：

- 事件发生前构建的环境图 \(G_{<t_0}=(V,E)\)；
- 观察时刻 \(T\) 的传播状态 \(y_T\in\{0,1\}^{|V|}\)；
- 事件发生前可获得的用户特征 \(x_v^{<t_0}\)；
- 全局消息内容 \(m_c\)，它对所有候选节点相同；
- 可选的粗粒度观察时间或缺失掩码；

预测平台内发起该传播事件的用户：

\[
\hat{s}=\arg\max_{v\in C_c}p_\theta(s=v\mid G_{<t_0},y_T,x^{<t_0},m_c).
\]

候选集 \(C_c\) 默认是被观察为感染/参与传播的用户。开放世界设置允许真实源不在已观察集合中。

### 1.2 必须收窄的语义

数据能可靠标注的通常是：

> **platform-level diffusion initiator**：平台内这条原始记录的作者。

它不一定是：

> **ultimate information origin**：该观点、新闻或谣言在现实世界中的最初来源。

论文必须使用前一种表述，避免把“原帖作者定位”夸大为“事实源头追踪”。

---

## Literature Landscape

### 2.1 经典方法与当前基准范式

| 年份 | 工作 | 主要贡献 | 数据/设置的局限 |
|---|---|---|---|
| 2019 | GCNSI | 使用 GCN 从感染快照识别源点 | 主要依赖模拟扩散 |
| 2022 | SL-VAE | 将源定位建模为图扩散逆问题 | 真实拓扑上模拟级联 |
| 2022 | IVGD | 可逆图扩散，学习前向与逆向映射 | 仍以预设扩散模型为主 |
| 2023 | DDMSL | 离散去噪扩散，恢复传播路径和源点 | 常用真实图拓扑加模拟状态 |
| 2024 | GIN-SD | 不完整节点观测下的源检测 | “用户状态”较弱，传播仍主要模拟 |
| 2024 | GraphSL | 统一实现多种源定位方法和数据接口 | 主要覆盖 Karate、Dolphins、Jazz、Cora-ML、Power Grid 等经典图及 IC/LT 模拟 |
| 2025 | SIDSL | 结构先验与迁移，缓解训练传播不足 | 未形成开放富属性真实基准 |
| 2025 | GDFSL | 学习更一般的传播动力学 | 未系统使用消息内容和因果历史用户特征 |
| 2026 | PDSL | 利用传播动力学与跨网络知识 | 表明“真实动力学 + 泛化”已成为竞争方向 |

代表性来源：

- [GraphSL: A Graph Source Localization Library](https://arxiv.org/abs/2405.03724)
- [DDMSL, NeurIPS 2023](https://openreview.net/forum?id=5Fr8Nwi5KF)
- [GIN-SD](https://arxiv.org/abs/2403.00014)
- [SIDSL](https://arxiv.org/abs/2502.17928)
- [GDFSL, IJCAI 2025](https://www.ijcai.org/proceedings/2025/325)
- [PDSL](https://arxiv.org/abs/2605.03550)

### 2.2 与“富属性真实传播”最接近的工作

| 工作 | 已经覆盖的内容 | 对本项目的含义 |
|---|---|---|
| [NFSL, CIKM 2024](https://dl.acm.org/doi/10.1145/3627673.3679704) | 真实传播场景、用户中心建模、动态异质传播行为 | “首次面向真实传播/用户行为”不能再宣称 |
| [DSLF, IJCAI 2024](https://www.ijcai.org/proceedings/2024/268) | Twitter–Weibo 跨平台联合定位，使用用户画像和传播特征 | “首次加入用户画像”不成立 |
| [CRSLL, IJCAI 2025](https://www.ijcai.org/proceedings/2025/348) | 用户画像、评论、LLM、跨平台语义对齐 | 普通 text/profile fusion 明显拥挤 |
| [TSR-RSD, 2026](https://www.mdpi.com/2079-9292/15/5/914) | GNN、结构过滤、LLM 语义推理；GossipCop/PolitiFact/PHEME | 使用富内容假新闻数据做 source detection 已出现 |

**结论**：原始 idea 有问题意识，但需要把贡献从“使用更多特征”升级为：

> **严格定义什么特征在观察时真正可用，消除转换泄漏，并学习真实的条件传播动力学。**

---

## Novelty Verification

| 候选贡献 | 查新结论 | 可保留的差异化 |
|---|---|---|
| 首次用真实传播级联做源定位 | 不成立 | 改为首次系统实施 causal cutoff 与 leakage audit |
| 首次加入用户画像 | 不成立，DSLF 已覆盖 | 强调事件前特征、source-disjoint 和身份泄漏探针 |
| 首次加入评论/文本/LLM | 不成立，CRSLL 与 TSR-RSD 已覆盖 | 全局内容共享、历史写作/兴趣匹配和 forward likelihood |
| 首次做跨平台源定位 | 不成立，DSLF/CRSLL 已覆盖 | 改为无配对、跨时间或 source-disjoint 泛化 |
| 首次学习传播动力学 | 不成立，GDFSL/PDSL 已接近 | 学习 message-user-edge 条件 hazard，并用于 inverse consistency |
| 公开统一真实快照 benchmark | 尚未发现等价工作 | 这是最有希望的核心贡献，但必须验证 NFSL 数据发布与任务细节 |

综合结论：**OpenTrace 的新颖性来自任务和评价协议，而不是“数据里有更多字段”**。CausalHazard-SL 的新颖性来自条件前向传播核和反演一致性，而不是普通 feature fusion。

---

## 3. 数据集转换审计

### 3.1 候选数据集

| 数据源 | 内容/用户特征 | 传播关系 | 环境图 | 源标签 | 转换结论 |
|---|---:|---:|---:|---:|---|
| PHEME | 帖子文本、用户、时间 | 回复会话树 | 通常无完整社会图 | 根推文 | 可做辅助集；若暴露树结构则源点平凡 |
| FakeNewsNet | 新闻、推文、用户画像、社交上下文 | 回复/转发等 | 设计上包含 social network，但复现依赖 hydration | 原始新闻/推文可追踪 | 可用但数据缺失和 API 漂移严重 |
| UPFD | 用户偏好/画像特征、传播图 | PyG 中的传播树/图 | 不是标准全体社会图 | 根节点可恢复 | 适合快速 pilot，不适合单独支撑新 benchmark |
| MuMiN | 多语言、多模态、用户/帖子/声明异构图 | 社交互动 | 有较丰富异构关系 | 需重新定义事件级源 | 适合扩展，但标签映射复杂 |
| Bluesky Social Dataset | 帖子、用户、关注、回复、引用、转发、时间 | 可恢复真实 repost/quote 事件 | 有时间化的 follow/interaction 数据 | 原帖作者 | **主推荐** |
| BlueTempNet | 纵向互动与网络动态 | 时间化社交互动 | 支持 pre-event graph | 可结合原帖/转发记录 | **推荐用于时间切片和分布漂移** |
| FediData / Mastodon | 公开帖子、回复、boost、账号信息 | 联邦平台互动 | 可构建实例/用户交互图 | boost 原帖可作为平台源 | 作为第二平台，需先核验数据完整性 |

数据来源：

- [PHEME dataset](https://figshare.com/articles/dataset/PHEME_dataset_for_Rumour_Detection_and_Veracity_Classification/6392078)
- [FakeNewsNet](https://arxiv.org/abs/1809.01286)
- [UPFD repository](https://github.com/safe-graph/GNN-FakeNews)
- [MuMiN](https://github.com/MuMiN-dataset/mumin)
- [Bluesky Social Dataset, ICWSM 2025](https://ojs.aaai.org/index.php/ICWSM/article/view/35800)
- [A longitudinal dataset of social interactions on Bluesky](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0327407)
- [BlueTempNet](https://arxiv.org/abs/2507.07478)
- [FediData](https://github.com/FDUDataNET/FediData)

### 3.2 为什么“直接转换 PHEME/UPFD”不够

1. **Root leakage**：传播树的根节点就是源点；
2. **方向泄漏**：转发/回复边的方向可直接定位零入度节点；
3. **时间泄漏**：最早时间戳直接给出答案；
4. **记录类型泄漏**：original post 与 repost 的字段不同；
5. **source-only content leakage**：只有源节点持有完整原文，其他节点为空；
6. **图定义不匹配**：传播树不是“全体社会关系图”，无法与经典任务公平比较；
7. **未来信息泄漏**：当前用户画像、粉丝数、历史帖子可能包含事件发生后的状态；
8. **身份记忆**：随机划分使同一用户在训练/测试中反复作为源点。

这组问题本身可以形成论文的重要贡献：**Real-Cascade Source Localization Leakage Taxonomy**。

---

## 4. 推荐的 OpenTrace Benchmark

### 4.1 数据实例构造

以 Bluesky 原帖及其 repost/quote 为一个事件：

1. 原帖作者为平台内源点 \(s\)，原帖时间为 \(t_0\)；
2. 在观察截止时间 \(T\) 前转发/引用该记录的用户构成感染集合 \(I_T\)；
3. 环境图只能用 \(t_0\) 之前的数据构造：
   - `follow graph`：事件前已存在的关注边；
   - `interaction graph`：过去 \(H\) 天的 reply/quote/repost 互动；
   - `multiplex graph`：同时保留两类边；
4. 为控制规模，取感染节点周围的 \(k\)-hop 子图，并加入度匹配的未感染节点；
5. 删除当前事件的转发父边、精确动作时间、record type、URI 等可直接暴露答案的字段；
6. 用户文本历史和画像严格截断到 \(t_0\)；
7. 全局消息文本 \(m_c\) 对所有候选源共享，不将其只挂在源节点上。

### 4.2 四种任务协议

| 协议 | 输入 | 回答的问题 |
|---|---|---|
| `Topo` | 环境图 + 最终感染状态 | 与经典 source localization 公平兼容 |
| `Causal-Profile` | `Topo` + 事件前画像/历史行为 | 用户信息是否真正增加可识别性 |
| `Causal-Content` | 上述输入 + 全局消息及用户历史语义 | 消息—用户匹配是否帮助定位 |
| `Partial/Open` | 节点缺失、源可能未被观察 | 更接近部署场景 |

### 4.3 必须提供的划分

- **Chronological split**：训练早期事件，测试未来事件；
- **Source-disjoint split**：测试源用户在训练中从未作为源；
- **Content-family-disjoint split**：同 URL、同原帖或近重复内容不能跨集合；
- **Platform/domain split**：若加入 Mastodon，则训练 Bluesky、测试 Mastodon；
- **Transductive split**：保留重复用户，作为真实运营场景，但必须单独报告。

### 4.4 泄漏探针

每个数据版本发布前运行：

- timestamp-only；
- node-degree-only；
- record-type-only；
- user-ID embedding-only；
- source propensity count；
- text availability mask-only；
- post-event profile-only；
- shuffled-label canary。

若任一非语义泄漏探针获得异常高的 Hit@1，必须停止方法比较并修正数据。

---

## Ranked Ideas

| 排名 | Idea | 新颖性 | 可行性 | 主要风险 | 决策 |
|---:|---|---:|---:|---|---|
| 1 | **OpenTrace：无泄漏、因果时间切片的真实源定位基准 + sim2real 审计** | 高 | 高 | 数据清洗与快照成本 | **推荐，主贡献** |
| 2 | **CausalHazard-SL：学习消息/用户/边条件传播核，再进行源反演** | 中高 | 中高 | 前向 hazard 学不好 | **推荐，方法贡献** |
| 3 | **Partial/Open-World SL：源缺失与不完整观察** | 中高 | 中 | 标签与评价复杂 | 推荐作为扩展 |
| 4 | **跨时间/跨平台 domain generalization** | 中 | 中 | 第二平台数据质量 | 作为泛化实验 |
| 5 | **信息层边际可识别性诊断** | 中 | 高 | 单独成文偏弱 | 并入主论文 |
| 6 | **重复传播者先验** | 中低 | 高 | 容易退化为身份泄漏 | 仅作 transductive baseline |
| 7 | **内容变异时钟** | 中 | 中低 | repost 文本变异信号可能很弱 | 小 pilot 后决定 |
| 8 | **LLM persona cascade simulator** | 中 | 低 | 模拟真实性不可证 | 暂缓 |
| 9 | **普通 text+profile GNN** | 低 | 高 | NFSL/DSLF/CRSLL/TSR-RSD 已拥挤 | **淘汰为主 idea** |
| 10 | **直接把 PHEME/UPFD 根节点当源做 benchmark** | 低 | 高 | root/time/structure leakage | **淘汰** |

### 推荐组合

不是三篇松散工作，而是一个明确论文故事：

> **OpenTrace exposes how simulated and artifact-prone source localization differs from causal real-world localization; CausalHazard-SL closes part of this gap by learning the real, context-dependent forward process before inversion.**

---

## 6. 推荐方法：CausalHazard-SL

### 6.1 核心思想

对消息 \(m\)、传播者 \(u\)、接收者 \(v\) 和事件前边特征 \(e_{uv}\)，学习传播 hazard：

\[
\lambda_{uv}^{(c)}
=\operatorname{softplus}
\left(f_\theta(z_u^{<t_0},z_v^{<t_0},e_{uv}^{<t_0},m_c)\right).
\]

其中 \(z_u^{<t_0}\) 只来自事件发生前的画像、行为和文本历史。

对每个候选源 \(s\)，前向模块预测观察时刻的感染概率：

\[
\hat{y}_T^{(s)}=\mathcal{F}_{\theta}
(G_{<t_0},s,m_c,z^{<t_0},T).
\]

然后根据观测状态计算源后验：

\[
p(s\mid y_T,G,x,m)
\propto p(s)\,
\exp\{-D(y_T,\hat{y}_T^{(s)})\}.
\]

为避免逐候选完整模拟，使用两阶段推断：

1. 逆向 GNN/Graph Transformer 生成 top-\(K\) 候选；
2. 用 learned forward hazard 对 top-\(K\) 做 consistency reranking。

### 6.2 损失函数

\[
\mathcal{L}
=\mathcal{L}_{src}
+\alpha\mathcal{L}_{forward}
+\beta\mathcal{L}_{cycle}
+\gamma\mathcal{L}_{calib}.
\]

- `source loss`：真实源的排序/分类损失；
- `forward loss`：真实传播边或时间分桶的 likelihood；
- `cycle loss`：预测源经过前向扩散后应重构观测快照；
- `calibration loss`：避免在开放世界设置中过度自信。

### 6.3 与近邻工作的区分

- 对比 NFSL：重点不是用户行为编码，而是**因果特征政策 + 真实 forward kernel + inverse consistency**；
- 对比 DSLF/CRSLL：不是跨平台 feature fusion/LLM semantic alignment，而是严格快照任务和传播动力学反演；
- 对比 GDFSL/PDSL：条件传播核显式依赖消息内容、用户历史和边上下文，并在开放真实数据上验证；
- 对比 TSR-RSD：不是直接让 LLM 判断源，而是用可审计的前向传播 likelihood 约束源后验。

---

## 7. 最小可行性实验（MVE）

### Pilot 0：数据可构造性（CPU，2–4 小时）

抽取 1,000 个 Bluesky repost cascades，检查：

- 原帖作者覆盖率；
- 事件前 follow/interaction graph 的节点覆盖率；
- 级联规模、深度、候选数分布；
- 删除泄漏字段后是否仍能构成非空任务。

**继续门槛**：至少 80% 的合格级联拥有可用的 pre-event 环境图；中位候选数不低于 10。

### Pilot 1：任务是否非平凡（CPU/单 GPU，<2 小时）

运行：

- random；
- degree；
- closeness / Jordan center；
- earliest-time（只作为 oracle，不属于合法输入）；
- topology-only GCN；
- user-ID-only。

**继续门槛**：

- degree/ID-only 不能异常接近监督模型；
- topology-only 应显著高于 chance，但远未饱和；
- source-disjoint 后性能应合理下降，而不是归零。

### Pilot 2：因果特征是否有信号（单 GPU，<2 小时）

比较：

- topology-only；
- + pre-event profile；
- + pre-event history；
- + global content；
- + post-event features（泄漏上界，不作为正式模型）。

**继续门槛**：source-disjoint MRR 至少提升 5% 相对值，且提升不能主要来自用户 ID 或 post-event 特征。

若 Pilot 2 失败，保留 benchmark/sim2real 论文，放弃 feature-aware 方法主张。

---

## 8. 完整实验设计

### 8.1 Baselines

- Rumor/Jordan center、distance centrality；
- GCNSI；
- SL-VAE；
- IVGD；
- DDMSL；
- GIN-SD；
- SIDSL / GDFSL / PDSL（代码可获得时）；
- GraphSL 中的统一实现；
- feature-only MLP；
- topology GNN；
- source propensity；
- NFSL/DSLF 风格适配模型。

### 8.2 Metrics

- Hit@1、Hit@5；
- MRR；
- graph-distance error；
- NLL / Brier score；
- ECE；
- inference latency；
- open-world AUROC / AUPRC。

### 8.3 主实验

1. **Benchmark validity**：泄漏探针、任务难度和跨划分稳定性；
2. **Sim2real gap**：在同一环境图上用 IC/LT/SIR 模拟训练，再测试真实级联；
3. **Feature value**：profile/history/content 的边际贡献；
4. **Method comparison**：CausalHazard-SL 对 topology-only 和直接 fusion；
5. **Generalization**：chronological、source-disjoint、跨平台；
6. **Robustness**：10%–70% 节点缺失、边缺失、源不可见；
7. **Calibration**：未知源拒识和置信度。

### 8.4 关键消融

- learned hazard → 固定 IC；
- 删除 global content；
- 删除历史文本；
- 删除 profile；
- 删除 forward loss；
- 删除 cycle consistency；
- 不做 top-\(K\) reranking；
- follow graph vs interaction graph vs multiplex；
- transductive vs source-disjoint。

---

## External Critical Review

### 9.1 最强审稿意见

1. **“你定位的只是原帖作者，不是谣言真正源头。”**  
   回应：严格改名为 platform-level diffusion initiator localization；开放世界扩展讨论外部注入。

2. **“Bluesky 的 repost 记录本来就指向原帖，你人为删掉这个字段后制造了一个任务。”**  
   回应：这是为研究受限观测下的源定位；必须证明与经典输入定义一致，并提供不同信息可见度协议，不能宣称平台本身无法查原帖。

3. **“富特征提升只是记住高频 source users。”**  
   回应：source-disjoint、time split、ID-only probe、history truncation 是硬门槛。

4. **“完整 follower snapshot 并不可靠，删帖/取消关注会造成历史图误差。”**  
   回应：使用 timestamped follow records 和 pre-event interaction graph 双轨构图；报告覆盖率与图不确定性。

5. **“只做一个平台，结论不一般。”**  
   回应：第一版至少加入一个跨时间测试；若 FediData boost 链完整，再加入 Mastodon 外部测试。

6. **“benchmark + method 两个贡献都做不深。”**  
   回应：主贡献顺序必须固定为 benchmark/protocol 第一，方法第二；若数据 pilot 不稳，不继续复杂方法。

### 9.2 审稿评分预估

- 原始 “PHEME/UPFD + profile/text GNN”：**4/10，Reject**；
- OpenTrace benchmark，但无强实证：**6/10，Borderline**；
- OpenTrace + 泄漏审计 + 明确 sim2real gap：**7/10，Weak Accept**；
- 再加 source-disjoint 下有效的 CausalHazard-SL 和第二平台/时间泛化：**8/10，Accept 潜力**。

### 9.3 Reviewer Verdict

`TRIAGE_VERDICT: TOP_PICKS=[OpenTrace leakage-controlled benchmark, CausalHazard-SL, Partial/Open-World extension]`

---

## 10. Eliminated Ideas

### 普通富特征 GNN

淘汰作为主 idea。原因：NFSL、DSLF、CRSLL 和 TSR-RSD 已覆盖真实行为、画像、评论和语义推理。它只能作为 baseline。

### 直接转换 PHEME/UPFD

淘汰作为 benchmark 主体。原因：传播树根和时间结构极易泄漏，且缺少标准事件前环境图。

### 重复传播者先验

不作为主贡献。它在 transductive 场景可能有效，但在 source-disjoint 下失效，并可能被审稿人视作用户身份记忆。

### LLM 级联模拟

暂缓。真实度难验证，预算较高，而且会分散主线。

---

## 11. 推荐投稿定位

### 最优论文叙事

**题目草案**：

> **OpenTrace: Leakage-Controlled Real-World Source Localization with Causal Context**

或：

> **Learning the Forward Process to Solve the Inverse: Real-World Source Localization on Temporal Social Graphs**

### 适合的 venue

- KDD Datasets & Benchmarks / Research Track；
- WWW；
- CIKM；
- ICWSM（数据与社会计算叙事更强）；
- NeurIPS Datasets & Benchmarks（需要更强的数据治理、复现和广泛影响）。

---

## 12. 下一步

1. 先完成 Bluesky 数据字段和许可审计；
2. 实现 1,000 级联的 Pilot 0；
3. 运行 leakage probes 和 source-disjoint Pilot 1；
4. 只有任务非平凡时，才实现 CausalHazard-SL；
5. Pilot 2 若没有稳定特征增益，论文收缩为 benchmark + sim2real audit；
6. 不应先在 PHEME/UPFD 上投入复杂模型。

## Evidence Gate Status

- 本仓库缺少 `run_state.py` 与 `idea_discovery_gate.py` 的规范解析路径，因此自动 evidence gate 仍为：
  `BLOCKED: run_state.py/idea_discovery_gate.py helpers unavailable in resolution chain`。
- 本报告已完成文献查新、idea 筛选、竞争工作审计、外部批判式评审和实验方案设计，但**尚未运行真实数据 pilot**。
- 因此当前状态应表述为：**研究提案已收敛，经验可行性待 Pilot 0–2 验证**，而不是“idea 已被实验证实”。
