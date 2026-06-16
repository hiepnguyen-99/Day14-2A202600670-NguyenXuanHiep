# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Kết quả benchmark (Exercise 3.2):

**Overall pass rate:** **0.00%** (0/20)

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.602 | 0.000 | 1.000 | 0.452 |
| Relevance | 0.206 | 0.000 | 0.833 | 0.264 |
| Completeness | 0.141 | 0.000 | 0.545 | 0.168 |
| Overall Score | 0.316 | 0.000 | 0.648 | 0.225 |

**Score interpretation (theo bài giảng):**
- Good (0.8–1.0): **0**
- Needs Work (0.6–0.8): **1**
- Significant Issues (<0.6): **19**

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 7 | 35.00% |
| irrelevant | 7 | 35.00% |
| incomplete | 2 | 10.00% |
| off_topic | 4 | 20.00% |
| refusal | 0 | 0.00% |

**Notable observation (quan trọng):**
- Có ít nhất **4 câu** bị lỗi **Gemini 429 Too Many Requests**, dẫn đến câu trả lời rỗng/không ổn định, kéo tụt mạnh Relevance/Completeness và làm pass rate = 0%.
- Nhiều câu trả lời ra dạng: **“I don't know based on the provided context.”** dù thực tế context trong dataset có chứa đáp án → nghi ngờ **context injection bị lỗi / mismatch** hoặc prompt quá “strict” khiến model hay từ chối.

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

Theo bài giảng: phân loại failure trước khi fix. Không fix từng lỗi lẻ — cluster rồi fix root cause.

### Failure 1

**Question:** *Name three common LLM-as-Judge biases.*

**Agent Answer:**
I don't know based on the provided context.

**Scores:** Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**5 Whys Analysis:**

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent trả lời “I don’t know…” → điểm 0 toàn bộ. |
| Why 1 | Tại sao xảy ra? | Model “tin rằng” context không đủ/không liên quan nên kích hoạt câu trả lời từ chối theo prompt. |
| Why 2 | Tại sao model nghĩ context không đủ? | Context có thể **không được inject đúng** vào prompt (bị rỗng/None) hoặc không lấy đúng QAPair tương ứng. |
| Why 3 | Tại sao context có thể rỗng/không đúng? | `agent_fn(q: str)` đang tra `QA_MAP.get(q)` — nếu runner truyền vào `q` có khác biệt (trailing spaces/newlines/khác kiểu dữ liệu) → lookup fail → context = "" → model luôn “I don’t know…”. |
| Why 4 | Root cause là gì? | **Bug/thiết kế sai ở pipeline glue code (agent_fn & dataset interface)**: context không được đảm bảo truyền vào đúng + thiếu assert/log để phát hiện sớm. |

**Root cause (from `find_root_cause()`):**
> Multiple issues detected — review full pipeline

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý. Vì output cho thấy nhiều loại failure cùng lúc (irrelevant/off_topic/hallucination/incomplete) và đặc biệt có pattern “I don’t know…” hàng loạt + 429 rate limit. Đây thường là dấu hiệu lỗi hệ thống/pipeline (context không vào đúng, request API bị fail), không phải chỉ “một câu trả lời kém”.

**Proposed fix (cụ thể, actionable):**
1. **Sửa `agent_fn` để nhận đúng input**: hỗ trợ cả trường hợp runner truyền vào `QAPair` object; nếu là string thì `.strip()` trước khi lookup; nếu lookup fail thì log cảnh báo + raise (đừng im lặng context="").  
2. **Thêm kiểm tra “context must be non-empty”** cho các câu non-adversarial (easy/medium/hard). Nếu context rỗng → fail fast (để phát hiện bug sớm thay vì làm benchmark sai).  

---

### Failure 2

**Question:** *What is regression testing in an eval pipeline?*

**Agent Answer:**
I don't know based on the provided context.

