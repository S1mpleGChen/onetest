# 🏥 Medical QA Model Evaluation Report (V2 - Mixed Dataset)

| Item | Details |
| :--- | :--- |
| **Date** | 2026-03-19 |
| **Evaluation Script** | `calculate_metrics.py` |
| **Data Source** | `baseline_analysis_report.json` (300 Samples) |
| **Model Version** | Baseline Model (Updated) |
| **Test Type** | Mixed Dataset (Should Answer + Should Abstain) |

---

## 1. 📊 Executive Summary

This evaluation comprehensively tested the model's performance on **300 mixed samples** (including both "Should Answer" and "Should Abstain" categories). Compared to the previous version which only contained "Should Answer" samples, this test introduced **80 "Should Abstain" samples**, exposing severe issues in the model's **hallucination control**.

*   **Overall Performance**: The model remains robust on "Should Answer" questions but exhibits **massive hallucinations** on "Should Abstain" questions.
*   **Key Strengths**: 
    *   **High Key Entity Accuracy (86.82%)**: When the model answers based on evidence, key information extraction is highly accurate.
    *   **Perfect Evidence Faithfulness (100%)**: On "Should Answer" samples, the model strictly adheres to the provided evidence.
*   **Critical Flaws**: 
    *   **Extremely High Hallucination Rate (26.67%)**: Out of the 80 samples that should have been rejected, the model almost universally chose to "force an answer," leading to a hallucination flag. This is the primary driver for the significant drop in overall scores.
*   **Clinical Usability Rate**: **62.33%**. The loss is primarily due to hallucinations in the "Should Abstain" category.

---

## 2. 📈 Core Metrics Breakdown

### 2.1 Data Distribution

| Category | Count | Percentage | Description |
| :--- | :--- | :--- | :--- |
| **Total Samples** | **300** | 100% | - |
| **Should Answer** | **220** | 73.3% | Sufficient evidence; model should provide a specific answer. |
| **Should Abstain** | **80** | 26.7% | Insufficient evidence or high risk; model should refuse to answer. |

### 2.2 Accuracy & Quality

| Metric | Value | Trend | Interpretation | Rating |
| :--- | :--- | :--- | :--- | :--- |
| **Base Accuracy (Binary)** | **80.33%** | 🔻 Down | % of samples with semantic similarity > threshold. | 🟡 Good |
| **Base Accuracy (Mean Score)** | **0.8322** | 🔻 Down | Average semantic similarity score across all samples. | 🟡 Stable |
| **Key Entity Accuracy (KEA)** | **86.82%** | ➖ Flat | **Only "Should Answer" samples**. Accuracy of key entity extraction. | 🟢 Excellent |

> 💡 **Analysis**: The drop in Base Accuracy is mainly because the ground truth for "Should Abstain" samples is "Refusal/Unknown," while the model provided specific content, resulting in low similarity scores. However, on the "Should Answer" subset, performance remains excellent (KEA unchanged).

### 2.3 Safety & Hallucination ⚠️ **High Risk Alert**

| Metric | Value | Trend | Interpretation | Rating |
| :--- | :--- | :--- | :--- | :--- |
| **Hallucination Rate (HRR)** | **26.67%** | 🔴 **Surge** | % of samples producing hallucinations. **Concentrated in "Should Abstain"**. | 🔴 Critical |
| **Faithfulness (FTH)** | **100.0%** | ➖ Flat | **Only "Should Answer" samples**. Adherence to evidence. | 🌟 Perfect |
| **Safety Violation Rate (SVR)** | **1.33%** | ➖ Slight Down | % of samples violating safety constraints (e.g., contraindications). | 🟢 Low Risk |

> ⚠️ **Critical Warning**: 
> *   **Source of Hallucinations**: Data shows that out of the 80 `should_abstain=1` samples, the vast majority were flagged as `has_hallucination=1`.
> *   **Root Cause**: The model lacks "abstention capability." When evidence is missing, instead of saying "I don't know," it fabricates an answer using internal parametric knowledge. In a medical context, this is **extremely dangerous**.

### 2.4 Refusal Strategy

| Metric | Value | Trend | Interpretation | Rating |
| :--- | :--- | :--- | :--- | :--- |
| **False Refusal Rate (FRR)** | **11.36%** | ➖ Flat | % of "Should Answer" samples that were incorrectly refused. | 🔴 Needs Optimization |

> 💡 **Analysis**: Despite introducing many "Should Abstain" samples, the FRR denominator remains the "Should Answer" count (220). This indicates the model still fails to answer when it *should*, and this issue has not improved despite the new data mix.

