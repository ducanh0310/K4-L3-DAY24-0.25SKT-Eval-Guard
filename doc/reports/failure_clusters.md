# Phân tích lỗi Phase A

Nhóm: 0.25SKT

Người phụ trách: Nguyễn Đức Anh - 2A202602888

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-13T04:25:31.520214+00:00.
Generation mode/fallback và điều kiện chạy: 100% LLM mode sử dụng mô hình gpt-4o-mini-2024-07-18, không sử dụng extractive fallback; chạy thực nghiệm live trên tập 50 câu hỏi truy vấn dữ liệu chính sách nội bộ được lập chỉ mục trên Qdrant vector database.
Models generation/evaluator: generation: `gpt-4o-mini-2024-07-18`; evaluator: `gpt-4o-mini`, `text-embedding-ada-002` từ module m4.

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 0.8833 | 0.5186 | 0.8000 |
| answer_relevancy | 0.7540 | 0.7530 | 0.6858 |
| context_precision | 0.9583 | 0.8917 | 0.9833 |
| context_recall | 0.9000 | 0.8208 | 0.6167 |
| avg_score | 0.8739 | 0.7460 | 0.7715 |

## Bottom 10 và nguyên nhân

Bảng dưới đây thống kê chính xác 10 câu hỏi có điểm trung bình thấp nhất từ Task 3, kèm chẩn đoán và đề xuất khắc phục:

| Rank | ID | Câu hỏi | Phân phối | Avg Score | Worst Metric | Chẩn đoán | Đề xuất khắc phục |
|---|---|---|---|---|---|---|---|
| 1 | 39 | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | multi_hop | 0.0000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 2 | 5 | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | factual | 0.2500 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 3 | 50 | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | adversarial | 0.3333 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 4 | 33 | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | multi_hop | 0.3750 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 5 | 43 | Mật khẩu phải có tối thiểu bao nhiêu ký tự? | adversarial | 0.4167 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 6 | 9 | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | factual | 0.5000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 7 | 21 | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | multi_hop | 0.6795 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 8 | 31 | Nhân viên đi công tác trong nước, ở khách sạn 2 đêm với giá 1.500.000 VNĐ/đêm. Công ty thanh toán tối đa bao nhiêu cho tiền khách sạn? | multi_hop | 0.6838 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 9 | 7 | Nghỉ phép không lương 20 ngày cần ai phê duyệt? | factual | 0.6965 | context_precision | Too many irrelevant chunks | Add reranking or metadata filter |
| 10 | 24 | Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. B bị phạt bao nhiêu? | multi_hop | 0.7023 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |

Đối chiếu answer, contexts và reference của các câu đại diện:
- Câu 39 (multi_hop, rank 1, avg 0.0000): Câu hỏi yêu cầu tổng hợp đối chiếu hai phiên bản tài liệu khác nhau. Contexts truy xuất chứa các đoạn rời rạc về mật khẩu của cả hai bản nhưng LLM nhầm lẫn các thông số thời hạn đổi định kỳ giữa v1.0 và v2.0, sinh ra câu trả lời chứa thông tin không có trong tài liệu dẫn chứng, khiến metric faithfulness rơi xuống mức 0.0.
- Câu 5 (factual, rank 2, avg 0.2500): Quy định mua sắm nêu rõ mua thiết bị trên 50 triệu đồng phải có sự phê duyệt của Tổng Giám đốc. Tuy nhiên mô hình suy đoán cấp Giám đốc bộ phận là đủ thẩm quyền, dẫn đến sai lệch nghiêm trọng so với tài liệu gốc dù ngữ cảnh đã được truy xuất đầy đủ.
- Câu 50 (adversarial, rank 3, avg 0.3333): Câu hỏi gài bẫy giả định "để tăng bảo mật thêm không". Chính sách bảo mật nội bộ quy định tuyệt đối cấm VPN bên thứ ba khi truy cập mạng công ty. LLM bị dẫn dắt bởi lời giải thích giả định của câu hỏi nên đã trả lời nhượng bộ cho phép sử dụng nếu khai báo, vi phạm tính trung thực với tài liệu.
- Câu 7 (factual, rank 9, avg 0.6965): Đây là câu duy nhất trong bottom 10 có worst metric là context_precision. Hệ thống retrieval trả về nhiều chunk liên quan đến chế độ nghỉ phép năm thông thường thay vì chunk đặc thù về nghỉ không lương dài ngày (>14 ngày cần Giám đốc Nhân sự duyệt), làm loãng ngữ cảnh cung cấp cho mô hình.

