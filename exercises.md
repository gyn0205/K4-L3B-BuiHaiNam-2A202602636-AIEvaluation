# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi out-of-scope/adversarial (VD: hỏi tư vấn đầu tư): câu trả lời đúng là từ chối + giới thiệu vai trò, nên dùng ít từ trong context; hoặc câu trả lời diễn đạt lại (paraphrase) khiến word-overlap thấp dù ý vẫn đúng. | Câu trả lời chứa claim không có trong corpus: bịa thông số sản phẩm, trạng thái giao hàng, mức giảm giá, thời hạn bảo hành/đổi trả hoặc quyền lợi pháp lý. | Đọc thủ công từng claim so với evidence; bổ sung grounding instruction ("chỉ trả lời từ tài liệu"), bắt buộc trích dẫn, thêm hallucination checker. Block deploy nếu xảy ra ở câu hỏi chính sách. |
| Answer Relevance | Câu hỏi ngắn/mơ hồ, câu trả lời đúng nhưng dùng từ đồng nghĩa; câu từ chối hợp lệ cho prompt injection (không lặp lại từ khóa của câu hỏi). | Trả lời đúng chủ đề khác (hỏi đổi trả nhưng trả lời bảo hành), hoặc từ chối một câu hỏi in-scope do guardrail quá chặt. | Kiểm tra intent detection và system prompt; xem retriever có kéo nhầm tài liệu không; thêm few-shot cho các intent dễ nhầm. |
| Context Recall | Câu adversarial/out-of-scope mà corpus vốn không có đáp án; expected answer chứa câu chuyển hướng ("liên hệ support") không nằm nguyên văn trong chunk. | Câu hỏi in-scope nhưng retriever bỏ sót tài liệu chứa evidence (VD: câu multi-hop cần cả `05_returns` và `09_policy_updates` nhưng chỉ lấy được một). | Tăng top-k, sửa chunking (chunk quá nhỏ cắt mất điều kiện), query rewriting/hybrid search (BM25 + embedding), kiểm tra metadata filter. |
| Context Precision | Top-k lớn cho câu multi-hop nên có thêm vài chunk phụ; chunk liên quan vẫn nằm ở đầu danh sách. | Chunk nhiễu xếp trên chunk chứa evidence, hoặc gần như toàn bộ chunk không liên quan → generator dễ trả lời dựa trên nhiễu. | Thêm reranker (cross-encoder hoặc `rerank_by_overlap`), giảm top-k, cải thiện embedding/query; so sánh precision trước và sau rerank. |
| Completeness | Expected answer dài, có thông tin phụ không bắt buộc; câu trả lời ngắn gọn nhưng đủ ý chính; paraphrase làm overlap thấp. | Bỏ sót điều kiện/ngoại lệ quan trọng (VD: thiếu điều kiện "còn seal" khi đổi trả, thiếu bước escalate khi thiết bị phồng pin) khiến khách hiểu sai chính sách. | Kiểm tra trước Context Recall: recall thấp → sửa retrieval; recall cao mà completeness thấp → sửa prompt generation (few-shot câu trả lời đầy đủ, checklist điều kiện), tăng context window. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy khoảng 30 cặp (Answer A, Answer B) cho cùng câu hỏi, trong đó có cả cặp chất lượng chênh lệch rõ và cặp gần bằng nhau (đã có human label). Cho judge chấm pairwise trong hai conditions:
> - **Condition 1 (A-first):** prompt đặt A ở vị trí 1, B ở vị trí 2.
> - **Condition 2 (B-first):** cùng cặp đó nhưng đảo thứ tự, B ở vị trí 1.
> - (Tuỳ chọn) **Condition 3 (control):** cặp hai answer giống hệt nhau. Judge không bias phải chọn tie hoặc 50/50.
>
> Đo **consistency rate** (tỷ lệ cặp mà judge chọn cùng một answer ở cả hai thứ tự) và **first-position win rate**. Nếu không có bias, win rate của vị trí 1 xấp xỉ 50% và consistency cao. Nếu win rate của vị trí 1 lớn hơn rõ rệt 50% (VD trên 60%, kiểm định binomial/McNemar p < 0.05), hoặc nhiều cặp "lật" kết quả khi đảo thứ tự, thì có position bias. Khi chạy thật nên luôn chấm cả hai thứ tự và chỉ chấp nhận kết quả nhất quán, còn lại coi là tie.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Viết tiêu chí theo **nội dung đúng và đủ**, không theo độ dài: liệt kê các ý bắt buộc (key facts từ expected answer) và chấm theo số ý đạt.
> - Thêm tiêu chí phạt rõ ràng: "Thông tin thừa, lặp lại hoặc không có trong tài liệu sẽ bị trừ điểm"; câu trả lời có claim không có evidence thì tối đa 2 điểm, dù dài.
> - Ghi rõ trong rubric: "Câu trả lời ngắn nhưng đủ ý được 5 điểm; độ dài không phải tiêu chí."
> - Cho anchor examples: một câu ngắn được 5 điểm và một câu dài nhiều chữ chỉ được 2 điểm.
> - Tách thành nhiều dimension (correctness, completeness, conciseness) và chấm riêng từng dimension, tránh một điểm tổng chung chung dễ bị ảnh hưởng bởi độ dài.
> - Kiểm tra lại bằng cách tính tương quan giữa độ dài answer và điểm judge. Tương quan cao là dấu hiệu bias còn tồn tại.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM judge cũng là một model và có thể sai một cách có hệ thống: bias vị trí/độ dài/self-preference, chấm quá dễ (leniency) hoặc quá khắt khe, hiểu rubric khác với ý của người viết, không biết chính sách riêng của OrbitTech. Nếu không calibrate, ta không biết điểm 4/5 của judge có tương ứng với "câu trả lời tốt" theo chuyên gia hay không, nên quality gate có thể block nhầm hoặc cho qua câu trả lời sai. Calibrate bằng cách cho người (domain expert) chấm một tập mẫu (khoảng 50–100 câu), so với judge bằng agreement / Cohen's kappa / Spearman correlation. Sau đó chỉnh rubric, prompt hoặc ngưỡng cho đến khi đủ đồng thuận (VD kappa ≥ 0.6), rồi lặp lại định kỳ khi đổi model judge, đổi rubric hoặc đổi domain.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Theo bài giảng: agent có faithfulness < 0.7 thì không được deploy. Trong customer support, hallucination về chính sách đổi trả, bảo hành, giá, giảm giá gây thiệt hại trực tiếp cho khách và cửa hàng, nên đây là ngưỡng khắt khe nhất. Ngoài ngưỡng trung bình, nên block nếu có bất kỳ case hallucination nào ở nhóm câu hỏi chính sách. |
| Answer Relevance | 0.60 | Metric word-overlap cho điểm thấp với câu từ chối hợp lệ hoặc câu paraphrase, nên ngưỡng thấp hơn để tránh block nhầm. Vẫn đủ để phát hiện trả lời lạc đề hoặc từ chối sai một cách hàng loạt. |
| Completeness | 0.60 | Thiếu điều kiện/ngoại lệ gây hiểu sai nhưng ít nguy hiểm bằng bịa thông tin. Expected answer thường dài hơn câu trả lời thực tế nên overlap tự nhiên thấp. 0.6 là ranh giới "Needs work" của bài giảng. Kết hợp thêm regression gate: block nếu bất kỳ metric nào giảm > 0.05 so với baseline. |

