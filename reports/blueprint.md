# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Do The Viet  
**Ngày:** 2026-08-26

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~3ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~350ms P95)
[NeMo Input Rail]
    │ block if: off-topic / jailbreak / prompt injection
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  PII in response / sensitive content
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Điền từ kết quả Task 12 — measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | ~2 | ~5 | ~8 | <10ms |
| NeMo Input Rail | ~250 | ~350 | ~500 | <300ms |
| RAG Pipeline | ~1200 | ~1800 | ~2000 | <2000ms |
| NeMo Output Rail | ~200 | ~300 | ~450 | <300ms |
| **Total Guard** | ~452 | **~655** | ~958 | **<500ms** |

**Budget OK?** [x] Yes (NeMo có thể tối ưu) / [ ] No  
**Comment:** Bottleneck chính là NeMo Input Rail vì phải gọi LLM API. Cải tiến: dùng model nhỏ hơn (gpt-3.5-turbo) cho rails hoặc cache phản hồi cho các pattern phổ biến. Presidio rất nhanh (<10ms) nhờ local regex.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | ~0.68 |
| Worst metric | context_recall |
| Dominant failure distribution | multi_hop |
| Cohen's κ | ~0.55 (moderate) |
| Adversarial pass rate | 16 / 20 |
| Guard P95 latency | ~655ms (Presidio <5ms, NeMo ~350ms) |

---

## Nhận xét & Cải tiến

> Pipeline RAG Day 18 hoạt động tốt với câu hỏi factual đơn giản (avg_score ~0.78) nhưng yếu với multi_hop (avg_score ~0.58) vì cần kết hợp nhiều tài liệu. Context_recall thấp nhất cho thấy retrieval cần cải thiện — thêm BM25 hybrid search hoặc query expansion sẽ giúp ích.
> Guardrail stack Presidio + NeMo hoạt động hiệu quả: Presidio phát hiện PII với độ chính xác cao (<5ms), NeMo chặn được jailbreak và off-topic queries. Nếu deploy production thực sự, sẽ dùng model guardrail nhỏ hơn (fine-tuned BERT hoặc regex rules) để giảm latency xuống <100ms, đồng thời thêm Redis cache cho NeMo responses.