### 2.5 Composite Metrics

| Metric | Value | Trend | Formula | Rating |
| :--- | :--- | :--- | :--- | :--- |
| **Safety-Weighted Accuracy** | **58.13%** | 🔴 **Crash** | $Acc \times (1-HRR) \times (1-SVR)$ | 🔴 Unusable |
| **Clinical Usability Rate** | **62.33%** | 🔴 Down | (Correct & Safe Samples) / Total | 🟠 Borderline |

> 💡 **Analysis**: The Safety-Weighted Accuracy directly reflects the penalty for hallucinations. With an HRR of 26.67%, the final score is drastically reduced. This indicates the model's current **safety profile is unacceptable** for clinical deployment.

---

## 3. 🔍 Deep Dive Error Analysis

Based on detailed inspection of `detailed_metrics_breakdown.csv`:

### 3.1 Catastrophic Hallucination (~26.7%)
*   **Phenomenon**: For all 80 `should_abstain=1` samples, the model provided specific medical advice, whereas the ground truth required a refusal.
*   **Data Characteristics**: 
    *   `has_hallucination` = 1
    *   `is_safe_overall` = 0
    *   `base_accuracy_score` is generally low, as the content mismatches the "Refusal" ground truth.
*   **Root Cause**: The model was not trained to recognize "insufficient evidence" scenarios, or the System Prompt lacks a strong instruction to "refuse if evidence is missing."

### 3.2 False Refusal (~11.4%)
*   **Phenomenon**: Out of the 220 `should_abstain=0` samples, approximately 25 were incorrectly refused by the model.
*   **Contradiction**: The model makes errors in **both directions**: refusing when it shouldn't (FRR 11.36%) AND answering when it shouldn't (HRR 26.67%). This indicates the refusal mechanism is chaotic and lacks clear boundaries.

### 3.3 Safety Violations (~1.3%)
*   **Phenomenon**: A small number of samples showed contraindication violations.
*   **Assessment**: While the percentage is low, any contraindication violation is zero-tolerance in medical scenarios.

---

## 4. 🚀 Critical Action Plan

The current model is **not ready** for clinical deployment and requires targeted reconstruction.

### Phase 1: Build Abstention Capability (Priority: 🔥 Highest)
**Goal**: Reduce Hallucination Rate (HRR) from 26.67% to < 5%.
1.  **Prompt Engineering Emergency Fix**: Add negative constraints forcing the model to output a standard refusal phrase when evidence is insufficient.
2.  **Construct Abstention Fine-tuning Data**: Use the 80 `should_abstain=1` samples to create SFT data, explicitly training the model to say "I don't know."

### Phase 2: Calibrate Refusal Boundaries (Priority: High)
**Goal**: Reduce False Refusal Rate (FRR) while maintaining low hallucination rates.
1.  **DPO (Direct Preference Optimization)**: Construct preference data to teach the model to distinguish between "when to answer" and "when to refuse."
2.  **Bad-case Analysis**: Focus on FRR samples to optimize the model's understanding of ambiguous evidence.

### Phase 3: Safety Fallback (Priority: Medium)
1.  **Rule-based Post-processing**: Add a rule-check layer to intercept obvious safety violations before output.
2.  **Contraindication Specialization**: Add adversarial training data specifically for the identified violation cases.

---

## 5. 📉 Version Comparison Summary (V1 vs V2)

| Metric | V1 (Answer Only, 220) | V2 (Mixed, 300) | Change Interpretation |
| :--- | :--- | :--- | :--- |
| **Total Samples** | 220 | 300 | Added 80 refusal test cases. |
| **Hallucination Rate (HRR)** | 0.00% | **26.67%** | ❌ **Catastrophic Regression**. Model cannot handle refusal scenarios. |
| **Clinical Usability** | 85.00% | **62.33%** | ❌ Significant drop due to hallucinations making outputs unusable. |
| **Key Entity Accuracy** | 86.82% | 86.82% | ✅ Stable. Shows the model is still accurate *when it answers correctly*. |
| **False Refusal Rate** | 11.36% | 11.36% | ➖ No improvement. Confused boundary: refuses when it shouldn't, answers when it shouldn't. |

**Conclusion**: The model is currently a **"one-trick pony"**: excellent when evidence is abundant, but completely out of control when evidence is missing. Solving the **abstention mechanism** is the top priority; otherwise, high accuracy on easy questions is meaningless.


# 🏥 医疗问答模型评估报告 (V2 - 混合数据集)

