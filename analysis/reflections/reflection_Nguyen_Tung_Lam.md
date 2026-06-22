# Reflection — Lab 18 Production RAG
**Họ tên:** Nguyễn Tùng Lâm

---

## Phần 1: Mapping bài giảng

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Threshold 0.85 (mặc định) tạo ra các chunk có ý nghĩa trọn vẹn hơn so với basic chunking chia cắt giữa các câu. |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | RRF giải quyết được điểm yếu của Dense (thiếu chính xác với từ khóa cụ thể) và BM25 (không hiểu đồng nghĩa) để đưa ra kết quả kết hợp tốt nhất. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Reranking bằng mô hình `BAAI/bge-reranker-v2-m3` tăng đáng kể độ chính xác (precision) của các context được trả về cho LLM. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Nhờ RAGAS (Faithfulness, Answer Relevancy, Context Precision, Context Recall), ta định lượng được chất lượng pipeline thay vì cảm tính. Điểm Faithfulness và Context Precision khá cao, tuy nhiên Answer Relevancy cần cải thiện thêm. |
| Contextual embeddings | M5 | `contextual_prepend()` | Việc chèn thông tin tài liệu gốc giúp model không bị mất bối cảnh, giảm thiểu retrieval failure, đặc biệt khi chunk bị cắt nhỏ ở M1. |

---

## Phần 2: Khó khăn & giải quyết

1. **Lỗi `UnicodeEncodeError` khi in ra console**
   - **Lỗi gặp phải:** `UnicodeEncodeError: 'charmap' codec can't encode characters in position 2-3: character maps to <undefined>` khi script in ra ký tự emoji (`⚠️ Bỏ qua...`) hoặc các ký tự tiếng Việt ra môi trường console mặc định của Windows.
   - **Cách debug & giải quyết:** Nguyên nhân do console trên Windows mặc định dùng encoding `cp1252` thay vì `utf-8`. Tôi đã sửa lỗi bằng cách thiết lập biến môi trường `$env:PYTHONIOENCODING="utf-8"` trước khi chạy lệnh `python src/pipeline.py` bằng PowerShell.
   - **Kiến thức thiếu & bổ sung:** Thiếu kiến thức về cách xử lý text encoding chuẩn (UTF-8) trên terminal Windows cho Python.

2. **Lỗi Underthesea tokenizer không khớp với BM25**
   - **Lỗi gặp phải:** Khó khăn ở việc kết hợp `underthesea` với BM25. `underthesea` nối từ ghép bằng `_` (VD: `nghỉ_phép`), dẫn đến BM25 không khớp được từ khóa.
   - **Cách debug & giải quyết:** Xử lý bằng cách dùng lệnh `.replace("_", " ")` sau khi đã segment text để trả về khoảng trắng chuẩn cho BM25 xử lý.

---

## Phần 3: Action Plan cho project

### Project: Hệ thống AI hỏi đáp về Chính sách & Cẩm nang nhân viên (Production RAG)

### Hiện tại
- **RAG pipeline hiện tại:** Base trên LlamaIndex hoặc LangChain với vector search thông thường, không có xử lý từ vựng tiếng Việt chuyên sâu và đánh giá định lượng.
- **Known issues:**
  - Kết quả trả về sai lệch khi người dùng gõ từ khóa tiếng Việt phức tạp (không khớp keyword).
  - Thiếu context do chunking cắt gãy câu.
  - Tốn kém token do nhồi nhét quá nhiều document thừa vào LLM.

### Plan áp dụng
1. [x] **Chunking strategy:** Chuyển sang Hierarchical chunking kết hợp Semantic chunking để đảm bảo trích xuất chính xác nhưng vẫn cung cấp context toàn diện.
2. [x] **Search:** Sử dụng Hybrid Search (BM25 + Dense) kết hợp `underthesea` cho Tiếng Việt và RRF để đảm bảo độ bao phủ (Recall).
3. [x] **Reranking:** Áp dụng `bge-reranker-v2-m3` để chọn lọc lại top 3 docs chất lượng nhất (Precision).
4. [x] **Evaluation:** Đưa RAGAS vào CI/CD pipeline để benchmark trên Test-set gồm 50 câu hỏi trước mỗi lần release mô hình.
5. [x] **Enrichment:** Triển khai **Combined mode (1 API call)** cho việc Summarize, Contextual Prepend và Auto Metadata nhằm tiết kiệm chi phí gọi API OpenAI mà vẫn đạt hiệu quả cao.

### Timeline dự kiến
- **Tuần 1:** Triển khai M1 (Advanced Chunking) và M5 (Enrichment). Xây dựng cơ sở hạ tầng Qdrant DB.
- **Tuần 2:** Triển khai M2 (Hybrid Search) và M3 (Reranking). Đảm bảo pipeline luân chuyển data trơn tru.
- **Tuần 3:** Tích hợp M4 (RAGAS Evaluation), đo lường và tinh chỉnh tham số (Threshold, Top_k) để đạt KPIs mong đợi.

---

### Phụ lục: Latency Breakdown Report

Thời gian thực thi ghi nhận trên tập test 107 chunks và 20 queries:

| Bước | Thời gian |
|------|-----------|
| M1 Chunking | < 2s |
| M5 Enrichment (107 chunks) | ~ 434.5s |
| M2 Indexing (BM25+Dense) | ~ 48.6s |
| M3 Reranking (tải model) | 0.0s |
| M4 RAGAS Eval (20 queries) | ~ 49.4s |
| **Total Pipeline Thời Gian Chờ Thực Thi** | **~ 725.4s** |
