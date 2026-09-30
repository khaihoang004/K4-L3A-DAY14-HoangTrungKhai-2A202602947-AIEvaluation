# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up

### Exercise 1.1 — RAGAS Metric Thresholds

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | 0.6–0.8 trong câu trả lời ít rủi ro, có thể review | <0.6 hoặc claim chính sách không có evidence | Block claim, kiểm tra context và prompt |
| Answer Relevance | Câu hỏi mơ hồ nhưng vẫn nêu limitation đúng | <0.6 hoặc trả lời sai intent | Cải thiện intent/query rewrite |
| Context Recall | Câu hỏi đơn giản không cần nhiều evidence | <0.6 với policy nhiều điều kiện | Mở rộng retrieval/chunking |
| Context Precision | Có noise nhưng chunk đúng vẫn ở đầu | <0.6 hoặc evidence chính bị chôn | Rerank và giảm noise |
| Completeness | Câu hỏi chỉ cần một fact và đã đủ ý | <0.6 với policy/edge case | Tăng context và thêm checklist answer |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> **Position bias:** chấm cùng cặp answer A/B ở hai điều kiện: A trước/B sau và B trước/A sau. Đảo thứ tự ngẫu nhiên, lặp nhiều lần; bias xuất hiện nếu cùng một answer được ưu tiên khi đứng trước.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> **Verbosity bias:** rubric chấm coverage, correctness và evidence theo claim; không cộng điểm chỉ vì dài. Câu trả lời ngắn nhưng đủ ý phải có thể đạt 5.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> **Self-preference:** dùng nhiều judge/model khác nhau, ẩn model/metadata, và calibrate với human labels.

### Exercise 1.3 — Evaluation trong CI/CD

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Claim không grounded có rủi ro cao, phải block |
| Answer Relevance | 0.60 | Cần phát hiện trả lời lệch intent |
| Completeness | 0.60 | Policy thiếu điều kiện có thể gây hành động sai |

Offline eval chạy trước merge và sau prompt/retriever/model change; online eval theo dõi production drift và feedback; human review dùng cho policy, privacy, safety và các failure mới.

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

Đại diện: E01 (scope, easy), M01 (orders/cancellation, medium), A01 (prompt injection, adversarial). Evidence đều là substring nguyên văn từ corpus; validator đã PASS.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | 0.938 | 1.000 | 0.667 | 0.667 | 0.938 | 0.757 | Yes | - |
| E02 | 0.800 | 1.000 | 0.200 | 0.750 | 0.400 | 0.450 | No | hallucination |
| E03 | 0.625 | 0.887 | 0.067 | 0.667 | 0.625 | 0.453 | No | hallucination |
| E04 | 0.818 | 0.833 | 0.300 | 0.571 | 0.636 | 0.503 | No | off_topic |
| E05 | 0.400 | 0.833 | 0.000 | 0.000 | 0.000 | 0.000 | No | hallucination |
| M01 | 0.667 | 1.000 | 0.143 | 0.400 | 0.444 | 0.329 | No | hallucination |
| M02 | 0.000 | 0.000 | 0.000 | 0.000 | 0.000 | 0.000 | No | hallucination |
| M03 | 0.200 | 0.750 | 0.074 | 1.000 | 0.400 | 0.491 | No | hallucination |
| M04 | 0.286 | 1.000 | 0.176 | 0.286 | 0.143 | 0.202 | No | hallucination |
| M05 | 0.500 | 0.450 | 0.122 | 0.500 | 0.333 | 0.319 | No | hallucination |
| M06 | 0.700 | 1.000 | 0.107 | 0.400 | 0.200 | 0.236 | No | hallucination |
| M07 | 0.400 | 0.639 | 0.000 | 0.500 | 0.200 | 0.233 | No | hallucination |
| H01 | 0.700 | 1.000 | 0.149 | 0.500 | 0.300 | 0.316 | No | hallucination |
| H02 | 1.000 | 1.000 | 0.130 | 0.750 | 0.800 | 0.560 | No | hallucination |
| H03 | 0.778 | 1.000 | 0.167 | 0.667 | 0.556 | 0.463 | No | hallucination |
| H04 | 0.900 | 1.000 | 0.333 | 0.667 | 0.500 | 0.500 | No | off_topic |
| H05 | 0.800 | 1.000 | 0.238 | 0.429 | 0.800 | 0.489 | No | hallucination |
| A01 | 1.000 | 0.750 | 0.000 | 0.500 | 0.750 | 0.417 | No | hallucination |
| A02 | 0.200 | 1.000 | 0.098 | 0.200 | 0.200 | 0.166 | No | hallucination |
| A03 | 0.667 | 1.000 | 0.067 | 0.250 | 0.333 | 0.217 | No | hallucination |

**Aggregate Report**

- Overall pass rate: 5.0%
- Avg Context Recall: 0.619
- Avg Context Precision: 0.857
- Avg Faithfulness: 0.152
- Avg Relevance: 0.485
- Avg Completeness: 0.428
- Failure type distribution: {'hallucination': 17, 'off_topic': 2}

**Ba cases có Overall Score thấp nhất**

1. ID: E05  | Score: 0.000 | Failure type: hallucination
2. ID: M02  | Score: 0.000 | Failure type: hallucination
3. ID: A02  | Score: 0.166 | Failure type: hallucination

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*


### Exercise 3.3 — Domain Rubric

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Dimensions: Correctness, Completeness, Evidence, Actionability, Safety/privacy.

| Score | Tiêu chí domain-specific |
|---:|---|
| 5 | Đúng hoàn toàn theo corpus, đủ điều kiện/ngoại lệ, evidence rõ, hành động an toàn, không lộ dữ liệu. |
| 4 | Đúng phần lớn, thiếu một chi tiết phụ nhưng không làm sai policy hay hành động. |
| 3 | Đúng ý chính nhưng thiếu điều kiện/evidence/bước xử lý khiến khách phải hỏi lại. |
| 2 | Có lỗi policy, bỏ sót điều kiện quan trọng hoặc hướng dẫn chưa an toàn. |
| 1 | Sai/không liên quan, bịa claim, hoặc làm theo yêu cầu lộ prompt/private data. |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

| Tiêu chí | RAGAS | DeepEval |
|---|---|---|
| Setup complexity | Dataset + metrics, phù hợp RAG | Test-case + metric objects, tích hợp test rõ |
| Metrics available | Faithfulness, relevance, recall, precision | Faithfulness, answer relevancy, hallucination và custom metrics |
| CI/CD integration | Có thể chạy qua pytest/script | Tự nhiên với test runner/CI |
| Kết quả trecùng dataset | Có thể strict theo overlap/LLM judge | Có thể khác do threshold/judge |
| Insight | Mạnh về RAG pipeline metrics | Mạnh về assertion và quality gate |

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| E01 | 0.938 | 0.938 | 1.000 | 1.000 | 0.000 |
| E02 | 0.800 | 0.800 | 1.000 | 1.000 | 0.000 |
| E03 | 0.625 | 0.625 | 0.887 | 0.887 | 0.000 |
| E04 | 0.818 | 0.818 | 0.833 | 1.000 | +0.167 |
| E05 | 0.400 | 0.400 | 0.833 | 0.833 | 0.000 |
| **Avg** | **0.716** | **0.716** | **0.911** | **0.944** | **+0.033** |

**Tại sao Recall dự kiến không đổi?**

> Recall không đổi vì reranking chỉ đổi thứ tự, không thêm/xóa chunk.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
