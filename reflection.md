# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 30.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.799 | 0.037 | 1.000 | High precision but some missed contexts |
| Context Precision | 0.925 | 0.000 | 1.000 | Very high precision |
| Faithfulness | 0.787 | 0.000 | 1.000 | Generally grounded, 2 hallucinations |
| Relevance | 0.365 | 0.000 | 0.833 | Very low relevance overall |
| Completeness | 0.833 | 0.000 | 1.000 | Mostly complete when relevant |
| Overall Score | 0.662 | 0.022 | 0.944 | Relevance dragged down the overall score |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision, Context Recall, Faithfulness
- Metrics/cases ở mức Needs Work (0.6–0.8): Completeness
- Metrics/cases ở mức Significant Issues (<0.6): Relevance

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 2 | 10.0% |
| irrelevant | 7 | 35.0% |
| incomplete | 0 | 0.0% |
| off_topic | 5 | 25.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Generation. Context Recall and Precision are high, showing retriever is finding the right info. However, Relevance is very low, meaning the generation is missing the user intent or providing too much noise.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* H02 - I am an international student and want to withdraw from all my courses to take a medical leave. What steps must I take?

**Expected answer:**

> *Điền:* You must consult the International Student Office before reducing your registered load, submit a leave request before the census date or retroactively within 30 days of last participation with medical documentation, and follow the withdrawal procedures.

**Actual answer:**

> *Điền:* Just withdraw on the portal.

**Scores:** Context Recall: 0.037 | Context Precision: 0.000 | Faithfulness: 0.000 |
Relevance: 0.067 | Completeness: 0.000 | Overall: 0.022

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* The retriever pulled incorrect info ("Portal withdrawal is allowed"). Missed crucial chunks about International Student Office.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Answer is hallucinatory and wrong |
| Why 1 | Tại sao symptom xảy ra? | Generator got bad context |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever missed the actual policy doc |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Missing fallback for empty or low confidence retrievals |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | No threshold on retriever score |
| Why 5 | Root cause có thể hành động được là gì? | Add retrieval threshold and refine embeddings for edge cases |

**Root cause từ `find_root_cause()`:**

> *Paste output:* Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Yes, context recall is extremely low (0.037), meaning the evidence wasn't there to answer the question safely.

**Proposed fix cụ thể:**

> *Câu trả lời:* Improve retriever embedding logic to capture specific entity needs like "International Student Office".

### Failure 2

**ID và question:**

> *Điền:* E05 - What percentage of scheduled sessions are students expected to attend?

**Expected answer:**

> *Điền:* Students are expected to attend at least 80% of scheduled sessions in courses that record attendance.

**Actual answer:**

> *Điền:* Students must attend 100% of classes and get A grades.

**Scores:** Context Recall: 0.100 | Context Precision: 1.000 | Faithfulness: 0.286 |
Relevance: 0.286 | Completeness: 0.200 | Overall: 0.257

**Evidence inspection:**

> *Câu trả lời:* Retriever pulled an out of context chunk ("Students get A grades") causing the model to hallucinate the answer.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Hallucinated answer with incorrect information |
| Why 1 | Tại sao symptom xảy ra? | Model invented "100%" and "A grades" |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Context retrieved was useless/unrelated |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Generator did not have a strict "I don't know" rule |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Prompt guardrails are not strict enough |
| Why 5 | Root cause có thể hành động được là gì? | Improve prompt to refuse answering without context |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause: Context is missing or irrelevant — improve retrieval. Fix: Add strict prompt guardrails to output "I don't know" if evidence is lacking.

### Failure 3

**ID và question:**

> *Điền:* M01 - If I drop a course before the census date, will it affect my scholarship?

**Expected answer:**

> *Điền:* Dropping below 12 graded credits on or before the census date triggers an immediate scholarship eligibility review.

**Actual answer:**

> *Điền:* The census date is in September.

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.667 |
Relevance: 0.182 | Completeness: 0.154 | Overall: 0.334

**Evidence inspection:**

> *Câu trả lời:* Retriever pulled the correct chunks with perfect recall and precision, but the generator completely missed answering the question.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Irrelevant answer despite perfect context |
| Why 1 | Tại sao symptom xảy ra? | Generator ignored the user intent |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Model is focusing on surface keywords ("census date") |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt does not emphasize addressing the specific user question |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Lack of intent analysis before generation |
| Why 5 | Root cause có thể hành động được là gì? | Refine intent detection logic |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause: Answer does not address the question — improve prompt clarity. Fix: Explicitly instruct model to directly address the user's intent rather than restating facts.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Answer does not address the question — improve prompt clarity | M01, M02, M03, E04, A01, A02 | High |
| 2 | Context is missing or irrelevant — improve retrieval | H02, E05 | Medium |
| 3 | Answer is missing key information — increase context window or improve generation | H01, M06, H04, H05, A03 | Low |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Cluster 1 (Relevance issue). With an average relevance of 0.365 and 7 irrelevant failures, it is the most widespread issue. Since retrieval is already good (Recall ~0.8), fixing the prompt generator will yield the highest ROI.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| E04 | irrelevant | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| E05 | hallucination | Answer is missing key information — increase context window or improve generation | Improve prompt clarity to address user intent | Open |
| M01 | irrelevant | Answer is missing key information — increase context window or improve generation | Enhance intent detection to filter out off-topic requests | Open |
| M02 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| M03 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| M04 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| M06 | off_topic | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| H01 | off_topic | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| H02 | hallucination | Multiple issues detected — review full pipeline | Review failure and tune model/pipeline | Open |
| H04 | off_topic | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| H05 | off_topic | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| A01 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| A02 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |
| A03 | off_topic | Answer does not address the question — improve prompt clarity | Review failure and tune model/pipeline | Open |

**Ba improvement suggestions ưu tiên**

1. Implement hallucination checker to filter unsupported claims
2. Improve prompt clarity to address user intent
3. General pipeline review and tuning

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Implement hallucination checker | Faithfulness | Measure average faithfulness pre/post deployment |
| Improve prompt clarity | Relevance | Evaluate average answer relevance score |
| General pipeline review | Overall Score | Run BenchmarkRunner and run_regression |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> On every code release or prompt change (pull request).

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:*
> Yes. A 0.05 drop in metrics like Faithfulness or Completeness could translate into many students receiving wrong or missing policy details, which can have financial or academic consequences.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> Faithfulness and Completeness must block deployment. Relevance and Context Recall/Precision can be set to alert.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Eval] → [Regression Check] → [Human Approval] → Deploy
```

> *Giải thích:*
> First run full evaluation offline. Then check for regression compared to baseline. Finally, if there is a warning or alert, involve human approval before deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Improve system prompt intent targeting | Relevance | Higher satisfaction for complex questions |
| 2 | Add retrieval thresholding | Faithfulness | Lower hallucination rates |
| 3 | Enhance context window limit | Completeness | Less missing information in long answers |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> Add H02 and M01. H02 will test the retriever improvements. M01 will test the prompt generator improvements.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> The fact that retrieval recall was very high (0.8+) but relevance was abysmally low (~0.3). I assumed if the right context was there, the LLM would naturally give a relevant answer.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> Word-overlap is rigid and penalizes synonyms or paraphrasing. In production, I would replace it with an LLM-as-a-Judge using a strict rubric to evaluate semantic similarity and adherence.
