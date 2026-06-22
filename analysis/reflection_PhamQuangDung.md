# Individual Reflection — Lab 18

**Tên:** Phạm Quang Dũng  
**Module phụ trách:** Cá nhân — M1, M2, M3, M4, M5  
**Tests:** 37/37 passed  

---

## Phần 1: Mapping bài giảng vào code

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Text được tách thành câu, encode bằng `all-MiniLM-L6-v2`, rồi gom câu theo cosine similarity. Cách này giảm rủi ro cắt giữa ý so với paragraph chunking cơ bản. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Parent chunk giữ bối cảnh rộng, child chunk nhỏ hơn để retrieve chính xác. Pipeline hiện dùng child để index nhưng lưu `parent_id` để có thể mở rộng trả parent context. |
| Structure-aware chunking | M1 | `chunk_structure_aware()` | Markdown headers được giữ trong chunk và đưa vào metadata `section`, giúp truy vết section tốt hơn khi tài liệu có cấu trúc chính sách. |
| BM25 + Dense fusion | M2 | `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | BM25 xử lý keyword tiếng Việt bằng `underthesea`; dense search dùng `BAAI/bge-m3`; RRF merge hai danh sách để cân bằng keyword exact match và semantic match. |
| Vietnamese tokenization | M2 | `segment_vietnamese()` | `underthesea` tạo token có `_`, nên code replace `_` thành space để query "nghỉ phép" vẫn match với tài liệu chứa "nghỉ_phép". |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Interface rerank nhận query-document pairs và sort theo `rerank_score`. Trong môi trường lab, fallback lexical giúp test chạy ổn; production có thể bật model thật bằng `LAB18_USE_REMOTE_RERANKER=1`. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Pipeline đo `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`. Kết quả production tốt ở retrieval nhưng yếu hơn ở answer generation. |
| Failure diagnosis | M4 | `failure_analysis()` | Bottom failures được map sang Diagnostic Tree: hallucination/unsupported claim, missing chunks, irrelevant context, hoặc answer không đúng intent. |
| Contextual enrichment | M5 | `_enrich_single_call()`, `contextual_prepend()` | Enrichment thêm summary, hypothesis questions, context line và metadata trước khi embed, giúp retrieval có thêm vocabulary bridge. |

## Phần 2: Khó khăn và cách giải quyết

### Lỗi 1: Docker/Qdrant chưa chạy

- **Exact error:** `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`
- **Nguyên nhân:** Docker Desktop chưa bật, nên Qdrant không start được.
- **Cách debug:** Kiểm tra `docker info`, `docker compose ps`, sau đó bật Docker Desktop và chạy lại `docker compose up -d`.
- **Bài học:** Dense search phụ thuộc infrastructure local; trước khi chạy pipeline cần kiểm tra service readiness, không chỉ kiểm tra package Python.

### Lỗi 2: Cài dependency bị đứt khi tải Torch

- **Exact error:** `IncompleteRead ... torch-2.12.1-cp311-cp311-win_amd64.whl`
- **Nguyên nhân:** Download gói lớn bị ngắt giữa chừng.
- **Cách debug:** Chạy lại `pip install` với `--retries 10 --timeout 120 --resume-retries 10`.
- **Bài học:** Với stack ML, dependency install nên có retry và cache; tốt nhất dùng virtual environment riêng để tránh conflict global packages.

### Lỗi 3: Pipeline chạy lâu ở enrichment

- **Observation:** `[2/4] Enriching 125 chunks` mất khoảng `506.9s`.
- **Nguyên nhân:** M5 gọi OpenAI tuần tự 1 API call/chunk. 125 chunks nghĩa là 125 calls trước khi indexing.
- **Cách debug:** Kiểm tra process Python, kiểm tra Qdrant collection; vì Qdrant chưa có collection nên biết pipeline chưa qua indexing.
- **Bài học:** Enrichment hữu ích nhưng cần batch/concurrency/cache. Nếu không, latency tăng rất nhanh theo số chunk.

### Lỗi 4: HuggingFace model download bị chậm/kẹt

- **Exact warning:** `You are sending unauthenticated requests to the HF Hub`
- **Nguyên nhân:** `BAAI/bge-m3` tải từ HuggingFace không có `HF_TOKEN`, dễ chậm/rate-limit.
- **Cách debug:** Kiểm tra cache `~/.cache/huggingface/hub/models--BAAI--bge-m3`, thấy file weight chưa tăng; thêm `HF_TOKEN` vào `.env`.
- **Bài học:** Production RAG cần chuẩn bị model cache trước lab/deploy; không nên để lần chạy chính phụ thuộc download remote.

### Lỗi 5: Qdrant tắt giữa pipeline

- **Exact error:** `qdrant_client.http.exceptions.ResponseHandlingException: [WinError 10061] No connection could be made because the target machine actively refused it`
- **Nguyên nhân:** Docker Desktop/Qdrant không còn chạy khi pipeline tới bước indexing.
- **Cách debug:** `docker compose ps`, gọi `QdrantClient(...).get_collections()`, sau đó start lại Docker/Qdrant.
- **Bài học:** Pipeline nên có health check trước khi indexing và retry connection tới vector DB.

## Phần 3: Action Plan cho project

## Project: Internal Policy RAG Assistant

### Hiện tại

- RAG pipeline hiện tại: load tài liệu nội bộ, chunk theo hierarchical strategy, enrich chunk bằng LLM, index bằng BM25 + dense Qdrant, rerank, trả lời bằng LLM, đánh giá bằng RAGAS.
- Known issues:
  - Một số câu multi-hop trả lời "Không tìm thấy" dù tài liệu có dữ kiện.
  - Version conflict: ví dụ chính sách nghỉ phép v2023 là 12 ngày, v2024 hiện hành là 15 ngày.
  - Enrichment chậm vì gọi tuần tự theo từng chunk.
  - Một số câu numeric reasoning tính chưa đúng pro-rata hoặc cần ghép nhiều tài liệu.

### Plan áp dụng

1. [ ] Chunking strategy: dùng hierarchical chunking làm default; khi child được retrieve thì trả kèm parent/section để LLM có đủ bối cảnh.
2. [ ] Search: giữ hybrid BM25 + Dense + RRF; tăng recall bằng multi-query retrieval cho câu hỏi dài hoặc multi-hop.
3. [ ] Reranking: dùng CrossEncoder thật khi môi trường đã cache model; thêm rule boost cho version hiện hành và exact entity match.
4. [ ] Evaluation: dùng RAGAS cho 4 metrics chính, bổ sung custom exactness checks cho câu hỏi numeric/version như ngày phép, mức lương, phần trăm phạt.
5. [ ] Enrichment: giữ combined mode nhưng thêm cache theo hash chunk; extract metadata `version`, `is_current`, `effective_date`, `category`, `entities`.
6. [ ] Prompting: yêu cầu LLM trích dẫn context và ưu tiên chính sách hiện hành khi có nhiều phiên bản mâu thuẫn.

### Timeline

- Tuần 1: Hoàn thiện metadata versioning và parent-context retrieval.
- Tuần 2: Thêm query decomposition cho multi-hop/numeric questions.
- Tuần 3: Bật CrossEncoder thật, benchmark latency và quality.
- Tuần 4: Chạy RAGAS định kỳ, phân tích bottom failures, tối ưu theo từng nhóm lỗi.

## Tự đánh giá

| Tiêu chí | Tự chấm (1-5) | Ghi chú |
|----------|---------------|---------|
| Hiểu bài giảng | 4 | Hiểu rõ vai trò của chunking, hybrid search, rerank và RAGAS qua pipeline thực tế. |
| Code quality | 4 | Code pass tests và có fallback, nhưng cần tối ưu latency/caching nếu production thật. |
| Problem solving | 4 | Debug được Docker, dependency, HF token, Qdrant và RAGAS/pipeline issues. |
| Khả năng áp dụng vào project | 5 | Có plan rõ để xử lý version conflict, multi-hop retrieval và numeric reasoning. |
