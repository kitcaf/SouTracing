# Experiment Plan

## Phase A — Data Feasibility

### A1. Field audit

- 检查 Bluesky 数据中的 post、repost、quote、follow、reply、timestamp、profile 字段；
- 检查原帖作者与参与用户覆盖；
- 检查 follow 边是否可按事件时间截断；
- 输出数据字典和隐私/许可说明。

### A2. Cascade extraction

- 过滤 bot、自转发、重复记录；
- 级联最小规模：10；
- 至少生成 1,000 个 pilot cascades；
- 构建 follow、interaction、multiplex 三种环境图。

### A3. Leakage audit

- 删除 parent/repost reference、record type、精确时间；
- 运行 timestamp、degree、ID、availability-mask 和 post-event probes；
- 人工抽查 100 个样本。

**Gate A**：环境图覆盖率 ≥80%，且无单一泄漏 probe 饱和。

## Phase B — Baseline Non-triviality

运行：

1. Random；
2. Degree；
3. Jordan/rumor center；
4. Closeness；
5. Source propensity；
6. GCN infection-state baseline；
7. user-ID-only；
8. exact-time oracle。

划分：

- random cascade；
- chronological；
- source-disjoint。

**Gate B**：合法 topology baseline 高于 chance，但 Hit@1 不应接近饱和；ID-only 在 source-disjoint 下应接近失效。

## Phase C — Feature Signal

配置：

- `Topo`；
- `Topo + Profile(<t0)`；
- `Topo + History(<t0)`；
- `Topo + Global Content`；
- `All Causal Features`；
- `Post-event Features`（泄漏上界）。

**Gate C**：`All Causal Features` 在 source-disjoint MRR 上相对 `Topo` 提升 ≥5%，至少两个时间切片稳定。

## Phase D — CausalHazard-SL

### D1. Forward model

- edge-level repost likelihood；
- time-bin prediction；
- fixed IC/LT 对照；
- calibration。

### D2. Inverse model

- top-\(K\) candidate generator；
- forward-consistency reranker；
- joint training。

### D3. Ablations

- no hazard；
- no cycle；
- no content；
- no history；
- no profile；
- no reranking；
- different graph modalities。

## Phase E — Main Claims

### Claim 1: sim2real gap

同一真实环境图上：

- IC/LT/SIR 生成训练级联；
- 真实级联测试；
- real-trained upper bound。

### Claim 2: causal feature value

比较因果特征与 post-event 泄漏特征，量化每一层的边际收益。

### Claim 3: method effectiveness

比较 classical、GraphSL、recent neural methods、direct fusion 和 CausalHazard-SL。

### Claim 4: generalization

- future time；
- unseen sources；
- optional cross-platform。

### Claim 5: robustness

- missing nodes；
- missing edges；
- source unobserved；
- different observation fractions。

## Metrics

- Hit@1 / Hit@5；
- MRR；
- graph-distance error；
- NLL / Brier；
- ECE；
- AUROC/AUPRC for unknown-source；
- training/inference cost。

## First Three Runs

1. `P0_extract_1k_cascades`；
2. `P1_leakage_and_centrality_baselines`；
3. `P2_topology_vs_causal_features_source_disjoint`。

## Compute Budget

- Phase A/B：CPU；
- Pilot feature model：单卡 1–2 小时；
- 完整基准：按数据规模逐步扩展；
- 在 Gate C 前不进行大模型微调，只使用冻结文本 embedding。

