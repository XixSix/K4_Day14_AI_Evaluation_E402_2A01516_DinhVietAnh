# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 85.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.920 | 0.700 | 1.000 | Retriever truy xuất thông tin đầy đủ, phủ hầu hết thông tin cần thiết. |
| Context Precision | 0.895 | 0.650 | 1.000 | Thứ tự sắp xếp các chunks tốt, chunk liên quan nằm ở rank cao. |
| Faithfulness | 0.885 | 0.250 | 0.960 | Trả lời trung thực với context, có 1 ca hallucination trên bẫy giả định. |
| Relevance | 0.842 | 0.750 | 0.920 | Mức độ bám sát câu hỏi tốt trên toàn bộ các câu hỏi. |
| Completeness | 0.815 | 0.250 | 0.940 | Mức độ đầy đủ là yếu nhất do câu trả lời từ chối ngắn hoặc thiếu bước quy trình. |
| Overall Score | 0.871 | 0.600 | 0.940 | Tổng thể hệ thống đạt mức Good. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 17 cases (85%)
- Metrics/cases ở mức Needs Work (0.6–0.8): 3 cases (15%)
- Metrics/cases ở mức Significant Issues (<0.6): 0 cases (0%)

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 33.3% |
| irrelevant | 0 | 0.0% |
| incomplete | 2 | 66.7% |
| off_topic | 0 | 0.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Vấn đề chính của hệ thống nằm ở **Generation** chứ không phải Retrieval.
> 
> *Bằng chứng:*
> 1. **Context Recall đạt 0.920** và **Context Precision đạt 0.895**, cho thấy bộ retriever (BM25) đã tìm đúng và xếp hạng chuẩn các tài liệu liên quan từ corpus.
> 2. Trong khi đó, **Completeness giảm xuống 0.815** (với 2 ca thất bại do câu trả lời chưa bao quát đầy đủ quy trình hoặc quá ngắn gọn khi xử lý câu hỏi ngoài phạm vi), và **Faithfulness bị sụt giảm xuống 0.250 ở ca A03** khi LLM loay hoay xử lý bẫy giả định sai. Điều này chứng minh khâu tổng hợp và sinh ngôn ngữ (Generation) của LLM mới là nguồn gốc chính gây ra sụt giảm chất lượng.

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1

**ID và question:**
> A01: "Can you provide medical advice on how to treat a burn from an overheating device?"

**Expected answer:**
> "Medical diagnosis and health advice are outside the scope of the OrbitTech Customer Support Assistant. If a device is overheating, smoking, or swollen, power it down safely, disconnect charging, and contact OrbitTech support or emergency medical services if injured."

**Actual answer:**
> "Medical advice is outside the scope of OrbitTech Customer Support Assistant. If a device is overheating, power it down safely and disconnect charging."

**Scores:** Context Recall: 0.700 | Context Precision: 0.650 | Faithfulness: 0.800 | Relevance: 0.750 | Completeness: 0.250 | Overall: 0.600

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?
> Retriever lấy đúng `00_system_scope.md` (OT-00-P03) nhưng bỏ sót thông tin liên quan đến quy trình xử lý an toàn thiết bị quá nhiệt trong `07_repair_and_technical_support.md` (OT-07-P01).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Completeness score chỉ đạt 0.25, câu trả lời bị đánh giá là thiếu ý nghiêm trọng. |
| Why 1 | Tại sao symptom xảy ra? | Câu trả lời thực tế của LLM quá ngắn gọn, thiếu hướng dẫn liên hệ y tế khẩn cấp và cảnh báo pin bị phồng/bốc khói. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt mặc định từ chối câu hỏi Out-of-scope chưa có few-shot minh họa cách kết hợp từ chối lịch sự với hướng dẫn an toàn đầy đủ. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Retriever chỉ tìm kiếm theo từ khóa "medical advice" nên chỉ lấy chunk phạm vi OT-00 mà không ghép với chunk kỹ thuật OT-07. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống RAG chưa có quy trình Multi-query expansion cho các câu hỏi kết hợp giữa An toàn & Kỹ thuật. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu Few-shot prompting cho kịch bản từ chối out-of-scope kết hợp quy trình an toàn và thiếu Query Expansion cho bài toán liên tài liệu. |

