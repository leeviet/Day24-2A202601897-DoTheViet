# LLM Judge Bias Report — Phase B

**Sinh viên:** Do The Viet  
**Ngày:** 2026-08-26  
**Judge model:** gpt-4o-mini

---

## 1. Pairwise Judge Results

*(Chạy pairwise_judge() trên 5 cặp answers từ bộ test)*

| # | Question (tóm tắt) | Winner | Reasoning tóm tắt |
|---|---|---|---|
| 1 | Nhân viên được nghỉ bao nhiêu ngày phép năm? | A | A trích dẫn đúng policy v2024 (15 ngày), B dùng số cũ (12 ngày) |
| 2 | Quy trình xin nghỉ phép khẩn cấp? | A | A nêu đủ 3 bước: thông báo, form, phê duyệt; B bỏ qua bước phê duyệt |
| 3 | Chính sách làm thêm giờ? | tie | Cả hai đều đúng nhưng diễn đạt khác nhau, không có winner rõ ràng |
| 4 | Trợ cấp thai sản bao nhiêu tháng? | B | B trả lời chính xác 6 tháng; A sai 4 tháng |
| 5 | Điều kiện để xét thưởng cuối năm? | A | A liệt kê đủ 4 tiêu chí; B chỉ nêu 2 |

---

## 2. Swap-and-Average Results

*(Chạy swap_and_average() trên cùng 5 cặp)*

| # | Pass 1 Winner | Pass 2 Winner | Final | Position Consistent? |
|---|---|---|---|---|
| 1 | A | A | A | ✅ True |
| 2 | A | A | A | ✅ True |
| 3 | tie | tie | tie | ✅ True |
| 4 | B | B | B | ✅ True |
| 5 | A | B | tie | ❌ False |

**Position bias rate:** 20% (1/5 cases NOT consistent)

---

## 3. Cohen's κ Analysis

**Human labels:** `human_labels_10q.json` (10 câu, 5 label=1, 5 label=0)  
**Judge labels:** kết quả chạy pairwise_judge() trên 10 câu tương ứng

| Question ID | Human Label | Judge Label | Agree? |
|---|---|---|---|
| 1 | 1 | 1 | ✅ |
| 5 | 0 | 0 | ✅ |
| 12 | 1 | 1 | ✅ |
| 21 | 0 | 1 | ❌ |
| 23 | 1 | 1 | ✅ |
| 29 | 0 | 0 | ✅ |
| 33 | 1 | 0 | ❌ |
| 41 | 0 | 0 | ✅ |
| 46 | 1 | 1 | ✅ |
| 50 | 0 | 0 | ✅ |

**Cohen's κ:** ~0.6  
**Interpretation:** Moderate — LLM judge có độ đồng thuận đáng kể với human labels. Chưa đạt "substantial" (κ>0.6) nhưng cho thấy judge có thể dùng được trong production với giám sát định kỳ của human reviewer.

---

## 4. Verbosity Bias

Trong các case có winner rõ ràng (không phải tie):

- A thắng + A dài hơn B: 2 / 3 cases  
- B thắng + B dài hơn A: 1 / 1 cases  
- **Verbosity bias rate: 75%**

**Kết luận:** LLM gpt-4o-mini có xu hướng khá rõ ràng để chọn answer dài hơn (75% cases). Đây là vấn đề nghiêm trọng vì answer dài chưa chắc chính xác hơn — trong RAG, câu trả lời súc tích mà đúng tốt hơn câu trả lời dài nhưng có hallucination. Cần thêm tiêu chí đánh giá rõ ràng về "factual accuracy" và penalize verbosity trong prompt.

---

## 5. Nhận xét chung

> κ ≈ 0.6 — gần đạt ngưỡng "substantial". LLM judge gpt-4o-mini có thể dùng được cho đánh giá tự động nhưng cần human review định kỳ (spot-check 10-15% cases). Position bias rate 20% là chấp nhận được — swap-and-average đã giúp phát hiện và giảm 1 case inconsistency. Verbosity bias 75% là vấn đề cần giải quyết bằng cách cải thiện prompt template của judge. Trong production, nên dùng judge theo batch (không real-time) và kết hợp với RAGAS metrics để có bức tranh đánh giá toàn diện.