**Scores:** Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**5 Whys Analysis:**

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent từ chối dù dataset có context mô tả “regression = metric drop > 0.05 vs baseline”. |
| Why 1 | Tại sao agent từ chối? | Prompt bắt buộc “ONLY the CONTEXT” và cho phép trả lời “I don't know…” nếu thiếu. |
| Why 2 | Tại sao prompt lại dẫn đến từ chối? | Nếu context inject bị rỗng hoặc bị format sai (không đọc được), model sẽ coi như “không có bằng chứng”. |
| Why 3 | Tại sao context có thể bị format sai/rỗng? | Pipeline hiện không có logging: không in ra length/context snippet trước khi gọi Gemini; thêm vào đó có 429 ở vài câu làm kết quả benchmark thiếu ổn định → khó debug. |
| Why 4 | Root cause là gì? | **Thiếu observability + robustness**: không có guardrails kiểm tra input/context, không có retry/backoff tốt, và không có caching → benchmark dễ rơi vào trạng thái fail hàng loạt. |

**Root cause:**
> Root cause chính: context không được đảm bảo truyền đúng + thiếu log/guardrails; cộng thêm Gemini rate-limit gây “flaky run”.

**Proposed fix:**
1. Thêm log debug tối thiểu: `print(f"[DEBUG] qid={id} ctx_len={len(context)}")` và (tùy chọn) in 80 ký tự đầu context khi chạy local.  
2. Bật cơ chế **retry + exponential backoff + throttle + cache** để không bị 429 làm “rụng” câu trả lời (đảm bảo run ổn định).  

---

### Failure 3

**Question:** *Propose CI/CD thresholds for Faithfulness, Relevance, Completeness and justify briefly.*

**Agent Answer:**
I don't know based on the provided context.

**Scores:** Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**5 Whys Analysis:**

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent không đưa ra threshold dù đây là câu “hard” nhưng context trong dataset có ví dụ threshold. |
| Why 1 | Tại sao không đưa ra được? | Model trả lời theo fallback “I don’t know…” (refusal-like). |
| Why 2 | Tại sao fallback bị kích hoạt? | Prompt quá strict + (rất có thể) context không được đưa vào / lookup QAPair bị fail. |
| Why 3 | Tại sao lookup/context dễ fail mà không phát hiện? | Không có assert/exception khi lookup fail; không có unit test kiểm tra “agent_fn inject context đúng cho 20 câu”. |
| Why 4 | Root cause là gì? | **Thiếu test + thiếu ràng buộc giao diện (contract) giữa BenchmarkRunner và agent_fn** → chạy được nhưng sai logic. |

**Root cause:**
> Root cause chính là “pipeline contract mismatch” + prompt strict khiến hệ thống dễ rơi vào refusal.

**Proposed fix:**
1. Viết 1 unit test nhỏ: với mỗi QAPair, build prompt và assert `context` xuất hiện trong prompt + `ctx_len > 0`.  
2. Thêm 1 “few-shot mini example” trong prompt: nếu context có đáp án rõ ràng thì phải trả lời trực tiếp, không được nói “I don’t know…”.  

---

## 3. Failure Clustering

