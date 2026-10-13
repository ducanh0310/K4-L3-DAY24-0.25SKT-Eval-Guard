# Blueprint CI/CD cho RAG và guardrails

Nhóm: 0.25SKT

Người phụ trách: Nguyễn Đức Anh - 2A202602888

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-13T04:54:04.436562+00:00; guard models: `gpt-4o-mini`.
Models generation/evaluator/judge từ A/B:
- generation: `gpt-4o-mini-2024-07-18` (từ `artifacts/ragas_50q.json`)
- evaluator: `gpt-4o-mini`, `text-embedding-ada-002` (từ `artifacts/ragas_50q.json`)
- judge: `gpt-4o-mini` (từ `artifacts/judge_results.json`)
- guard: `gpt-4o-mini` (từ `artifacts/guard_results.json`)

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

Mô tả Data Flow chi tiết:
1. Client gửi yêu cầu truy vấn đến Gateway.
2. Tầng 1 (Sanitization & PII Scanning): Chạy qua Microsoft Presidio Analyzer và Anonymizer trên local CPU. Tầng này tập trung phát hiện thông tin định danh nhạy cảm theo danh mục bảo vệ trọng tâm (VN CCCD, số điện thoại Việt Nam, địa chỉ email). Nếu phát hiện PII, request bị chặn lập tức (short-circuit), trả về thông báo từ chối an toàn và ghi log bảo mật.
3. Tầng 2 (NeMo Input Rails): Truy vấn sạch PII tiếp tục được kiểm tra an toàn intent bằng NeMo Guardrails (LLMRails). Tầng này phát hiện các hình thức jailbreak, prompt injection, và câu hỏi lệch chủ đề chính sách (off-topic). Nếu vi phạm, request bị chặn (short-circuit) và không được gửi vào hệ thống RAG phía sau.
4. Tầng 3 (RAG Pipeline): Truy vấn an toàn được đưa vào pipeline cốt lõi (Module M1 đến M5): tìm kiếm vector trên Qdrant, lọc lại ngữ cảnh qua reranking, và gửi prompt kèm context tối ưu tới LLM generator để tạo câu trả lời.
5. Tầng 4 (NeMo Output Rails): Câu trả lời từ LLM được quét qua tầng kiểm soát đầu ra trước khi gửi về người dùng. Nếu phát hiện rò rỉ thông tin nội bộ không được phép, câu trả lời sẽ bị từ chối hoặc bôi đen (redact).

Failure Action, Logging & Quản lý Secrets:
- Cơ chế ghi log: Khi request bị chặn ở bất kỳ tầng nào, hệ thống tạo một audit log có gắn `request_id`, thời điểm, tên tầng thực thi chặn (`presidio`, `nemo_input`, `nemo_output`) và `flagged_reason`. Mọi nội dung PII đều được che giấu (masking) trước khi lưu vào SIEM để đảm bảo tuân thủ quyền riêng tư dữ liệu.
- Quản lý Secrets: Toàn bộ khóa bảo mật (`OPENAI_API_KEY`, `QDRANT_API_KEY`) được lưu trữ tập trung tại Secret Manager / Key Vault và được nạp vào container qua biến môi trường tại runtime; tuyệt đối không commit vào repository.
- Điều kiện Fallback: Áp dụng cơ chế fail-closed đối với các rào chắn bảo vệ (nếu tầng guardrails gặp lỗi kết nối quá 2.0 giây thì từ chối xử lý truy vấn nhằm ngăn chặn tấn công bypass).

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2; phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 11.69 ms | 14.85 ms | 14.85 ms | Rất tốt, tối ưu hoàn toàn (xử lý regex và spaCy cục bộ) |
| NeMo input | 954.20 ms | 1403.75 ms | 2044.70 ms | Vượt ngân sách đo lường do phụ thuộc network API OpenAI |
| Input guard sequential | 967.46 ms | 1413.36 ms | 2054.24 ms | Vượt ngân sách 500ms (latency_budget_ok = False) |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế: 1413.36 ms (P95); budget_ok: False.

Phân tích điểm nghẽn (bottleneck) và kế hoạch đo lường:
- Tầng Presidio đạt hiệu năng xuất sắc với P95 chỉ 14.85 ms, chiếm xấp xỉ 1% tổng thời gian và hoàn toàn phù hợp với mọi tiêu chuẩn realtime.
- Điểm nghẽn lớn nhất nằm ở tầng NeMo Input Guardrails (P95 đạt 1403.75 ms). Nguyên nhân do NeMo LLMRails phải thực hiện truy vấn embedding và hoàn thành cuộc gọi mạng tới OpenAI API để phân loại ngữ nghĩa. Khi chạy tuần tự không short-circuit, tổng độ trễ input guard chạm ngưỡng 1413.36 ms, không đạt mục tiêu ngân sách 500ms.
- Đối với các phần ngoài phạm vi bắt buộc (RAG pipeline, NeMo output và full short-circuit request), giá trị hiện tại được giữ nguyên là "chưa đo". Kế hoạch đo lường thực tế sẽ sử dụng distributed tracing (OpenTelemetry) với kích thước mẫu 100 requests, 5 lượt warm-up ban đầu và lưu trữ timestamp chi tiết tại từng chặng để đánh giá toàn diện chu trình xử lý end-to-end.

