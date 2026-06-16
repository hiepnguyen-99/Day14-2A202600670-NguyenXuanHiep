# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Paste results từ Exercise 3.2 và tóm tắt:

**Overall pass rate:** ____%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | | | | |
| Relevance | | | | |
| Completeness | | | | |
| Overall Score | | | | |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8–1.0)? ___
- Bao nhiêu metrics ở Needs Work (0.6–0.8)? ___
- Bao nhiêu metrics ở Significant Issues (<0.6)? ___

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | | |
| irrelevant | | |
| incomplete | | |
| off_topic | | |
| refusal | | |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

Theo bài giảng: "Phân loại failure TRƯỚC KHI fix. Đừng fix từng failure riêng lẻ — CLUSTER rồi fix root cause."

### Failure 1

**Question:** *paste question here*

**Agent Answer:** *paste actual output*

**Scores:** Faithfulness: ___ | Relevance: ___ | Completeness: ___ | Overall: ___

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | |
| Why 1 | Tại sao xảy ra? | |
| Why 2 | Tại sao Why 1 xảy ra? | |
| Why 3 | Tại sao Why 2 xảy ra? | |
| Why 4 | Root cause là gì? | |

**Root cause (from `find_root_cause()`):**
> *Output của function:*

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> *Your answer:*

**Proposed fix (cụ thể, actionable):**
> *Your answer: 1–2 actions cụ thể*

---

### Failure 2

**Question:** *paste question here*

**Agent Answer:** *paste actual output*

**Scores:** Faithfulness: ___ | Relevance: ___ | Completeness: ___ | Overall: ___

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | | |
| Why 1 | | |
| Why 2 | | |
| Why 3 | | |
| Why 4 | | |

**Root cause:**
> *Your answer:*

**Proposed fix:**
> *Your answer:*

---

### Failure 3

**Question:** *paste question here*

**Agent Answer:** *paste actual output*

**Scores:** Faithfulness: ___ | Relevance: ___ | Completeness: ___ | Overall: ___

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | | |
| Why 1 | | |
| Why 2 | | |
| Why 3 | | |
| Why 4 | | |

**Root cause:**
> *Your answer:*

**Proposed fix:**
> *Your answer:*

---

## 3. Failure Clustering

Theo bài giảng: "Fix 1 root cause giải quyết nhiều failures cùng lúc."

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | | | High/Medium/Low |
| 2 | | | |
| 3 | | | |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> *Your answer:*

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```
[paste markdown table output here]
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. ___
2. ___
3. ___

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Chạy trong CI trước khi merge vào main (mỗi PR), sau mỗi thay đổi prompt/retriever/index; có thể thêm nightly run để bắt drift.

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> Thường phù hợp cho baseline chung. Nếu domain rủi ro cao (compliance/finance) → strict hơn (0.02–0.03). Nếu domain chat không critical → có thể loose hơn.

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> Block nếu regression xảy ra ở metric chính (đặc biệt faithfulness). Alert nếu regression nhỏ ở metric phụ hoặc do noise tạm thời, nhưng vẫn cần review.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**
> Code change → [Unit tests] → [Offline benchmark + run_regression] → [Staging/Canary + monitoring] → Deploy


---

## 6. Continuous Improvement Loop

Theo bài giảng: Evaluate → Analyze → Improve → Augment (add to benchmark) → lặp lại

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
|1|	Cải thiện retrieval (hybrid search + tăng top-k + metadata filter)|Context Recall / Faithfulness|Giảm thiếu evidence → giảm bịa|
|2|Thêm reranking (cross-encoder hoặc lexical rerank)|Context Precision / Relevance|Giảm nhiễu, tăng đúng trọng tâm|
|3|Prompt theo checklist + self-check “đủ ý chưa?”|Completeness|Trả lời đầy đủ và có cấu trúc|

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** _____ (RAGAS-inspired heuristic)

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> *Tham khảo trade-offs table trong bài giảng:*

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... |RAGAS/DeepEval/TruLens có metric chuẩn + tích hợp retrieval/generation tốt hơn heuristic.|
| CI/CD integration vì... |Có API/CLI rõ ràng để chạy regression, lưu artifacts, và gate deploy.|
| Team workflow vì... |Dễ chuẩn hoá rubric, theo dõi history, debug failure theo dashboard/log.|
