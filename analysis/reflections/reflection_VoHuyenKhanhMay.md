# Individual Reflection — Lab 18

**Tên:** Võ Huyền Khánh Mây - 2A202600858
**Module phụ trách:** Toàn bộ M1–M5 + pipeline (bài cá nhân)

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 RAGAS Eval, M5 Enrichment + tối ưu `pipeline.py`.
- **Hàm/class chính:**
  - M1: `chunk_semantic`, `chunk_hierarchical`, `chunk_structure_aware`
  - M2: `segment_vietnamese`, `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion`
  - M3: `CrossEncoderReranker`, `FlashrankReranker`
  - M4: `evaluate_ragas` (try/except + nan-safe), `failure_analysis`
  - M5: `summarize_chunk`, `generate_hypothesis_questions`, `contextual_prepend`, `extract_metadata`, `_enrich_single_call`
  - Pipeline: **small-to-big retrieval** (child → parent), prompt siết + `temperature=0`
- **Số tests pass:** 37/37 (M1:13, M2:5, M3:5, M4:4, M5:10)

## 2. Kiến thức học được — Mapping bài giảng → code

| Khái niệm bài giảng | Module | Hàm cụ thể | Quan sát từ thực nghiệm |
|---|---|---|---|
| Semantic / Hierarchical chunking | M1 | `chunk_hierarchical` | Child 256 ký tự cắt vụn → phải dùng parent làm context |
| BM25 + Dense fusion (RRF) | M2 | `reciprocal_rank_fusion` | Hybrid bù chỗ dense yếu với từ khóa/số liệu tiếng Việt |
| Vietnamese tokenization | M2 | `segment_vietnamese` | underthesea nối `_`; phải `replace("_"," ")` cho BM25 |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank` | top-20 → top-k, đẩy đúng đoạn liên quan lên đầu |
| RAGAS 4 metrics | M4 | `evaluate_ragas` | Faithfulness thấp nhất, đặc biệt với câu numeric |
| Contextual embeddings (enrichment) | M5 | `_enrich_single_call` | Prepend context giúp retrieval; nhưng nên trả parent sạch cho LLM |
| Small-to-big retrieval | pipeline | `run_query` | Retrieve child (precision) → trả parent (recall) — đòn bẩy lớn nhất |

- **Khái niệm mới nhất:** small-to-big retrieval — tách "đơn vị để tìm" khỏi "đơn vị để đọc".
- **Điều bất ngờ nhất:** Production **ban đầu THUA** baseline ở 3/4 metric — nhiều kỹ thuật chưa chắc tốt hơn nếu ghép sai.
- **Kết nối bài giảng:** chunking strategies, hybrid search, reranking, RAGAS evaluation.

## 3. Khó khăn & Cách giải quyết

| Khó khăn | Lỗi/triệu chứng | Cách giải quyết | ~Debug |
|---|---|---|---|
| Qdrant không kết nối | `WinError 10053` ở `recreate_collection`; IPv6 `[::1]:6333` reset, IPv4 OK | Đổi `QDRANT_HOST` → `127.0.0.1` (env-overridable) | ~20'|
| Production thua baseline | faithfulness↓, context_recall 0.79 | Phân tích: child vụn + không trả parent → áp **small-to-big + temperature=0 + prompt siết** | ~30'|
| `main.py` crash cuối | `FileExistsError` ở `os.rename` (Windows không ghi đè) | Đổi sang `os.replace` | ~5'|

- **Kiến thức thiếu → bổ sung:** hành vi resolve `localhost` (IPv4/IPv6) trên Windows+Docker; cơ chế chấm của RAGAS faithfulness (phạt giá trị tính toán không có nguyên văn trong context).

## 4. Action Plan cho project cá nhân

**Hiện tại:** RAG pipeline cơ bản (chunk cố định + dense-only), chưa đánh giá định lượng.

**Plan áp dụng (rút ra từ lab):**
1. [x] **Chunking:** hierarchical + small-to-big (retrieve child, đọc parent) — đòn bẩy mạnh nhất.
2. [x] **Search:** Hybrid BM25 + Dense + RRF (BM25 quan trọng cho tiếng Việt/số liệu).
3. [x] **Reranking:** CrossEncoder `bge-reranker-v2-m3` (FlashRank nếu cần latency thấp).
4. [x] **Evaluation:** RAGAS 4 metrics + failure analysis bottom-N theo diagnostic tree.
5. [x] **Enrichment:** combined single-call (summary+questions+context+metadata) để tiết kiệm chi phí.
6. [x] **Generation:** `temperature=0` + prompt "chỉ trả lời theo context" để tối đa faithfulness.

**Timeline:** Sẽ hoàn thành trong trong các bài tiếp theo.
## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|:---:|
| Hiểu bài giảng | 5 |
| Code quality | 5 |
| Teamwork | N/A (làm cá nhân) |
| Problem solving | 5 |
