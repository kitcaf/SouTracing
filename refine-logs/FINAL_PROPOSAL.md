# Final Proposal

## Problem Anchor

在不暴露传播树根、转发方向、精确时间戳和事件后信息的条件下，给定事件发生前的环境图、观察时刻感染状态、因果可用的用户历史特征以及全局消息内容，定位平台内传播发起者。

## Working Title

**OpenTrace: Leakage-Controlled Real-World Source Localization with Causal Context**

## Core Thesis

现有传播源定位的主要障碍不只是模型能力，而是训练/评价任务与真实部署之间存在三重错配：

1. 模拟传播动力学与真实传播不一致；
2. 真实级联转换容易泄漏源点；
3. 富属性特征缺少严格的时间可用性约束。

OpenTrace 通过 causal cutoff、source-disjoint split 和 leakage probes 建立真实快照基准；CausalHazard-SL 学习消息—用户—边条件下的前向传播 hazard，再通过 forward consistency 反演源点。

## Contributions

1. 首个面向公开纵向社交数据的 leakage-controlled、causally sliced 源定位基准；
2. 统一的四级输入协议：Topo、Causal-Profile、Causal-Content、Partial/Open；
3. 对 sim2real gap、身份记忆和未来信息泄漏的系统审计；
4. CausalHazard-SL：learn-forward-then-invert 的特征条件方法；
5. 时间、source-disjoint 和可选跨平台泛化评价。

## Dataset Plan

### Primary

- Bluesky Social Dataset / longitudinal Bluesky data；
- 以 original post + repost/quote records 构造真实事件；
- 使用事件前 follow graph 和 historical interaction graph。

### Secondary

- FediData/Mastodon：仅在 boost 链、时间和账号映射完整时纳入；
- PHEME/UPFD：只做兼容性和 leakage 对照，不作为主 benchmark。

## Method

### Encoder

- 用户画像与历史行为编码；
- 历史帖子语义编码；
- 全局消息编码；
- follow/interaction multiplex graph 编码。

### Forward Hazard

\[
\lambda_{uv}^{(c)}
=\mathrm{softplus}(f_\theta(z_u,z_v,e_{uv},m_c)).
\]

### Inverse Inference

逆向网络先产生 top-\(K\) 源候选，再使用 learned forward process 重构观察快照并重排序。

### Training Objective

\[
\mathcal{L}=\mathcal{L}_{src}
+\alpha\mathcal{L}_{forward}
+\beta\mathcal{L}_{cycle}
+\gamma\mathcal{L}_{calib}.
\]

## Falsifiable Hypotheses

- H1：同一环境图上，模拟级联训练到真实级联测试产生显著 sim2real 性能下降；
- H2：严格 source-disjoint 后，用户 ID/重复源先验的优势显著消失；
- H3：pre-event profile/history/content 仍能相对 topology-only 提升至少 5% MRR；
- H4：learned hazard 优于固定 IC/LT，并能解释一部分 sim2real gap；
- H5：forward-consistency 提升跨时间和部分观测鲁棒性。

## Kill Criteria

- 无法为至少 80% 合格级联构建事件前环境图；
- 删除转发树/时间字段后，任务几乎无法学习；
- degree/ID-only 已接近饱和，说明数据存在结构或身份泄漏；
- feature gain 在 source-disjoint 下消失；
- learned hazard 不优于简单 topology GNN；
- 所谓真实源只能由平台内部引用字段定义，无法形成合理的受限观测应用场景。

## Recommended Scope

先把 benchmark/protocol 做扎实。方法保持简单、可解释；不要把 LLM 作为主模块。只有 Pilot 0–2 通过后才扩展到跨平台和 open-world。