*Lưu ý:* các ngưỡng trên là đề xuất cho quality gate. Pass rule trong code vẫn giữ nguyên quy định của starter (cả ba score ≥ 0.5) và `overall_score()` là trung bình ba answer metrics.

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation:** chạy trên golden dataset cố định trước khi deploy. Dùng mỗi khi đổi code, prompt, model, chunking hoặc retriever; tích hợp vào CI/CD làm quality gate và regression test (so với baseline). Ưu điểm: rẻ, lặp lại được, so sánh được giữa các phiên bản. Nhược điểm: chỉ phủ các case đã biết.
> - **Online evaluation:** chạy trên traffic thật sau khi deploy, gồm monitoring metric (faithfulness tự động, tỷ lệ từ chối, tỷ lệ escalate, latency), feedback của user (thumbs up/down, CSAT) và A/B test giữa hai phiên bản. Dùng để phát hiện drift, câu hỏi mới chưa có trong golden set, hoặc khi chính sách/corpus thay đổi.
> - **Human review:** dùng khi cần độ tin cậy cao hoặc khi metric tự động không đủ: (1) xây dựng và review golden dataset; (2) calibrate LLM judge; (3) xem các case bị gắn cờ (điểm thấp, judge và metric bất đồng, user báo sai); (4) các chủ đề rủi ro cao như bảo mật tài khoản, thanh toán gian lận, an toàn pin/thiết bị; (5) trước các release lớn hoặc demo. Kết quả human review được đưa ngược vào golden dataset (vòng Evaluate → Analyze → Improve → Augment).

---

## Part 2 — Core Coding (9:45–10:40)

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

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

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

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

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
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
