# 🏥 Medical QA Model Evaluation Report (V2 - Mixed Dataset)

| Item | Details |
| :--- | :--- |
| **Date** | 2023-10-XX |
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
*   **Clinical Usability Rate**: **62.33%**. A sharp decline from the previous 85%. The loss is primarily due to hallucinations in the "Should Abstain" category.

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

---
*Generated by Automated Evaluation Pipeline*
