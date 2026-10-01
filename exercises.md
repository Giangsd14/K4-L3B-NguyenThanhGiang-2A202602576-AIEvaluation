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
| Faithfulness | Khi câu trả lời chứa các câu chào hỏi, lời cảm ơn xã giao ("Chào bạn, OrbitTech rất vui được hỗ trợ") mà các từ này không nằm trong technical context. | Khi câu trả lời bịa đặt số liệu sai lệch về chính sách bảo hành, ngày hiệu lực, giá cả hoặc phí đổi trả (hallucination). | Bổ sung negative constraints vào system prompt: "Chỉ trả lời dựa trên context, nếu không có thông tin hãy từ chối"; giảm temperature về 0.0. |
| Answer Relevance | Khi câu hỏi của người dùng quá ngắn hoặc mơ hồ, bot trả lời tổng quan và đặt câu hỏi làm rõ (clarifying question). | Khi người dùng hỏi về phí ship/giao hàng nhưng bot trả lời về quy trình bảo hành linh kiện, lạc đề hoàn toàn so với user intent. | Tối ưu hóa prompt generation yêu cầu trả lời trực diện intent; thêm bước intent classification / query rewriting trước khi sinh câu trả lời. |
| Context Recall | Khi câu hỏi đơn giản chỉ cần 1 factual claim đơn lẻ, trong khi gold context chứa nhiều chi tiết ngữ cảnh rườm rà không bắt buộc. | Khi câu hỏi yêu cầu các trường hợp ngoại lệ từ chối bảo hành (rơi vỡ, ngấm nước) nhưng retriever bỏ sót toàn bộ các chunks chứa điều khoản loại trừ. | Mở rộng top-k retrieved chunks; chuyển sang semantic chunking có overlap; áp dụng hybrid search (kết hợp BM25 với dense embeddings) hoặc Multi-Query. |
| Context Precision | Khi chunk liên quan nhất bị xếp ở vị trí rank 2 hoặc 3 nhưng generator vẫn đủ mạnh để tổng hợp câu trả lời chính xác mà không bị nhiễu. | Khi các chunks liên quan bị đẩy xuống cuối (rank 4, 5) hoặc danh sách chứa toàn distractor chunks khiến generator bị "lost in the middle" và trả lời sai. | Tích hợp reranker model (cross-encoder/BGE-reranker) sau retrieval; điều chỉnh trọng số BM25 và độ dài chunk để phạt chunks chứa từ khóa rác. |
| Completeness | Khi câu hỏi quá rộng, câu trả lời mang tính tóm tắt cô đọng các ý chính yếu theo yêu cầu người dùng thay vì dàn trải chi tiết phụ. | Khi câu hỏi yêu cầu quy trình 4 bước đổi trả hàng lỗi nhưng bot chỉ nêu được bước 1 và bỏ quên hoàn toàn 3 bước cùng điều kiện hóa đơn đi kèm. | Cải thiện prompt yêu cầu trả lời dạng bullet points có cấu trúc; sử dụng self-reflection / chain-of-thought để kiểm tra đủ điều kiện trước khi output. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Condition 1 (Forward order):** Gửi cho Judge LLM đánh giá cặp câu trả lời với thứ tự `[Answer A, Answer B]` cho cùng một câu hỏi và cùng một rubric. Ghi nhận điểm số hoặc quyết định thắng/thua ($Score_{A1}, Score_{B1}$).
> - **Condition 2 (Reverse / Swapped order):** Đảo ngược vị trí thành `[Answer B, Answer A]` và gửi cho Judge LLM với prompt, câu hỏi và rubric giống hệt Condition 1. Ghi nhận điểm số ($Score_{B2}, Score_{A2}$).
> - **Đo lường & Kết luận:** Tính tỷ lệ ưu tiên vị trí thứ nhất (Position 1 Win Rate). Nếu vị trí xuất hiện đầu tiên luôn nhận điểm cao hơn có ý nghĩa thống kê ($Score_{first} - Score_{second} > \Delta$), hệ thống có Position Bias. Biện pháp xử lý là chạy song song cả 2 conditions và lấy điểm trung bình (swap-and-average protocol).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Tách biệt tiêu chí:** Tách bạch giữa tiêu chí *Completeness* (tính đầy đủ của factual information) và tiêu chí *Conciseness / Precision* (tính cô đọng, súc tích).
> 2. **Chấm điểm theo Factual Checklist:** Yêu cầu Judge chấm điểm dựa trên số lượng key factual claims được thỏa mãn, tuyệt đối không tính điểm theo độ dài đoạn văn hoặc phong cách hoa mỹ.
> 3. **Phạt điểm câu trả lời lan man:** Bổ sung quy định trừ điểm rõ ràng trong rubric: *"Phạt 1-2 điểm nếu câu trả lời chứa thông tin thừa thãi, lặp lại câu hỏi hoặc không phục vụ trực tiếp cho việc giải quyết vấn đề của khách hàng"*.
> 4. **Giới hạn độ dài tham chiếu:** Định nghĩa rõ ngưỡng độ dài tối ưu cho từng mức điểm (ví dụ: điểm 5 cần đạt đủ ý trong 50–120 từ; câu trả lời dài trên 200 từ mà không thêm fact mới sẽ bị giới hạn tối đa điểm 3).

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. **Đo lường độ tin cậy:** LLM Judge là mô hình xác suất và có các bias nội tại (như self-preference đối với văn phong của chính nó). Calibrate với human labels là cách duy nhất để đo lường độ tương quan (Inter-Annotator Agreement qua Cohen's Kappa hoặc Spearman correlation) giữa Judge và con người.
> 2. **Điều chỉnh ngưỡng và Few-shot Calibration:** Giúp phát hiện sai lệch hệ thống (systematic bias - ví dụ: model luôn chấm quá dễ tính hoặc quá khắt khe) để bổ sung các ví dụ few-shot chuẩn vào prompt rubric nhằm kéo điểm của LLM Judge về khớp với chuẩn mực của chuyên gia domain.
> 3. **Xác định ranh giới tự động hóa:** Giúp đội ngũ xác định ngưỡng confidence nào LLM Judge có thể tự động phê duyệt, và ngưỡng nào cần đưa vào luồng Human-in-the-loop để chuyên gia đánh giá thủ công.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Trong domain hỗ trợ khách hàng và chính sách OrbitTech, ảo giác (hallucination) về giá, bảo hành hay đổi trả có thể gây thiệt hại tài chính và tranh chấp pháp lý nghiêm trọng. Ngưỡng 0.85 đảm bảo câu trả lời được neo chặt (grounded) vào tài liệu nguồn. |
| Answer Relevance | 0.80 | Đảm bảo câu trả lời đi thẳng vào vấn đề khách hàng cần hỏi, không trả lời vòng vo hoặc lạc đề làm giảm mức độ hài lòng của khách hàng (CSAT). Cho phép dung sai 0.20 cho các câu chào hỏi, lời chúc dịch vụ. |
| Completeness | 0.75 | Đảm bảo khách hàng nhận được đầy đủ các bước thực hiện và điều kiện cốt lõi của chính sách. Ngưỡng 0.75 cho phép linh hoạt đối với các câu hỏi mở mà câu trả lời mang tính định hướng tóm tắt. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment / CI/CD Quality Gate):** Chạy tự động trên bộ Golden Dataset (20 QA chuẩn) mỗi khi có pull request hoặc thay đổi về prompt, chunking, retriever, model weights. Dùng để chặn deploy nếu có hồi quy (regression drop > 0.05). Chi phí thấp, lặp lại nhanh, an toàn trước khi ra production.
> - **Online Evaluation (Post-deployment / Production Monitoring):** Chạy liên tục hoặc lấy mẫu trên traffic thực tế của khách hàng. Theo dõi các chỉ số runtime (latency P95, token cost, retrieval failure rate, thumbs up/down feedback, intent drift). Giúp phát hiện sớm các sự cố phát sinh ngoài tập test chuẩn trong môi trường thực.
> - **Human Review (Auditing & Calibration):** Thực hiện định kỳ hàng tuần/tháng bởi chuyên gia domain hoặc khi hệ thống phát hiện cảnh báo: duyệt các case bot từ chối trả lời, case có CSAT thấp, case LLM Judge không chắc chắn, và cập nhật mở rộng bộ Golden Dataset theo các edge cases mới từ thực tế.

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
