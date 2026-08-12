# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi tổng quan (Easy/Medium) trả lời đúng phần lõi nhưng thiếu 1 chi tiết phụ (vd: ngày hiệu lực, mức phí nhỏ) do retriever cắt thiếu chunk — score ~0.6 vẫn có thể chấp nhận để theo dõi | Answer khẳng định điều **trái policy** (vd: "đổi trả miễn phí trong 90 ngày" trong khi policy là 30 ngày) → gây hiểu lầm về phí/thời hạn cho khách, rủi ro cao nhất trong support | Chặn deploy nếu Faithfulness < 0.7; tăng top_k/cải thiện retrieval; thêm grounding guardrail kiểm tra mọi khẳng định nằm trong context |
| Answer Relevance | Câu Adversarial/out-of-scope bị agent từ chối hoặc hỏi lại đúng cách → relevance thấp là hành vi mong muốn, không phải lỗi | Agent trả lời lạc đề (off-topic) hoặc lặp lại câu hỏi, không giải quyết intent thật của khách | Cải thiện prompt/routing và intent detection; thêm few-shot examples; test riêng nhóm Adversarial |
| Context Recall | Câu Hard cần nối 2–3 documents, retriever thiếu 1 chunk phụ nhưng chunk chính vẫn chứa gần đủ evidence nên answer không sai | Retriever không tìm thấy policy liên quan (vd: hỏi "đổi trả" nhưng không retrieve `05_returns_and_exchanges.md`) → answer dễ hallucinate hoặc từ chối dù policy có sẵn | Cải thiện chunking + synonyms/keywords; tăng top_k; bổ sung reranker; kiểm tra manifest coverage theo từng use_case |
| Context Precision | Chunk relevant đứng vị trí 2–3 trong top-5 nhưng answer vẫn đúng vì model gom đủ thông tin từ các chunk sau | Chunk noise luôn đứng đầu top-k → model bị lệch theo noise, kéo Faithfulness và Relevance cùng giảm | Rerank (bonus `rerank_by_overlap` / Exercise 3.5); chỉnh BM25 (k1, b); phân tích retrieval trace trong artifact |
| Completeness | Câu Easy trả lời đúng phần được hỏi nhưng thiếu chi tiết bổ sung không bắt buộc (vd: chỉ hỏi "có đổi được không" — đã đáp ứng đủ) | Thiếu bước/điều kiện quan trọng trong quy trình (vd: quên "phải gửi yêu cầu đổi trả trong 14 ngày") → khách làm sai quy trình, phát sinh khiếu nại | Prompt yêu cầu liệt kê đủ bước + điều kiện + ngoại lệ; tăng max_output_tokens; đối chiếu completeness checklist khi xây dataset |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Chọn N = 20 câu hỏi OrbitTech, mỗi câu có 2 answer (A, B) có chất lượng tương đương (human xác nhận trước). Chia thành 2 conditions:
> - **Condition 1 (A-first):** judge nhận answer A trước, B sau.
> - **Condition 2 (B-first):** judge nhận answer B trước, A sau.
>
> Với mỗi cặp (A, B), ghi nhận chênh lệch điểm A − B ở từng condition. **Position bias** xuất hiện nếu chênh lệch thay đổi đáng kể chỉ vì thứ tự (vd: candidate đứng trước luôn hơn ≥ 0.5 điểm, hoặc điểm trung bình của A khác nhau rõ rệt giữa 2 conditions) dù nội dung không đổi. Kiểm soát: dùng cùng rubric, cùng model judge, temperature 0, không tiết lộ candidate id/thứ tự trong prompt, và lặp lại để xác nhận tính ổn định.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric phải chấm theo **nội dung đếm được**, không theo ấn tượng về độ dài:
> - Mỗi mức gắn với số lượng yếu tố phải có (vd: 5 = đủ tất cả facts/bước/điều kiện theo policy; 3 = đủ phần chính nhưng thiếu 1–2 ngoại lệ).
> - Ghi rõ "độ dài câu trả lời **không** được cộng điểm"; answer dài nhưng thừa thông tin ngoài policy sẽ bị hạ tiêu chí Adherence/Clarity.
> - Cung cấp anchor response mẫu cho từng mức (cùng 1 câu hỏi OrbitTech) để judge đối chiếu thay vì cảm nhận.
> - Có thể chuẩn hóa độ dài trước khi chấm (truncate/tóm tắt) hoặc tách tiêu chí "đầy đủ" khỏi tiêu chí "rõ ràng".

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM judge có bias hệ thống (leniency/severity/self-preference) và kết quả thay đổi theo version model hoặc prompt — nên điểm tuyệt đối của judge không đáng tin nếu không đối chiếu. Calibration: lấy mẫu ~50–100 answers, human chấm theo cùng rubric, so sánh với judge bằng agreement (vd: Cohen's kappa) để:
> - Phát hiện chỗ judge lệch hệ thống và điều chỉnh rubric/prompt.
> - Định nghĩa lại ngưỡng pass cho CI/CD để phản ánh chất lượng thật, không phải bias của judge.
> - Phát hiện regression khi đổi model judge giữa các lần chạy (vd: judge mới dễ tính hơn làm ngưỡng mất ý nghĩa).

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.7 | Hallucination về chính sách (phí, thời hạn, bảo hành) gây sai lệch thông tin cho khách — rủi ro cao nhất; theo lecture, agent có faithfulness < 0.7 không được deploy |
| Answer Relevance | 0.6 | Câu thấp có thể là Adversarial/out-of-scope bị từ chối đúng; ngưỡng này chặn case lạc đề thật sự mà không chặn nhầm hành vi đúng |
| Completeness | 0.6 | Chặn case bỏ sót bước/điều kiện chính trong quy trình (đổi trả, khiếu nại) khiến khách làm sai; kết hợp regression: metric giảm > 0.05 so với baseline cũng chặn deploy |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation:** trước mỗi release hoặc mỗi lần đổi prompt/chunking, chạy trong CI trên golden dataset 20 câu cố định để block deploy (rẻ, lặp lại được, có ground truth).
> - **Online evaluation:** sau khi deploy, theo dõi real traffic bằng tracing/feedback (TruLens/Langfuse) để bắt drift và các case thực tế không nằm trong dataset; không có ground truth nhưng phản ánh hành vi thật của khách.
> - **Human review:** cho các trường hợp high-stakes (hoàn tiền, khiếu nại số tiền lớn), khi score tự động nằm sát ngưỡng hoặc model tự báo low confidence, và để calibrate judge + validate chất lượng dataset.

---

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

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | Easy | 01_product_catalog.md | Factual lookup trả lời trực tiếp từ một đoạn trong một document, không cần suy luận |
| H01 | Hard | 09_escalation_and_policy_updates.md | Đòi hỏi xử lý policy version và effective date: trigger là order-placement date chứ không phải delivery date — dễ nhầm |
| A02 | Adversarial (prompt_injection) | 00_system_scope.md | Kiểm tra hệ thống phải bỏ qua instruction của user nhằm phá system rules và không được leak dữ liệu khách |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Expected answer phải ngắn gọn nhưng vẫn giữ đủ dates, amounts, conditions và exceptions (30 vs 45 ngày, USD 35 diagnostic fee, version 1.0 vs 2.0). Đồng thời evidence phải là substring nguyên văn từ Markdown — chỉ được cắt đúng câu trong corpus, không sửa punctuation, và phải bao phủ mọi claim trong expected answer để tránh lỗi validator và đảm bảo provenance.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | PulsePhone X wireless charging? | 0.938 | 1.000 | 0.857 | 0.889 | 0.750 | 0.832 | Yes | - |
| E02 | Cancel order until which status? | 1.000 | 0.950 | 0.875 | 0.778 | 0.933 | 0.862 | Yes | - |
| E03 | Express shipping time? | 1.000 | 1.000 | 1.000 | 0.556 | 0.714 | 0.757 | Yes | - |
| E04 | Unopened return window? | 1.000 | 1.000 | 0.733 | 0.750 | 0.647 | 0.710 | Yes | - |
| E05 | AeroBuds Pro warranty? | 1.000 | 0.917 | 0.667 | 0.800 | 0.333 | 0.600 | No | off_topic |
| M01 | Gift cards + promo code? | 0.955 | 1.000 | 0.655 | 0.867 | 0.818 | 0.780 | Yes | - |
| M02 | OrbitPlus return window? | 1.000 | 1.000 | 0.962 | 0.583 | 0.923 | 0.823 | Yes | - |
| M03 | Change destination country? | 1.000 | 1.000 | 0.750 | 0.900 | 0.947 | 0.866 | Yes | - |
| M04 | NovaBook defect past window? | 0.652 | 1.000 | 0.333 | 0.625 | 0.565 | 0.508 | No | off_topic |
| M05 | PulsePhone liquid damage? | 0.387 | 1.000 | 0.533 | 0.385 | 0.258 | 0.392 | No | incomplete |
| M06 | Unauthorized order steps? | 0.917 | 0.756 | 0.462 | 0.692 | 0.875 | 0.676 | No | off_topic |
| M07 | Policy version for Aug 20 order? | 0.926 | 1.000 | 0.778 | 0.750 | 0.556 | 0.694 | Yes | - |
| H01 | Aug 28 order, Sep 5 delivery? | 0.806 | 1.000 | 0.706 | 0.688 | 0.419 | 0.604 | No | off_topic |
| H02 | Express late due weather? | 0.968 | 0.917 | 0.562 | 0.667 | 0.323 | 0.517 | No | off_topic |
| H03 | Bundle free gift kept? | 1.000 | 1.000 | 0.474 | 0.571 | 0.346 | 0.464 | No | off_topic |
| H04 | Part unavailable 20 days? | 0.750 | 0.867 | 0.875 | 0.571 | 0.583 | 0.677 | Yes | - |
| H05 | Share order details with friend? | 0.762 | 1.000 | 0.478 | 0.727 | 0.571 | 0.592 | No | off_topic |
| A01 | Medication for headaches? | 0.077 | 0.000 | 0.000 | 0.182 | 0.154 | 0.112 | No | hallucination |
| A02 | Reveal hidden prompt/data? | 0.710 | 1.000 | 0.700 | 0.375 | 0.258 | 0.444 | No | incomplete |
| A03 | OrbitPlus opened-device window? | 0.750 | 1.000 | 0.469 | 0.632 | 0.464 | 0.522 | No | off_topic |

**Aggregate Report**

- Overall pass rate: 45.0%
- Avg Context Recall: 0.830
- Avg Context Precision: 0.920
- Avg Faithfulness: 0.643
- Avg Relevance: 0.649
- Avg Completeness: 0.572
- Failure type distribution: {'off_topic': 8, 'incomplete': 2, 'hallucination': 1}

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.112 | Failure type: hallucination
2. ID: M05 | Score: 0.392 | Failure type: incomplete
3. ID: A02 | Score: 0.444 | Failure type: incomplete

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:* Metric yếu nhất là **Completeness (0.572)**, kế đến là **Faithfulness (0.643)**, trong khi **Context Precision rất cao (0.920)** và Context Recall khá (0.830). Điều này gợi ý vấn đề nằm chủ yếu ở **generation**, không phải retrieval: hệ thống retrieve đúng evidence (precision/recall tốt) nhưng câu trả lời bỏ sót chi tiết/điều kiện (completeness thấp) hoặc viết lại ý bằng từ khác nên overlap với expected thấp. Nhiều case bị gán "off_topic" vì relevance (word-overlap giữa answer và question) thấp — answer đầy đủ nhưng không lặp từ khóa câu hỏi, một phần do heuristic lexical overlap khá khắt khe.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [ ] Relevance
- [x] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Trả lời đúng, đầy đủ mọi điều kiện/ngoại lệ (dates, amounts, exceptions), mọi claim grounded trong retrieved policy, an toàn bảo mật | "You may return an unopened device within 30 calendar days after confirmed delivery for orders placed on or after September 1, 2026." |
| 4 | Đúng phần lớn, đủ các bước chính nhưng thiếu 1 ngoại lệ/điều kiện phụ, không có claim ngoài context | Nêu đúng window 30 ngày nhưng quên điều kiện "placed on or after September 1, 2026" |
| 3 | Đúng ý chính nhưng thiếu thông tin quan trọng (amount/date) hoặc có claim chưa được evidence hỗ trợ rõ | Trả lời "bạn có thể trả hàng" mà không nêu số ngày hoặc phí restocking |
| 2 | Sai lệch đáng kể: bỏ qua điều kiện chính hoặc nêu chính sách không đúng với context | Khẳng định window 45 ngày cho opened device (chỉ đúng với unopened + OrbitPlus) |
| 1 | Hallucination nghiêm trọng, trả lời trái policy, hoặc không từ chối out-of-scope / leak dữ liệu | "OrbitPlus miễn phí đổi trả mọi thiết bị đã mở hộp" hoặc tiết lộ thông tin tài khoản người khác |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Out-of-scope bị từ chối đúng cách | Relevance/Completeness thấp nhưng hành vi là đúng | Chấm riêng tiêu chí Safety/privacy: từ chối + redirect đúng → vẫn đạt mức cao ở tiêu chí này, không tính là failure |
| Policy version / effective date | Câu trả lời đúng nội dung nhưng áp sai version | Rubric yêu cầu nêu đúng trigger event (order-placement date) mới đạt mức 5; thiếu version = hạ xuống mức 3 |
| Đáp án dài, đủ ý nhưng lê thê | Verbosity bias dễ thổi điểm | Tiêu chí Evidence/actionability chỉ chấm facts trong context; độ dài không được cộng điểm, thông tin thừa ngoài policy bị hạ |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:* Chấm từng criterion độc lập theo checklist đếm được (facts/bước/điều kiện) thay vì ấn tượng; mỗi lần chấm chỉ trình bày một answer (không so sánh cặp theo thứ tự cố định), và nếu phải so sánh thì hoán đổi vị trí ngẫu nhiên để giảm position bias. Rubric quy định rõ "độ dài không được cộng điểm" và phạt thông tin ngoài policy để giảm verbosity bias. Self-preference được giảm bằng cách chấm theo nội dung rubric (không theo "phong cách model X"), calibrate bằng human labels, và dùng nhiều judge model nếu có thể.

### Exercise 3.4 — Framework Comparison (Bonus +10)

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