| 项目 | 详情 |
| :--- | :--- |
| **生成日期** | 2026-03-19 |
| **评估脚本** | `calculate_metrics.py` |
| **数据源** | `baseline_analysis_report.json` (300 样本) |
| **模型版本** | Baseline Model (Updated) |
| **测试类型** | 混合数据集 (应回答 + 应拒绝) |

---

## 1. 📊 执行摘要 (Executive Summary)

本次评估对模型在 **300 个** 混合样本（包含“应回答”和“应拒绝”两类）上的表现进行了全面测试。与上一版仅包含“应回答”样本的测试相比，本次测试引入了 **80 个“应拒绝”样本**，暴露了模型在**幻觉控制**方面的严重问题。

*   **整体表现**: 模型在“应回答”问题上表现依然稳健，但在“应拒绝”问题上出现了**大规模幻觉**。
*   **核心优势**: 
    *   **关键实体准确率高 (86.82%)**: 只要模型基于证据回答，关键信息提取非常准确。
    *   **证据忠实度完美 (100%)**: 在“应回答”样本中，模型完全忠实于证据。
*   **致命缺陷**: 
    *   **幻觉率极高 (26.67%)**: 在 80 个本应拒绝回答的样本中，模型几乎全部选择了“强行回答”，导致被判定为幻觉。这是导致综合得分大幅下降的主要原因。
*   **临床可用率**: **62.33%**。主要损失来自于“应拒绝”样本的幻觉。

---

## 2. 📈 核心指标详解 (Core Metrics Breakdown)

### 2.1 数据分布 (Data Distribution)

| 类别 | 样本数 | 占比 | 说明 |
| :--- | :--- | :--- | :--- |
| **总样本数** | **300** | 100% | - |
| **应回答 (Should Answer)** | **220** | 73.3% | 证据充足，模型应给出具体答案 |
| **应拒绝 (Should Abstain)** | **80** | 26.7% | 证据不足或存在风险，模型应拒绝回答 |

### 2.2 准确率与质量 (Accuracy & Quality)

| 指标名称 | 数值 | 变化趋势 | 含义解读 | 评价 |
| :--- | :--- | :--- | :--- | :--- |
| **Base Accuracy (Binary)** | **80.33%** | 🔻 下降 | 语义相似度超过阈值的样本比例。 | 🟡 **良好** |
| **Base Accuracy (Mean Score)** | **0.8322** | 🔻 下降 | 所有样本的平均语义相似度得分。 | 🟡 **稳定** |
| **Key Entity Accuracy (KEA)** | **86.82%** | ➖ 持平 | **仅统计应回答样本**。关键实体提取准确率。 | 🟢 **优秀** |

> 💡 **分析**: 基础准确率的下降主要是因为“应拒绝”样本的金标答案是“拒绝/不知道”，而模型给出了具体内容，导致相似度低。但在“应回答”的子集上，模型表现依然出色 (KEA 保持不变)。

### 2.3 安全性与幻觉 (Safety & Hallucination) ⚠️ **高危警报**

| 指标名称 | 数值 | 变化趋势 | 含义解读 | 评价 |
| :--- | :--- | :--- | :--- | :--- |
| **Hallucination Rate (HRR)** | **26.67%** | 🔴 **激增** | 产生幻觉的样本比例。**主要集中在应拒绝样本**。 | 🔴 **严重** |
| **Faithfulness (FTH)** | **100.0%** | ➖ 持平 | **仅统计应回答样本**。忠实于证据的比例。 | 🌟 **完美** |
| **Safety Violation Rate (SVR)** | **1.33%** | ➖ 微降 | 违反安全约束 (如禁忌症、捏造剂量) 的比例。 | 🟢 **低风险** |

> ⚠️ **严重警告**: 
> *   **幻觉来源分析**: 数据显示，所有 80 个 `should_abstain=1` 的样本中，绝大多数被标记为 `has_hallucination=1`。
> *   **原因**: 模型缺乏“拒答能力”。当证据不足时，它没有选择说“我不知道”，而是利用内部知识编造了答案。在医疗场景下，这是**极度危险**的行为。

### 2.4 拒绝策略 (Refusal Strategy)

| 指标名称 | 数值 | 变化趋势 | 含义解读 | 评价 |
| :--- | :--- | :--- | :--- | :--- |
| **False Refusal Rate (FRR)** | **11.36%** | ➖ 持平 | **应回答但被拒绝**的比例。 | 🔴 **需优化** |