Theo bài giảng: fix 1 root cause giải quyết nhiều failures cùng lúc.

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | **Context injection / QA lookup mismatch** (context rỗng hoặc sai QAPair → “I don’t know…”, off_topic/irrelevant) | Likely affects đa số cases (đặc biệt các case Overall=0) | **High** |
| 2 | **Gemini API rate limit 429** → trả lời rỗng/flaky → metrics tụt mạnh | Ít nhất 4 cases bị log 429 (và có thể nhiều hơn) | **High** |
| 3 | **Prompt quá strict + thiếu few-shot + thiếu format constraints** → dễ refusal/thiếu completeness hoặc lạc đề | Nhiều case bị off_topic/incomplete | Medium |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn **Cluster 1 (context injection / QA lookup mismatch)** trước. Vì nếu context không vào đúng, mọi metric downstream đều meaningless (evaluation không phản ánh năng lực model mà phản ánh bug pipeline). Fix cluster này thường kéo được Relevance/Completeness lên ngay và làm failure analysis “thật” hơn.

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | incomplete | Answer is missing key information — increase context window or improve generation | Add a groundedness guardrail: only answer using claims supported by retrieved context (cite or quote context snippets). | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Improve retrieval quality (better embeddings / better chunking) so the generator has the right evidence to stay faithful. | Open |
| F003 | off_topic | Answer is missing key information — increase context window or improve generation | Tighten the prompt: restate the question, enforce answering only that intent, and add few-shot examples for on-topic answers. | Open |
| F004 | hallucination | Multiple issues detected — review full pipeline | Increase completeness: ask the model to cover all key points from the expected answer; consider increasing context window or adding follow-up retrieval. | Open |
| F005 | off_topic | Multiple issues detected — review full pipeline |  | Open |
| F006 | irrelevant | Multiple issues detected — review full pipeline |  | Open |
| F007 | irrelevant | Multiple issues detected — review full pipeline |  | Open |
| F008 | hallucination | Answer does not address the question — improve prompt clarity |  | Open |
| F009 | hallucination | Multiple issues detected — review full pipeline |  | Open |
| F010 | irrelevant | Answer does not address the question — improve prompt clarity |  | Open |
| F011 | irrelevant | Answer is missing key information — increase context window or improve generation |  | Open |
| F012 | incomplete | Answer is missing key information — increase context window or improve generation |  | Open |
| F013 | hallucination | Multiple issues detected — review full pipeline |  | Open |
| F014 | hallucination | Multiple issues detected — review full pipeline |  | Open |
| F015 | irrelevant | Multiple issues detected — review full pipeline |  | Open |
| F016 | irrelevant | Multiple issues detected — review full pipeline |  | Open |
| F017 | off_topic | Answer is missing key information — increase context window or improve generation |  | Open |
| F018 | hallucination | Multiple issues detected — review full pipeline |  | Open |
| F019 | irrelevant | Answer is missing key information — increase context window or improve generation |  | Open |
| F020 | hallucination | Context is missing or irrelevant — improve retrieval |  | Open |
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Add a groundedness guardrail: only answer using claims supported by retrieved context (cite or quote context snippets).
2. Improve retrieval quality (better embeddings / better chunking) so the generator has the right evidence to stay faithful.
3. Tighten the prompt: restate the question, enforce answering only that intent, and add few-shot examples for on-topic answers.

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
| 1 | Fix pipeline contract: đảm bảo context được inject đúng (handle QAPair vs str, strip key, assert ctx_len>0, add debug logs) | Relevance / Completeness (và Overall) | Tránh “I don’t know…” sai, làm benchmark phản ánh đúng năng lực |
| 2 | Làm ổn định Gemini calls: throttle + retry/backoff + đọc Retry-After + cache kết quả | Pass rate / Relevance / Completeness | Giảm flaky do 429, tránh answer rỗng |
| 3 | Tối ưu prompt: thêm few-shot, bắt buộc trả lời nếu context có đáp án; yêu cầu format ngắn gọn theo bullet | Off_topic / Incomplete | Tăng độ đúng trọng tâm và đầy đủ ý |

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** **RAGAS-inspired heuristic** (faithfulness / relevance / completeness + overall)

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | Chọn framework có metric chuẩn + tooling tốt (RAGAS/DeepEval/TruLens) để đánh giá groundedness & retrieval rõ ràng, có artifacts/debug trace. |
| CI/CD integration vì... | Có API/CLI rõ ràng để chạy regression, lưu kết quả theo commit, và gate deploy theo threshold. |
| Team workflow vì... | Dễ chuẩn hóa rubric, lưu lịch sử benchmark, và chia sẻ dashboard/log để debug failure theo cluster. |
