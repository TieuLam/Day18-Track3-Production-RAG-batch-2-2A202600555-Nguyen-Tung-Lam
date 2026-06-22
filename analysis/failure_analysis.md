# Failure Analysis — Lab 18 Production RAG

Dựa trên kết quả từ `ragas_report.json`, dưới đây là phân tích 5 câu hỏi (Bottom-5) có điểm trung bình thấp nhất, được ánh xạ với Diagnostic Tree.

| # | Question | Worst Metric | Score | Diagnosis | Suggested Fix |
|---|----------|-------------|-------|-----------|---------------|
| 1 | Bao lâu phải đổi mật khẩu một lần? | faithfulness | 0.00 | LLM hallucinating | Tighten prompt, lower temperature |
| 2 | Khi phát hiện malware trên máy, nhân viên có nên tự xử lý không? | answer_relevancy | 0.00 | Answer doesn't match question | Improve prompt template |
| 3 | Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu? | faithfulness | 0.00 | LLM hallucinating | Tighten prompt, lower temperature |
| 4 | Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT? | context_recall | 0.33 | Missing relevant chunks | Improve chunking or add BM25 |
| 5 | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | faithfulness | 0.50 | LLM hallucinating | Tighten prompt, lower temperature |

## Nhận xét:
- Phần lớn lỗi rơi vào `faithfulness` do LLM sinh ra câu trả lời chứa thông tin không có trong context được cấp (hallucinating). Để khắc phục, cần tối ưu lại prompt (chỉ định rõ "Chỉ trả lời dựa trên context được cung cấp") và giảm nhiệt độ (temperature) của mô hình OpenAI.
- Một số câu hỏi phức tạp (nhiều vế như câu 4) bị lỗi `context_recall` do pipeline không truy xuất đủ các chunk cần thiết chứa toàn bộ thông tin. Việc cải thiện bằng **HyQA (Hypothetical Questions)** có thể giúp nối từ khóa tốt hơn cho các truy vấn phức tạp.