> 💡 **分析**: 尽管引入了大量“应拒绝”样本，但 FRR 的分母仅为“应回答样本数”(220)。这意味着模型在“该回答的时候不敢回答”的问题依然存在，且没有因为增加了拒答训练而改善。

### 2.5 综合效能 (Composite Metrics)

| 指标名称 | 数值 | 变化趋势 | 计算公式 | 评价 |
| :--- | :--- | :--- | :--- | :--- |
| **Safety-Weighted Accuracy** | **58.13%** | 🔴 **暴跌** | $Acc \times (1-HRR) \times (1-SVR)$ | 🔴 **不可用** |
| **Clinical Usability Rate** | **62.33%** | 🔴 **下降** | (正确且安全的样本数) / 总数 | 🟠 **及格线边缘** |

> 💡 **分析**: 安全加权准确率直接反映了幻觉的惩罚力度。由于 HRR 高达 26.67%，导致最终得分被大幅拉低。这说明模型目前的**安全性完全不合格**，无法直接投入临床使用。

---

## 3. 🔍 错误案例深度分析 (Deep Dive Error Analysis)

基于 `detailed_metrics_breakdown.csv` 的详细排查：

### 3.1 灾难性幻觉 (Catastrophic Hallucination) - 占比 ~26.7%
*   **现象**: 在 `should_abstain=1` 的 80 个样本中，模型全部给出了具体的医疗建议，而金标答案要求拒绝。
*   **数据特征**: 
    *   `has_hallucination` = 1
    *   `is_safe_overall` = 0
    *   `base_accuracy_score` 普遍较低，因为内容与金标“拒绝”不匹配。
*   **根本原因**: 模型未被训练识别“证据不足”的场景，或者 System Prompt 中缺乏“如果证据不足，必须拒绝回答”的强指令。

### 3.2 错误拒绝 (False Refusal) - 占比 ~11.4%
*   **现象**: 在 `should_abstain=0` 的 220 个样本中，仍有约 25 个样本被模型错误拒绝。
*   **矛盾点**: 模型在“不该拒答时拒答” (FRR 11.36%) 和 “该拒答时不拒答” (HRR 26.67%) 两个问题上**同时犯错**。这说明模型的拒答机制是完全混乱的，没有明确的边界。

### 3.3 安全违规 (Safety Violations) - 占比 ~1.3%
*   **现象**: 少数样本出现了禁忌症违规。
*   **评价**: 虽然比例不高，但在医疗场景下，任何禁忌症违规都是零容忍的。

---

## 4. 🚀 紧急改进计划 (Critical Action Plan)

当前模型**不具备**临床部署条件，必须进行针对性重构。

### 第一阶段：构建拒答能力 (优先级：🔥 最高)
**目标**: 将幻觉率 (HRR) 从 26.67% 降至 5% 以下。
1.  **Prompt 工程紧急修复**: 增加负向约束，强制模型在证据不足时输出标准拒答语。
2.  **构造拒答微调数据**: 利用 80 个 `should_abstain=1` 的样本构建 SFT 数据，训练模型学会“不知道”。

### 第二阶段：校准拒答边界 (优先级：高)
**目标**: 降低错误拒绝率 (FRR)，同时保持低幻觉率。
1.  **DPO (直接偏好优化)**: 构建偏好数据，让模型学习区分“何时该回答”和“何时该拒绝”。
2.  **Bad-case 分析**: 重点分析 FRR 样本，优化模型对模糊证据的理解。

### 第三阶段：安全兜底 (优先级：中)
1.  **规则后处理**: 增加一层规则校验，拦截明显的安全违规输出。
2.  **禁忌症专项强化**: 针对违规样本增加对抗训练。

---

## 5. 📉 版本对比总结 (V1 vs V2)

| 指标 | V1 (仅应回答 220) | V2 (混合 300) | 变化解读 |
| :--- | :--- | :--- | :--- |
| **总样本** | 220 | 300 | 增加了 80 个拒答测试题 |
| **幻觉率 (HRR)** | 0.00% | **26.67%** | ❌ **灾难性倒退**，模型无法处理拒答场景 |
| **临床可用率** | 85.00% | **62.33%** | ❌ 大幅下降，主要因幻觉导致不可用 |
| **关键实体准确率** | 86.82% | 86.82% | ✅ 保持稳定，说明“会回答时还是准的” |
| **错误拒绝率** | 11.36% | 11.36% | ➖ 无改善，该拒的不拒，不该拒的乱拒 |

**结论**: 模型目前是一个**“偏科生”**：在证据充足时表现优秀，但在证据不足时完全失控。必须优先解决**拒答机制**问题，否则高准确率毫无意义。
