# Báo cáo Lab 22 — LangSmith + Prompt Versioning

Họ tên: Nguyễn Mạnh Hải · MSSV: 2A202602988 · Provider: Google Gemini (`gemini-2.5-flash`, embedding `models/gemini-embedding-001`)

## 1. Tổng quan kết quả

| Nhiệm vụ | Kết quả | Evidence |
|---|---|---|
| 1. RAG + LangSmith tracing | 50 câu hỏi chạy qua RAG (107 chunks, k=3). Truy vấn LangSmith bằng API thấy **100 trace `rag-query`** (≥ 50) do pipeline được chạy nhiều lần. | `01_langsmith_traces.png` |
| 2. Prompt Hub + A/B routing | Push và pull thành công 2 prompt `nguyen-manh-hai-rag-prompt-v1/v2`, không dùng fallback local. Routing bằng MD5 chia V1 = 19, V2 = 31 câu. LangSmith ghi nhận 50 trace `ab-rag-query`. | `02_prompt_hub.png`, `02_ab_routing_log.txt` |
| 3. Đánh giá RAGAS | Faithfulness ≥ 0,8 ở cả hai prompt (`target_met: true`). | `03_ragas_scores.png`, `03_ragas_report.json` |
| 4. Guardrails | `PIIDetector` che đủ 4 loại PII (6 test case), `JSONFormatter` sửa 3 loại lỗi JSON và trả JSON dự phòng khi không sửa được (5 test case). | `04_pii_demo_log.txt`, `04_json_demo_log.txt` |

## 2. So sánh V1 và V2 (RAGAS, 50 cặp QA)

| Chỉ số | V1 (ngắn gọn, 2–4 câu) | V2 (có cấu trúc, 3–5 câu) | Chênh lệch |
|---|---|---|---|
| faithfulness | 0,9809 | **0,9823** | +0,0014 |
| answer_relevancy | 0,8080 | **0,8196** | +0,0116 |
| context_recall | 1,0000 | 1,0000 | 0 |
| context_precision | **0,9633** | 0,9600 | −0,0033 |

### Ý nghĩa các chỉ số
- **faithfulness**: câu trả lời có bám vào context đã retrieve hay không (đo mức bịa thêm).
- **answer_relevancy**: câu trả lời có đúng trọng tâm câu hỏi không.
- **context_recall**: context retrieve được có chứa đủ thông tin của đáp án chuẩn không.
- **context_precision**: các đoạn context liên quan có được xếp lên trên các đoạn không liên quan không.

### Phân tích
- **context_recall và context_precision gần như giống nhau giữa V1 và V2.** Hai chỉ số này chỉ phụ thuộc bước retrieve (cùng FAISS, cùng k=3, cùng embedding), không phụ thuộc prompt. Chênh lệch nhỏ của context_precision (0,0033) là nhiễu do LLM chấm điểm, không phải do prompt.
- **faithfulness của cả hai đều rất cao (> 0,98)** vì cả hai prompt đều yêu cầu chỉ dựa trên context và đều chứa `{context}`. Chênh lệch 0,0014 quá nhỏ để kết luận prompt nào tốt hơn.
- **V2 cao hơn V1 khoảng 0,012 ở answer_relevancy.** Đây là khác biệt lớn nhất giữa hai prompt. Giả thuyết hợp lý: V2 yêu cầu xác định facts liên quan rồi viết 3–5 câu có tổ chức, nên câu trả lời đầy đủ hơn và bao phủ ý của câu hỏi tốt hơn; V1 bị giới hạn 2–4 câu nên đôi khi bỏ sót ý. Tôi chưa kiểm chứng giả thuyết này bằng cách so sánh độ dài hay nội dung từng câu trả lời.
- **Lưu ý về độ tin cậy.** RAGAS dùng LLM để chấm điểm nên mỗi lần chạy có sai lệch nhỏ. Điểm V1 giữa hai lần chạy của tôi lệch nhau khoảng 0,008 ở faithfulness (0,9889 lần 1 và 0,9809 lần 2), cùng cỡ với chênh lệch giữa V1 và V2. Vì vậy chỉ nên coi V2 hơn V1 ở answer_relevancy là xu hướng, chưa phải kết luận chắc chắn. Muốn kết luận cần chạy lặp nhiều lần hoặc dùng tập câu hỏi lớn hơn.
- **Hạn chế:** V1 có 1/50 mẫu `answer_relevancy` chấm lỗi (không ra điểm), được bỏ qua khi tính trung bình.

## 3. Sự cố gặp phải và cách xử lý

- **RAGAS lần chạy 1 cho V2 bị `NaN`** ở `answer_relevancy` và `context_precision`. Nguyên nhân là Gemini trả lỗi `429 RESOURCE_EXHAUSTED` và có job bị `TimeoutError` khi RAGAS gọi song song nhiều request. Cách xử lý: thêm `RunConfig(timeout=300, max_workers=4, max_retries=10)` vào `evaluate()` và chạy lại. Lần chạy 2 cho đủ điểm. Tôi cũng sửa phần tính điểm để bỏ qua mẫu lỗi và in cảnh báo số mẫu lỗi thay vì để `NaN` lan vào kết quả.
- **Điểm số trong `03_ragas_report.json` và `data/ragas_report.json` là kết quả lần chạy 2.** Kết quả lần 1 bị lỗi nên không dùng.

## 4. Chi tiết kỹ thuật cần nhớ

- **Guardrails:** với `OnFailAction.FIX`, validator phải trả về `FailResult(fix_value=...)` mới thay được output; `PassResult(value_override=...)` không có tác dụng. `on_fail` truyền vào constructor của validator, không truyền vào `Guard.use()`.
- **Redact số điện thoại:** với regex có sẵn, `(555) 867-5309` được che thành `([PHONE_REDACTED].`, còn sót dấu `(`. Phần số vẫn được che.
- **Trạng thái `✅ Pass` trong demo JSON** hiện ở cả case lỗi đã được sửa, vì khi `FIX` thành công Guardrails coi là pass.
- **A/B routing** dùng MD5 của `request_id` nên tất định: cùng `request_id` luôn cho cùng phiên bản. `random` thì mỗi lần chạy cho kết quả khác nhau, không tái lập được.
- **Prompt versioning** tách prompt khỏi code, nên đổi prompt không cần deploy lại và có thể quay về phiên bản cũ trên Hub.
