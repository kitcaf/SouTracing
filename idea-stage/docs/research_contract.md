# Research Contract

## ACTIVE Idea

**OpenTrace：面向真实传播快照的无泄漏基准与 CausalHazard-SL**

## Selection Rationale

“富属性数据 + GNN”已被 NFSL、DSLF、CRSLL 和 TSR-RSD 部分覆盖；仍具有明确空缺的是因果时间切片、泄漏控制、source-disjoint 泛化、sim2real 审计，以及基于真实条件传播核的逆问题求解。

## Core Claims

1. 现有真实级联转换协议可能通过根、方向、时间、身份和未来特征泄漏源点；
2. 在相同环境图上，模拟训练到真实测试存在可量化的 sim2real gap；
3. 事件前用户历史与消息上下文在 source-disjoint 设置下仍提供增益；
4. 学习 feature-conditioned forward hazard 并施加 cycle consistency，比直接 feature fusion 更可靠；
5. OpenTrace 可复现地支持经典 topology-only 与富属性方法的统一比较。

## Minimum Convincing Evidence

- ≥1,000 个通过泄漏审计的真实级联；
- chronological + source-disjoint 划分；
- degree/ID/timestamp/record-type probes；
- 至少 5 个经典/神经 baseline；
- 明确 sim2real gap；
- causal features 在 source-disjoint 下 ≥5% 相对 MRR 增益；
- learned hazard 或 forward consistency 至少一项有稳定增益。

## Kill Conditions

- 环境图覆盖率不足；
- 任务只能依靠泄漏字段完成；
- 特征增益完全来自用户身份记忆；
- learned hazard 不优于简单方法；
- 无法清晰区分“平台内发起者”和“现实世界最初来源”。

## Next Step

执行 `refine-logs/EXPERIMENT_PLAN.md` 的 P0–P3。

