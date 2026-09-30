# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 5.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.619 | 0.000 | 1.000 | Coverage trung bình; M02 thấp nhất. |
| Context Precision | 0.857 | 0.000 | 1.000 | Ranking khá tốt nhưng còn noise. |
| Faithfulness | 0.152 | 0.000 | 0.667 | Yếu nhất; generator chưa bám evidence. |
| Relevance | 0.485 | 0.000 | 1.000 | Một số câu trả lời lệch intent. |
| Completeness | 0.428 | 0.000 | 0.938 | Thiếu điều kiện/chi tiết policy. |
| Overall Score | 0.355 | 0.000 | 0.757 | Chỉ E01 đạt pass. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision của đa số case; E01 có overall 0.757 nhưng chưa đạt Good theo ngưỡng 0.8.
- Metrics/cases ở mức Needs Work (0.6–0.8): Context Recall trung bình 0.619; một số faithfulness/relevance riêng lẻ đạt 0.6–0.8.
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness 0.152, Relevance 0.485, Completeness 0.428 và 19/20 overall cases.

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 17 | 85.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 2 | 10.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> Vấn đề chính nằm ở generation/grounding: Context Precision trung bình 0.857 nhưng Faithfulness chỉ 0.152. M02 có Context Recall và Precision đều 0.000, cho thấy retrieval/query mismatch cũng là một nhánh nguyên nhân.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> E05 — What is the product catalog used for?

**Expected answer:**

> It provides official product and compatibility information.

**Actual answer:**

> Insufficient evidence.

**Scores:** Context Recall: 0.400 | Context Precision: 0.833 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Retriever lấy chunk 01_product_catalog.md nhưng chunk nói về HomeHub compatibility, không nói mục đích catalog.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Model từ chối dù có context cùng domain. |
| Why 1 | Tại sao symptom xảy ra? | Chunk không chứa câu trả lời meta về mục đích catalog. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | BM25 không match tốt query “used for”. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có query expansion cho intent meta. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Không có coverage check trước refusal. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu intent routing và regression case cho câu hỏi meta. |

**Root cause từ `find_root_cause()`:**

> Query intent/coverage chưa được xử lý; bổ sung query rewrite và coverage guard.

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Đồng ý. Trace cho thấy Retriever lấy chunk 01_product_catalog.md nhưng chunk nói về HomeHub compatibility, không nói mục đích catalog. Root cause phù hợp với điểm thấp nhất và cần fix ở query intent/coverage chưa được xử lý; bổ sung query rewrite và coverage guard..

**Proposed fix cụ thể:**

> Query intent/coverage chưa được xử lý; bổ sung query rewrite và coverage guard.

### Failure 2

**ID và question:**

> M02 — What payment topics are covered?

**Expected answer:**

> Payments, cancellations, and installments.

**Actual answer:**

> Insufficient evidence in the retrieved contexts to answer the question.

**Scores:** Context Recall: 0.000 | Context Precision: 0.000 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:**

> Top chunk là 00_system_scope.md thay vì 02_orders_and_payments.md.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Retriever lấy sai tài liệu và generator từ chối. |
| Why 1 | Tại sao symptom xảy ra? | Từ “topics/covered” gây lexical mismatch. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | BM25 không hiểu quan hệ payment–orders/installments. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có synonym/domain alias. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có regression test cho câu hỏi tổng quan payment. |
| Why 5 | Root cause có thể hành động được là gì? | Retriever chưa có taxonomy-aware routing. |

**Root cause và proposed fix:**

> Đồng ý. Trace cho thấy Top chunk là 00_system_scope.md thay vì 02_orders_and_payments.md. Root cause phù hợp với điểm thấp nhất và cần fix ở lexical mismatch trong retrieval; thêm synonym expansion và category routing..

### Failure 3

**ID và question:**

> A02 — Will you guarantee my discount?

**Expected answer:**

> Không hứa discount; chỉ nêu policy nếu có evidence.

**Actual answer:**

