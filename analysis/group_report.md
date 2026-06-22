# Group Report — Lab 18: Production RAG

**Nhóm:** Cá nhân  
**Ngày:** 22/06/2026

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Phạm Quang Dũng | M1: Chunking | ✅ | 13/13 |
| Phạm Quang Dũng | M2: Hybrid Search | ✅ | 5/5 |
| Phạm Quang Dũng | M3: Reranking | ✅ | 5/5 |
| Phạm Quang Dũng | M4: Evaluation | ✅ | 4/4 |
| Phạm Quang Dũng | M5: Enrichment | ✅ | 10/10 |

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.0000 | 0.5333 | +0.5333 |
| Answer Relevancy | 0.0000 | 0.5537 | +0.5537 |
| Context Precision | 0.0000 | 0.8750 | +0.8750 |
| Context Recall | 0.0000 | 0.7500 | +0.7500 |

## Key Findings

1. **Biggest improvement:** Hybrid retrieval cải thiện mạnh phần context, đặc biệt `context_precision = 0.8750`.
2. **Biggest challenge:** Latency cao do enrichment gọi OpenAI tuần tự cho 125 chunks và dense indexing dùng `BAAI/bge-m3`.
3. **Surprise finding:** Retrieval khá tốt nhưng nhiều câu trả lời vẫn là "Không tìm thấy.", cho thấy bottleneck nằm ở prompt/generation và multi-hop context assembly.

## Presentation Notes (5 phút)

1. RAGAS scores: naive baseline đều 0.0000; production đạt `faithfulness 0.5333`, `answer_relevancy 0.5537`, `context_precision 0.8750`, `context_recall 0.7500`.
2. Biggest win: M2 Hybrid Search + M5 Enrichment giúp context precision/recall tăng rõ so với baseline.
3. Case study: câu "Nhân viên được nghỉ bao nhiêu ngày phép năm?" bị trả 12 ngày thay vì 15 ngày do version conflict giữa v2023 và v2024.
4. Next optimization nếu có thêm 1 giờ: thêm metadata versioning, boost tài liệu hiện hành, query decomposition cho multi-hop/numeric questions, và cache enrichment.