## CI gates đề xuất

Các tiêu chí Quality Gates cho pipeline kiểm thử tự động:
- Gate 1 (RAG Quality): RAGAS faithfulness ≥ 0.75, avg_score ≥ 0.65 trên tập 50 câu hỏi chuẩn hóa.
- Gate 2 (Input Defense): Tỷ lệ chặn tấn công adversarial ≥ 18/20 (vượt mốc tiêu chuẩn 15/20); tỷ lệ chặn nhầm câu hỏi lành mạnh (clean false positive rate) bắt buộc bằng 0.0 (8/8 câu clean được cho phép).
- Gate 3 (Output Safety): 100% các trường hợp kiểm thử đầu ra (4/4 output cases) phải xử lý chính xác (giữ nguyên câu an toàn và redact/thay thế câu không an toàn).
- Gate 4 (Latency Budget): Ngưỡng P95 < 500ms sẽ được kích hoạt khi chuyển đổi NeMo Guardrails sang mô hình ONNX / Local SLM (như Llama-Guard chạy on-premise) thay vì phụ thuộc Internet API.

Lệnh thực thi trong quy trình CI:
```bash
python -m pytest tests/ -q --tb=short
python scripts/check_lab.py --with-tests
```
Nếu bất kỳ điều kiện nào không thỏa mãn, script kiểm thử sẽ trả về mã thoát khác 0 (exit code nonzero), qua đó tự động dừng pipeline và từ chối cấp quyền merge Pull Request. Quy trình này là thiết kế đề xuất cho hệ thống tự động hóa và hiện tại chưa được triển khai chạy ngầm trên remote runner.

## Monitoring và cải tiến

Chỉ số giám sát vận hành production và ngưỡng cảnh báo:
- False Positive Rate: Thiết lập cảnh báo mức High khi FPR trên luồng người dùng thực tế vượt ngưỡng 1.0% trong khoảng thời gian 1 giờ.
- Anomaly Block Rate: Cảnh báo khi số lượng truy vấn bị chặn tăng đột biến (> 20% tổng lưu lượng), dấu hiệu của đợt tấn công từ chối dịch vụ hoặc jailbreak diện rộng.
- Giám sát độ trôi (Drift): Định kỳ trích xuất 5% dữ liệu truy vấn ẩn danh để chạy đánh giá LLM Judge, tự động cảnh báo nếu hệ số Cohen’s κ sụt giảm dưới 0.60.

Số liệu thực tế thu được từ hệ thống kiểm thử:
- RAGAS mean: 0.7971; nhóm phân phối yếu nhất: multi_hop (điểm trung bình: 0.7460).
- Cohen’s κ: 1.0000 (đạt mức đồng thuận tuyệt đối 10/10 mẫu calibration).
- Attacks passed: 20/20 (100% kịch bản tấn công bị chặn); clean false_positive_rate: 0.0000.

Định hướng cải tiến kỹ thuật dựa trên bằng chứng thu thập:
1. Tối ưu độ trễ rào chắn: Triển khai mô hình phân loại cục bộ dạng ONNX runtime (như MiniLM hoặc RoBERTa fine-tuned) để thay thế bước gọi API mạng trong NeMo Guardrails, mục tiêu đưa P95 của Input Guard xuống dưới 50ms.
2. Cải thiện độ trung thực nhóm câu hỏi multi-hop: Bổ sung bước phân tách truy vấn (Query Decomposition) kết hợp thuật toán reranking tại module m3_rerank để nâng điểm faithfulness nhóm multi_hop từ 0.5186 lên trên 0.8000.
3. Mở rộng mẫu nhận diện PII cho thị trường Việt Nam: Tích hợp thêm regex và recognizer cho mã số thuế doanh nghiệp, số thẻ bảo hiểm y tế và biển số xe để gia tăng độ che phủ an toàn dữ liệu.
4. Đa dạng hóa bộ kiểm thử tự động: Bổ sung 50 kịch bản tấn công jailbreak thế hệ mới (như multi-language cipher và roleplay injection) vào CI suite để duy trì độ tin cậy của hệ thống phòng vệ.
