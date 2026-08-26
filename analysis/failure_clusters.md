# Failure Cluster Analysis — Phase A

**Sinh viên:** Do The Viet  
**Ngày:** 2026-08-26

---

## 1. Aggregate RAGAS Scores theo Distribution

| Metric | factual | multi_hop | adversarial |
|---|---|---|---|
| faithfulness | 0.82 | 0.64 | 0.71 |
| answer_relevancy | 0.79 | 0.61 | 0.68 |
| context_precision | 0.75 | 0.58 | 0.62 |
| context_recall | 0.71 | 0.48 | 0.55 |
| **avg_score** | **0.77** | **0.58** | **0.64** |

---

## 2. Bottom 10 Questions

| Rank | Distribution | Question | avg_score | worst_metric |
|---|---|---|---|---|
| 1 | multi_hop | Lương thử việc của nhân viên được tính như thế nào khi kết hợp với phụ cấp? | 0.31 | context_recall |
| 2 | adversarial | Theo policy v2023, nhân viên có được phép nghỉ bao nhiêu ngày? | 0.35 | faithfulness |
| 3 | multi_hop | Tính tổng phí phạt tạm ứng nếu nhân viên vay 10 triệu và hoàn trả muộn 30 ngày? | 0.38 | context_recall |
| 4 | adversarial | Có nên tự xử lý vi phạm kỷ luật mà không báo HR không? | 0.41 | faithfulness |
| 5 | multi_hop | Nhân viên A làm việc 5 năm được tích lũy bao nhiêu ngày phép? | 0.43 | context_recall |
| 6 | adversarial | Chính sách VPN cá nhân có được phép không theo quy định cũ? | 0.44 | context_precision |
| 7 | multi_hop | Kết hợp policy nghỉ thai sản và nghỉ phép thường, tổng có thể nghỉ bao nhiêu? | 0.46 | context_recall |
| 8 | multi_hop | Phí overtime của nhân viên part-time tính theo công thức nào? | 0.48 | answer_relevancy |
| 9 | adversarial | Nếu policy cũ và mới mâu thuẫn, nên áp dụng cái nào? | 0.49 | faithfulness |
| 10 | multi_hop | Tổng chi phí bảo hiểm employer phải đóng cho nhân viên 1 tháng? | 0.51 | context_recall |

---

## 3. Failure Cluster Matrix

*(Mỗi ô = số câu có worst_metric = row, thuộc distribution = col)*

| worst_metric | factual | multi_hop | adversarial | Total |
|---|---|---|---|---|
| faithfulness | 2 | 3 | 5 | 10 |
| answer_relevancy | 3 | 4 | 2 | 9 |
| context_precision | 4 | 5 | 2 | 11 |
| context_recall | 11 | 8 | 1 | 20 |

---

## 4. Dominant Failure Analysis

**Dominant distribution:** multi_hop  
**Dominant metric:** context_recall

**Lý do phân tích:**

> Distribution `multi_hop` có nhiều failures nhất vì các câu hỏi yêu cầu pipeline phải kết hợp thông tin từ nhiều tài liệu khác nhau (cross-doc reasoning). Corpus HR policy tiếng Việt có nhiều phiên bản (v2023, v2024) và nhiều tài liệu liên quan — pipeline khó biết cần lấy tất cả chunks liên quan. Metric `context_recall` thấp nhất vì BM25 + dense search thường bỏ sót các chunks phụ trợ cần thiết cho multi-hop reasoning. Đặc biệt, các câu hỏi tính toán (lương, phí phạt, ngày phép tích lũy) cần nhiều số liệu từ nhiều nơi mà retrieval hiện tại không đủ khả năng gom lại.

---

## 5. Suggested Fixes

| Metric yếu | Root cause | Suggested fix |
|---|---|---|
| faithfulness | LLM hallucinating khi không tìm thấy thông tin chính xác | Tighten system prompt: "Chỉ trả lời dựa trên context được cung cấp, không suy đoán"; giảm temperature xuống 0.1 |
| context_recall | Missing relevant chunks, đặc biệt với multi-hop | Thêm BM25 hybrid search; tăng top-K retrieval từ 20 → 40; thêm query expansion/decomposition cho multi-hop |
| context_precision | Too many irrelevant chunks vào context | Tăng reranking threshold; dùng cross-encoder reranker thay vì bi-encoder; lọc theo metadata (policy version) |
| answer_relevancy | Answer drift khỏi câu hỏi gốc | Cải thiện prompt template: thêm câu hỏi gốc vào cuối prompt; dùng chain-of-thought để answer focused hơn |

---

## 6. Nhận xét về Adversarial Distribution

> Adversarial distribution có avg_score = 0.64, thấp hơn factual (0.77) nhưng cao hơn multi_hop (0.58). Pipeline không bị "nhầm" hoàn toàn bởi version conflicts (v2023 vs v2024) nhưng có 5 câu rơi vào failure cluster `faithfulness` — LLM trả lời dựa trên policy cũ thay vì policy hiện hành. Trong bottom 10, có 3 câu adversarial (rank 2, 4, 9): đều liên quan đến trả lời câu hỏi "có nên làm X không?" dạng negation trap, hoặc hỏi về policy cũ. Pipeline cần được cải thiện để phát hiện và ưu tiên tài liệu có version mới nhất (metadata filtering theo ngày tạo tài liệu).
