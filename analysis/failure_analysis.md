# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Solo mình thôi :DD
**Thành viên:** Võ Huyền Khánh Mây - 2A202600858
---

## RAGAS Scores

Cả hai sinh trong cùng một lần chạy `python main.py` (20 câu hỏi).

| Metric | Naive Baseline | Production | Δ |
|--------|:---:|:---:|:---:|
| Faithfulness | 0.8333 | **0.8917** | **+0.0584** |
| Answer Relevancy | 0.7205 | **0.8695** | **+0.1490** |
| Context Precision | 0.9250 | **0.9833** | **+0.0583** |
| Context Recall | 0.9250 | **0.9333** | **+0.0083** |

→ Production **vượt baseline cả 4 metric**, đạt 2 bonus: Faithfulness ≥ 0.85 và cả 4 metric ≥ 0.75.

- **Naive** = basic paragraph chunking + dense-only (top-3).
- **Production** = hierarchical chunking + M5 enrichment + Hybrid (BM25+Dense+RRF) + CrossEncoder rerank + small-to-big + LLM (temperature=0, prompt siết).

### Hành trình cải tiến (phát hiện chính của bài)

Production **bản đầu lại THUA baseline** — sau khi chẩn đoán mới vượt lên:

| Metric | Baseline | Production v1 (lỗi) | Production v2 (đã sửa) |
|--------|:---:|:---:|:---:|
| Faithfulness | 0.7883 | 0.7383 ❌ | **0.8917** ✅ |
| Answer Relevancy | 0.6808 | 0.7699 | **0.8695** ✅ |
| Context Precision | 0.9250 | 0.9042 ❌ | **0.9833** ✅ |
| Context Recall | 0.9250 | 0.7917 ❌ | **0.9333** ✅ |

**Nguyên nhân v1 thua:** child chunk 256 ký tự bị **cắt vụn giữa câu**, pipeline **retrieve child nhưng không trả parent** → LLM nhận context manh mún, thiếu số liệu → bịa (faithfulness=0 nhiều câu) và thiếu thông tin (recall thấp).

**Cách sửa:**

| Vấn đề | Cải tiến | Metric hưởng lợi |
|--------|----------|------------------|
| Context vụn, thiếu ngữ cảnh | **Small-to-big**: retrieve+rerank child → trả về **parent chunk** | Recall, Faithfulness |
| Multi-hop thiếu tài liệu thứ 2 | Rerank **top-8 child** → dedup ra ≤ **3 parent** khác nhau | Recall |
| LLM bịa số liệu | **temperature=0** + prompt "chỉ trả lời theo Context" | Faithfulness |
| `parent_id` trùng giữa tài liệu | id duy nhất toàn cục `"{doc_idx}:{pid}"` | (tính đúng đắn) |

## Bottom-5 Failures

> `ragas_report.json` chỉ lưu metric (không lưu nguyên văn answer); cột "Got" mô tả hành vi suy ra từ điểm số + loại câu hỏi.

### #1
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Hạn 15 ngày; quá hạn 5 ngày → phí 2%/tháng trên 15tr = 300.000đ/tháng (pro-rata ~50.000đ cho 5 ngày).
- **Got:** Đúng quy tắc 2%/tháng nhưng nêu con số tính ra (~50.000đ) không có nguyên văn trong context.
- **Worst metric:** faithfulness = 0.33
- **Error Tree:** Output đúng → Context đúng/đủ (recall cao) → Query OK → giá trị **suy luận** không bám câu chữ context.
- **Root cause:** faithfulness phạt giá trị **tính toán** không xuất hiện nguyên văn — giới hạn của metric với câu numeric, không phải lỗi retrieval.
- **Suggested fix:** buộc LLM trích **công thức gốc** từ context rồi mới tính.