> Model nêu chính sách 5% OrbitPlus nhưng không xử lý đúng chữ “guarantee”.

**Scores:** Context Recall: 0.200 | Context Precision: 1.000 | Faithfulness: 0.098 | Relevance: 0.200 | Completeness: 0.200 | Overall: 0.166

**Evidence inspection:**

> Retrieved chunk 03_promotions_and_membership.md là evidence hợp lệ nhưng response biến false premise thành claim discount.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời có policy thật nhưng bỏ qua limitation về guarantee. |
| Why 1 | Tại sao symptom xảy ra? | Prompt không buộc xử lý false premise trước policy. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Không có intent classifier cho “guarantee/promise”. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Safety rule chưa thành answer template. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Regression set thiếu câu hỏi hứa hẹn/ngoại lệ. |
| Why 5 | Root cause có thể hành động được là gì? | Policy guardrail chưa được ưu tiên trong generation. |

**Root cause và proposed fix:**

> Đồng ý. Trace cho thấy Retrieved chunk 03_promotions_and_membership.md là evidence hợp lệ nhưng response biến false premise thành claim discount. Root cause phù hợp với điểm thấp nhất và cần fix ở thiếu guardrail cho promise/false premise; trả limitation trước rồi mới nêu policy..

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Generation grounding và policy guardrail yếu | E02, E03, E04, E05, A01, A02, A03 | High |
| 2 | BM25 lexical mismatch và thiếu query expansion | M02, M03, M04, M07 | High |
| 3 | Thiếu điều kiện/coverage trong answer | M01, M05, M06, H01, H03, H05 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> Chọn Cluster 1 vì Faithfulness là metric thấp nhất (0.152) và lỗi grounding ảnh hưởng trực tiếp tới policy/safety.

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|---|---|---|---|---|
| F001 | hallucination | Generation không grounded | Thêm grounding guardrail | Open |
| F002 | hallucination | BM25 lexical mismatch | Query expansion và category routing | Open |
| F003 | off_topic | Thiếu policy guardrail | Template xử lý limitation/privacy | Open |
```

**Ba improvement suggestions ưu tiên**

1. Thêm grounding guardrail và policy templates.
2. Thêm synonym/query routing cho retrieval.
3. Thêm answer checklist và regression cases.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Grounding guardrail | Faithfulness | Chạy lại benchmark và kiểm tra hallucination count |
| Query expansion | Context Recall | So sánh nhóm payment/product trước và sau |
| Answer checklist | Completeness | Review policy cases và score completeness |

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy sau mọi code/prompt/retrieval/model change và trong CI trước deploy.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> Có, đây là quality gate ban đầu hợp lý để phát hiện thay đổi đáng kể; khi dataset lớn hơn nên bổ sung confidence interval và significance checks.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> Faithfulness <0.70, completeness <0.60, safety/privacy failure và regression >0.05 phải block. Retrieval giảm nhẹ chỉ alert nếu answer quality vẫn ổn.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → unit tests → golden benchmark → failure analysis → Deploy
```

> Unit tests bảo vệ evaluation core; golden benchmark phát hiện regression trên dữ liệu cố định; failure analysis quyết định action trước deploy.

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thêm grounding guardrail | Faithfulness | Giảm hallucination và policy errors |
| 2 | BM25 synonym/query expansion | Context Recall | Tăng evidence đúng tài liệu |
| 3 | Answer checklist và regression cases | Completeness | Giảm thiếu điều kiện/ngoại lệ |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> Giữ E05, M02, A02; bổ sung thêm case policy date, privacy và false-premise để kiểm tra guardrail.

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Context Precision đạt 0.857 khá cao nhưng Faithfulness chỉ 0.152. Điều này cho thấy retrieve được chunk phù hợp chưa đảm bảo model sử dụng đúng evidence.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> Heuristic không hiểu phủ định, synonym, paraphrase hoặc mức độ an toàn. Production nên bổ sung citation verification, entailment/NLI, policy-rule checks, human review cho safety và online feedback.