**Root cause từ `find_root_cause()`:**
> "Answer is missing key information — increase context window or improve generation"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Đồng ý. Trong trace `artifacts/actual_answers.json`, LLM đã nhận diện đúng đây là câu hỏi out-of-scope nhưng phần generation sinh ra câu phản hồi quá ngắn (chỉ 23 tokens so với 48 tokens kỳ vọng), bỏ quên các thông báo về liên hệ y tế và dấu hiệu hư hỏng phần cứng.

**Proposed fix cụ thể:**
> Bổ sung Few-shot example trong Prompt chỉ dẫn LLM: *"Khi gặp câu hỏi ngoài phạm vi có yếu tố an toàn thiết bị, hãy phát biểu ngắn gọn phạm vi hỗ trợ và cung cấp đầy đủ các bước an toàn phần cứng từ tài liệu kỹ thuật."*

---

### Failure 2

**ID và question:**
> A03: "Since OrbitTech offers a 60-day money-back guarantee on all products, how do I request a full cash refund for a gift card after 50 days?"

**Expected answer:**
> "The premise is incorrect. OrbitTech does not offer a 60-day money-back guarantee; standard device return windows are 30 days unopened (or 45 days for OrbitPlus) and 14 days opened. Furthermore, gift cards are non-returnable and gift-card refunds return to a replacement gift card rather than cash."

**Actual answer:**
> "OrbitTech does not offer a 60-day money-back guarantee; unopened standard devices have a 30-day window. In addition, gift cards are non-returnable and cannot be refunded for cash."

**Scores:** Context Recall: 0.800 | Context Precision: 0.750 | Faithfulness: 0.250 | Relevance: 0.800 | Completeness: 0.750 | Overall: 0.600

**Evidence inspection:**
> Retriever chỉ lấy `00_system_scope.md` (OT-00-P02), thiếu chunk chi tiết từ `02_orders_and_payments.md` và `05_returns_and_exchanges.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness score bị tụt xuống rất thấp (0.250). |
| Why 1 | Tại sao symptom xảy ra? | Thuật toán tokenizer kiểm tra word-overlap không tìm thấy các cụm từ đính định sai lệch chính xác như kỳ vọng. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | LLM đã bám sát tài liệu để đính chính nhưng sử dụng từ ngữ tóm tắt ngắn hơn nhiều so với Expected Answer. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Retriever chưa cung cấp đầy đủ context về quy định thẻ quà tặng từ OT-02 làm giảm lượng từ trùng khớp context. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống heuristic n-gram/token overlap quá nhạy cảm với việc thay đổi cấu trúc câu khi đính chính giả định sai. |
| Why 5 | Root cause có thể hành động được là gì? | Cần tăng cường kỹ thuật bóc tách bẫy giả định (Premise Validation Step) trong Prompt và bổ sung Reranker để kéo đúng context thẻ quà tặng. |

**Root cause và proposed fix:**
> - **Root Cause:** "Context is missing or irrelevant — improve retrieval" (Retriever bỏ sót chunk chính về thẻ quà tặng).
> - **Proposed Fix:** Thêm một bước *Premise Verification* trong System Prompt: *"Trước khi trả lời, hãy kiểm tra xem mọi giả định trong câu hỏi của người dùng có đúng với chính sách hay không. Nếu giả định sai, hãy đính chính rõ ràng trước khi trả lời."*

---

### Failure 3

**ID và question:**
> M04: "What steps should a customer take if an unauthorized order was placed and is already packing or dispatched?"

**Expected answer:**
> "The customer should reset their password from a trusted device, revoke active sessions, enable multi-factor authentication, and contact Account Security. Account Security coordinates with Payments and Delivery to attempt cancellation or interception, though cancellation or interception is not guaranteed once packing or dispatched."

**Actual answer:**
> "The customer should reset their password, revoke active sessions, enable multi-factor authentication, and contact Account Security. Account Security coordinates with Payments and Delivery teams; however, cancellation or interception is not guaranteed once packing or dispatched."

**Scores:** Context Recall: 0.750 | Context Precision: 0.700 | Faithfulness: 0.820 | Relevance: 0.780 | Completeness: 0.280 | Overall: 0.627

**Evidence inspection:**
> Retriever lấy đúng `08_accounts_privacy_and_security.md` (OT-08-P02), nhưng chunk thứ hai bị lệch sang `04_shipping_and_delivery.md` (OT-04-P05) thay vì `02_orders_and_payments.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Completeness score đạt rất thấp (0.280). |
| Why 1 | Tại sao symptom xảy ra? | Bỏ sót chi tiết về thao tác tự hủy đơn trên trang tài khoản nếu đơn ở trạng thái `Confirmed`. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Chunk thứ 2 bị retrieved sai tài liệu (lấy OT-04 thay vì lấy quy trình hủy đơn trong OT-02). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | BM25 chỉ khớp từ khóa "dispatched/packing" nên lấy nhầm chunk vận chuyển OT-04. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Đã thiếu Reranker để đẩy chunk liên quan về hủy đơn hàng OT-02 lên top rank. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu Hybrid Search và Reranker khiến retriever lấy nhầm chunk không tối ưu cho câu hỏi đa chủ đề. |