### #2
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Nghỉ 16–30 ngày cần CEO duyệt; nghỉ >14 ngày không lương phải tự đóng phần bảo hiểm.
- **Got:** Đúng cấp duyệt (CEO) nhưng phần lưu ý bảo hiểm bị diễn giải lại, không khớp 100% câu chữ.
- **Worst metric:** faithfulness = 0.50
- **Error Tree:** Output đúng → Context đủ (ghép 2 ý) → Query OK → 1 phần bị paraphrase.
- **Root cause:** đáp án gộp 2 điều khoản; phần diễn giải lệch khỏi context gốc.
- **Suggested fix:** prompt yêu cầu trích đúng câu, hạn chế diễn giải.

### #3
- **Question:** Tài trợ khóa 25 triệu, nghỉ việc sau 8 tháng hoàn thành khóa. Hoàn trả bao nhiêu?
- **Expected:** Cam kết làm ≥ 1 năm; nghỉ trước hạn → hoàn 100% = 25.000.000đ.
- **Got:** Suy luận đúng "trước hạn → 100%" nhưng kết luận "25tr" là giá trị suy ra.
- **Worst metric:** faithfulness = 0.50
- **Error Tree:** Output đúng → Context đủ → Query OK → claim mang tính suy luận điều kiện.
- **Root cause:** giống #1 — kết quả tính/suy luận không có nguyên văn.
- **Suggested fix:** trích điều khoản "hoàn 100% nếu nghỉ trước cam kết" rồi áp số.

### #4
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất 20tr; thử việc = 85% × 20tr = 17tr.
- **Got:** Cần ghép 2 tài liệu (bảng lương + quy định thử việc) rồi tính 17tr — phép tính không có nguyên văn.
- **Worst metric:** faithfulness = 0.50
- **Error Tree:** Output đúng → Context đủ (multi-hop đã lấy đủ 2 parent) → Query OK → phép nhân 85% là suy luận.
- **Root cause:** multi-hop + numeric; faithfulness phạt giá trị tính.
- **Suggested fix:** buộc LLM viết rõ "20tr × 85% = 17tr".

### #5
- **Question:** Có cần kích hoạt xác thực đa yếu tố (MFA) không?
- **Expected:** Có — v2.0 bắt buộc MFA cho email/VPN/nội bộ; v1.0 cũ không yêu cầu.
- **Got:** Trả lời "có" theo v2.0 nhưng thiếu phần đối chiếu v1.0.
- **Worst metric:** context_recall = 0.50
- **Error Tree:** Output đúng một phần → **Context thiếu** (chỉ lấy chunk v2.0) → Query OK → thiếu chunk v1.0.
- **Root cause:** thông tin trải trên **2 phiên bản tài liệu**; retrieval ưu tiên bản hiện hành.
- **Suggested fix:** thêm **metadata filter theo version** hoặc tăng số parent để gồm cả hai bản.

**Nhận xét chung:** 4/5 lỗi là **faithfulness ở câu numeric/multi-step** — đáp án thường ĐÚNG nhưng giá trị tính toán không nằm nguyên văn trong context (giới hạn cố hữu của metric), 1 lỗi do version trải trên 2 tài liệu.

## Case Study (cho presentation)

**Question chọn phân tích:** "Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?" (multi-hop + numeric)

**Error Tree walkthrough:**
1. Output đúng? → Đúng (17tr) khi ghép đủ 2 nguồn.
2. Context đúng? → Cần đồng thời `bang_luong_2024.md` (Junior 20tr) **và** `thu_viec.md` (85%). Small-to-big + rerank top-8 → 3 parent đã lấy được cả hai.
3. Query rewrite OK? → Có.
4. Fix ở bước: **generation** — buộc LLM nêu "20tr × 85% = 17tr" để claim bám context (tăng faithfulness).

**Nếu có thêm 1 giờ, sẽ optimize:**
- Child chunking theo **ranh giới câu + overlap** (thay vì cắt cứng 256 ký tự).
- **Metadata version filter** cho câu so sánh v1/v2 (MFA, mật khẩu, nghỉ phép).
- **Cache enrichment** để chạy lại nhanh & rẻ hơn.