## Matrix và nhóm yếu nhất

Ma trận phân phối số câu có metric thấp nhất (worst_metric) theo từng nhóm:

| Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng |
|---|---|---|---|---|
| faithfulness | 3 | 14 | 2 | 19 |
| context_recall | 1 | 2 | 8 | 11 |
| context_precision | 2 | 2 | 0 | 4 |
| answer_relevancy | 14 | 2 | 0 | 16 |

Nhóm yếu theo mean thấp nhất: multi_hop; mean: 0.7460.
Worst metric xuất hiện nhiều nhất: faithfulness; số câu: 19 câu (chiếm 38% tổng số câu).

Giải thích nguyên nhân nhóm multi_hop yếu:
Mặc dù nhóm factual đạt điểm trung bình rất cao (0.8739), nhóm multi_hop chỉ đạt 0.7460 do đặc thù bài toán đa bước đòi hỏi kết hợp nhiều ràng buộc cùng lúc (ví dụ: cấp bậc nhân viên + số năm thâm niên + bảng phụ cấp + phiên bản chính sách). Context retrieval cho multi-hop tương đối tốt (precision 0.8917, recall 0.8208), nhưng generator gặp hiện tượng hallucination khi kết nối các mảnh thông tin, dẫn tới 14/20 câu trong nhóm này bị điểm faithfulness thấp nhất (faithfulness trung bình nhóm multi_hop chỉ đạt 0.5186).

Đề xuất khắc phục cụ thể:
1. Siết chặt system prompt của generation: Yêu cầu LLM chỉ đưa ra kết luận khi có trích dẫn trực tiếp từ ngữ cảnh, nếu thiếu mảnh ghép thông tin giữa các bước phải tuyên bố rõ ràng thay vì tự ý ngoại suy.
2. Thiết lập temperature thấp (0.0 hoặc 0.1) trong cấu hình generator để triệt tiêu tính ngẫu nhiên khi tính toán số liệu và đối chiếu phiên bản.
3. Bổ sung kỹ thuật Query Decomposition trước khi tìm kiếm: Tách câu hỏi đa bước thành các truy vấn đơn lẻ để đảm bảo vector search gom đủ dữ liệu, sau đó chạy qua module m3_rerank để chọn đúng top chunks.
4. Cách kiểm chứng: Chạy lại `python scripts/run_experiments.py --phase a --live` và so sánh tệp `artifacts/ragas_50q.json` mới; điều kiện thành công là điểm faithfulness của nhóm multi_hop vượt ngưỡng 0.7500.

## Bẫy adversarial và giới hạn

So sánh với factual và multi_hop:
Nhóm adversarial có điểm context_recall thấp nhất toàn hệ thống (chỉ đạt 0.6167, với 8/10 câu có context_recall là worst metric). Các câu hỏi này sử dụng kỹ thuật cài bẫy phủ định ("không cần làm gì", "nhân viên thử việc có được nghỉ phép năm không") hoặc bẫy phiên bản cũ/mới (v2023 vs v2024). Retriever có xu hướng dựa vào keyword tương đồng và bỏ qua các tiền tố phủ định hoặc số hiệu phiên bản, dẫn đến việc lấy nhầm tài liệu cũ hoặc thiếu mất điều khoản loại trừ.

Điểm số thấp hơn không tự chứng minh pipeline đã nhận ra bẫy:
Thực tế điểm số thấp phản ánh generator bị lừa bởi câu hỏi gài bẫy và tạo ra câu trả lời sai lệch, chứ không phải hệ thống chủ động phát hiện bẫy để từ chối trả lời. Để pipeline thực sự xử lý được bẫy adversarial, cần kết hợp các cơ chế bảo vệ chuyên dụng như Guardrails ở đầu vào và đầu ra.

Giới hạn thực nghiệm:
Tập dữ liệu kiểm thử gồm 50 câu (20 factual, 20 multi_hop, 10 adversarial). Mô hình generation được sử dụng là gpt-4o-mini-2024-07-18; kết quả phụ thuộc vào prompt template hiện tại và chưa được đối chiếu mở rộng trên các mô hình khác nhau (như GPT-4o, Claude 3.5 Sonnet hay Llama 3).