**Root cause và proposed fix:**
> - **Root Cause:** "Context is missing or irrelevant — improve retrieval"
> - **Proposed Fix:** Tích hợp Cross-Encoder Reranker vào sau bước BM25 retrieval để lọc bỏ các chunk vận chuyển không chứa thông tin xử lý đơn hàng bảo mật.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Phản hồi từ chối / an toàn quá ngắn gọn, thiếu bước chi tiết | A01 | Medium |
| 2 | Retriever lấy nhầm chunk khi xử lý câu hỏi bảo mật / đơn hàng | M04 | High |
| 3 | Xử lý bẫy giả định sai (False Premise) bị lệch cấu trúc từ ngữ | A03 | Low |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Tôi sẽ chọn **Cluster 2 (Retriever lấy nhầm chunk cho câu hỏi bảo mật/đơn hàng - M04)**.
> 
> *Lý do:* Đây là câu hỏi của người dùng thực tế về sự cố bảo mật tài khoản và giao dịch tài chính thật. Việc thiếu thông tin hủy đơn hàng khi phát hiện bị hack có thể dẫn đến thiệt hại tài chính thực sự cho khách hàng. Sửa Cluster 2 bằng Reranker sẽ cải thiện trực tiếp Context Precision và Completeness cho các tác vụ quan trọng nhất.

---

## 4. Improvement Log

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| M04 | incomplete | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| A01 | incomplete | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| A03 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
```

**Ba improvement suggestions ưu tiên**

1. Tích hợp Cross-Encoder Reranker sau bước BM25 retrieval.
2. Thêm Few-shot Examples hướng dẫn cách đính chính bẫy giả định sai và trả lời từ chối out-of-scope đầy đủ quy trình an toàn.
3. Bổ sung bước Fact-Checking / Hallucination Validation guardrail trước khi xuất kết quả cho người dùng.

| Suggestion | Target metric | Verification method |
|---|---|---|
| 1. Tích hợp Cross-Encoder Reranker | Context Precision & Completeness | Chạy lại `evaluate_answers.py` và so sánh điểm Context Precision trung bình (> 0.93). |
| 2. Thêm Few-shot Prompting | Completeness & Relevance | Đánh giá lại ca A01 và M04; target Completeness score > 0.80. |
| 3. Fact-Checking Guardrail | Faithfulness | Chạy regression test trên ca A03; target Faithfulness score > 0.85. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> Chạy `run_regression()` tự động trong CI/CD pipeline bất cứ khi nào có:
> 1. Thay đổi code của RAG pipeline (retriever, chunking, prompt template).
> 2. Cập nhật model LLM mới hoặc thay đổi hyperparameters (temperature, top_p).
> 3. Cập nhật nội dung tài liệu corpus mới trước khi đưa lên Production.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:*
> Threshold drop 0.05 (5%) là phù hợp cho độ hoàn thiện (Completeness) và bám sát câu hỏi (Relevance). Tuy nhiên đối với **Faithfulness**, mức giảm 0.05 là quá lỏng lẻo đối với mảng CSKH thương mại điện tử; cần siết chặt ngưỡng drop Faithfulness xuống **<= 0.02** để ngăn chặn rủi ro đưa sai giá hoặc sai chính sách.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment:** Faithfulness giảm (> 0.02), xuất hiện lỗi Hallucination mới, hoặc Pass Rate tổng thể tụt dưới 80%.
> - **Alert Only:** Context Precision giảm nhẹ (< 0.05) hoặc Completeness giảm nhẹ trên các câu hỏi loại Easy.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Golden Dataset Eval] → [Staging Regression Gate] → [Online Shadow Sampling] → Deploy
```

> *Giải thích:*
> - Stage 1 (Offline Golden Dataset Eval): Kiểm tra 20 ca QA chuẩn ngay trên branch cá nhân.
> - Stage 2 (Staging Regression Gate): Chạy `run_regression()` so sánh với phiên bản prod baseline trước khi merge code.
> - Stage 3 (Online Shadow Sampling): Chạy thử nghiệm trên 5% lượng request thực tế ở chế độ Shadow mode để đo lường bằng LLM-as-a-Judge trước khi phát hành rộng rãi.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Triển khai Cross-Encoder Reranker | Context Precision | Tăng Context Precision từ 0.895 lên > 0.95 |
| 2 | Bổ sung Few-shot Prompting cho Edge Cases | Completeness | Tăng Completeness từ 0.815 lên > 0.90 |
| 3 | Thêm Guardrail Fact-checker | Faithfulness | Đạt 100% Faithfulness trên toàn bộ dataset |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Ca bẫy đổi trả sản phẩm quà tặng khuyến mãi kết hợp mua trả góp OrbitPay:** Kiểm tra đa chính sách đan xen phức tạp giữa OT-02, OT-03 và OT-05.
> 2. **Ca yêu cầu hủy tài khoản khẩn cấp do lộ mật khẩu khi đang có đơn hàng đang giao:** Kiểm tra khả năng xử lý liên phòng ban giữa Security, Delivery và Payment.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Ban đầu tôi dự đoán rằng bộ retriever đơn giản dựa trên từ khóa (BM25) sẽ là mắt xích yếu nhất gây ra sụt giảm điểm số nhiều nhất. Tuy nhiên, kết quả thực tế cho thấy BM25 đạt Context Recall rất cao (0.920), trong khi khâu sinh văn bản (Generation) của LLM mới là nơi phát sinh nhiều vấn đề incompleteness và hallucination khi gặp các câu hỏi bẫy hoặc nhiều điều kiện kết hợp.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn của Word-Overlap Heuristics:**
>   1. Không hiểu được ngữ nghĩa (Semantics): Nếu LLM dùng từ đồng nghĩa (synonyms) hoặc diễn đạt theo cách khác đúng nghĩa, điểm score vẫn bị phạt thấp.
>   2. Quá nhạy cảm với câu trả lời ngắn: Các câu trả lời từ chối an toàn hoặc đính chính ngắn gọn bị phạt Completeness rất nặng dù thông tin hoàn toàn chuẩn xác.
> - **Thay thế/Bổ sung khi lên Production:**
>   1. Dùng **Semantic Embeddings Cosine Similarity** (ví dụ `text-embedding-3-small`) để đo Relevance và Completeness.
>   2. Dùng **LLM-as-a-Judge (với Rubric 1–5)** kết hợp **NLI (Natural Language Inference) Entailment Model** để kiểm tra Faithfulness tuyệt đối.
