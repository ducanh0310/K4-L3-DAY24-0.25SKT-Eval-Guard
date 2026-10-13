# Đánh giá LLM Judge Phase B

Nhóm: 0.25SKT

Người phụ trách: Nguyễn Đức Anh - 2A202602888

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-13T04:26:17.059676+00:00.
Models generation/judge: generation: `gpt-4o-mini-2024-07-18`; judge: `gpt-4o-mini`.

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ artifacts/answers_50q.json:answer; B từ fixtures/human_labels_10q.json:model_answer; reference từ fixtures/test_set_50q.json:ground_truth.
Ids: 1, 5, 12, 21, 41. Không đổi nguồn sau khi xem kết quả judge.

Bảng kết quả đánh giá 5 cặp câu hỏi qua 2 lượt đổi vị trí (Pass 1: A trước B; Pass 2: B trước A):

| ID | Winner Pass 1 | Winner Pass 2 | Final Winner | Position Consistent | Scores Pass 1 (A/B) | Scores Pass 2 (A/B) | Tóm tắt lý do đánh giá |
|---|---|---|---|---|---|---|---|
| 1 | B | B | B | True | A: 0.8, B: 1.0 | A: 0.8, B: 1.0 | Candidate B trả lời đầy đủ số ngày nghỉ kết hôn và quy định không trừ vào phép năm theo đúng reference. |
| 5 | B | A | tie | False | A: 0.0, B: 0.5 | A: 0.5, B: 0.0 | Cả hai câu trả lời đều chưa chuẩn xác thẩm quyền duyệt mua sắm; vị trí xuất hiện đầu tiên được ưu tiên ở mỗi pass. |
| 12 | A | A | A | True | A: 1.0, B: 0.7 | A: 1.0, B: 0.8 | Candidate A nêu rõ ràng mốc điều kiện thâm niên 6 tháng và mức thưởng chi tiết hơn candidate B. |
| 21 | A | B | tie | False | A: 1.0, B: 0.8 | A: 0.6, B: 1.0 | Hai ứng viên có chất lượng thông tin tương đồng, việc đảo thứ tự làm thay đổi quyết định người thắng. |
| 41 | A | A | A | True | A: 1.0, B: 0.0 | A: 1.0, B: 0.0 | Candidate A cập nhật đúng quy định mới v2024 (15 ngày), trong khi B dùng bản cũ v2023 (12 ngày) nên bị trừ điểm hoàn toàn. |

Position bias count/rate: 2 câu / 40.0% (tỷ lệ 0.400).
Verbosity win rate: 100.0% (tỷ lệ 1.000); decisive cases: 3 câu (các câu ID 1, 12, 41 có quyết định phân thắng bại rõ ràng). Chi tiết verbosity: A thắng và A dài hơn trong 2 trường hợp (ID 12, ID 41); B thắng và B dài hơn trong 1 trường hợp (ID 1).

Giải thích các case không nhất quán và các winner dài hơn:
- Phân tích Position Bias: Có 2 trường hợp (ID 5 và ID 21) không nhất quán về vị trí (position_consistent = False). Ở câu 5, cả hai candidate đều mắc lỗi thiếu thẩm quyền phê duyệt của Tổng Giám đốc đối với tài sản 55 triệu đồng; khi không có câu trả lời nào vượt trội, LLM judge có xu hướng chọn candidate được trình bày trước. Ở câu 21, cả hai câu trả lời đều đáp ứng đúng số ngày phép và khoảng lương Senior, việc hoán vị dẫn đến mỗi lượt ưu tiên một bên. Kỹ thuật swap-and-average đã tự động hòa giải thành kết quả hòa (tie), loại bỏ hoàn toàn hiện tượng thiên vị thứ tự.
- Phân tích Verbosity Bias: Trong cả 3 trường hợp có phân định người thắng (decisive cases: ID 1, 12, 41), câu trả lời dài hơn đều giành chiến thắng (tỷ lệ 100%). Tuy nhiên, khi đối chiếu nội dung ngữ nghĩa với ground truth, câu dài hơn ở đây không phải do dài dòng lan man, mà vì cung cấp đầy đủ các điều kiện ràng buộc cần thiết (ví dụ câu 12 giải thích chi tiết mốc 6 tháng thâm niên; câu 41 giải thích rõ căn cứ chính sách v2024 thay vì chỉ nêu một con số cụ thể). Vì vậy, đây là kết quả đánh giá chất lượng ngữ nghĩa chính xác chứ không phải thiên vị hình thức.

## Calibration đúng model_answer

Helper đánh giá đúng question/model_answer đang có trong fixtures/human_labels_10q.json; không lấy answer mới của pipeline. Prompt chỉ nhận question, answer, reference. Không gửi human_label, human_note hoặc phân bố nhãn human vào prompt.

Bảng đối chiếu 10 câu calibration giữa quyết định của Judge và Human Label:

| Question ID | Model Answer Gốc | Judge Label | Human Label | Agree | Tóm tắt lý do của Judge |
|---|---|---|---|---|---|
| 1 | Được nghỉ 3 ngày và không bị trừ vào ngày phép năm. | 1 | 1 | True | Câu trả lời chính xác theo thông tin trong tài liệu tham khảo, nêu rõ số ngày nghỉ và điều kiện không trừ vào phép năm. |
| 5 | Cần Giám đốc phòng ban phê duyệt. | 0 | 0 | True | Câu trả lời sai vì cần Tổng Giám đốc phê duyệt chứ không phải Giám đốc phòng ban. |
| 12 | Nhân viên chính thức có từ 6 tháng thâm niên trở lên được thưởng 1 tháng lương. | 1 | 1 | True | Câu trả lời chính xác theo reference, nêu rõ điều kiện và mức thưởng cho nhân viên chính thức có từ 6 tháng trở lên. |
| 21 | Nhân viên Senior 9 năm được 16 ngày phép và lương từ 35-50 triệu. | 1 | 1 | True | Câu trả lời chính xác theo reference, nêu rõ số ngày phép và khoảng lương cho nhân viên Senior với 9 năm thâm niên. |
| 23 | Phải hoàn trả 25 triệu chi phí đào tạo. | 1 | 1 | True | Câu trả lời chính xác theo reference, nêu rõ nhân viên phải hoàn trả 25 triệu do chưa đủ 1 năm cam kết. |
| 29 | Phạt 100.000 VNĐ cho mỗi ngày quá hạn sau 30 ngày. | 0 | 0 | True | Câu trả lời sai vì không đề cập đến sự cần thiết có sự phê duyệt của Kế toán trưởng bên cạnh Trưởng phòng và sai số ngày phạt. |
| 33 | Tổng phụ cấp 3.000.000 VNĐ/tháng và 18 ngày phép năm. | 1 | 1 | True | Câu trả lời cung cấp đúng số ngày phép và tổng phụ cấp hàng tháng theo thông tin trong reference. |
| 41 | Được 12 ngày phép năm theo quy định. | 0 | 0 | True | Câu trả lời sai vì nó dựa trên chính sách cũ (v2023) thay vì chính sách hiện hành (v2024), trong đó quy định 15 ngày phép. |
| 46 | Nhân viên thử việc không có phép năm, phải xin nghỉ không lương và có phê duyệt của Trưởng phòng. | 1 | 1 | True | Câu trả lời chính xác theo reference, nêu rõ nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương. |
| 50 | Có thể sử dụng VPN cá nhân để tăng cường bảo mật. | 0 | 0 | True | Câu trả lời sai vì cho phép sử dụng VPN cá nhân trong khi chính sách nội bộ cấm điều này. |

Cohen’s κ: 1.0 (1.0000).
Observed/chance agreement và diễn giải:
- Tỷ lệ đồng thuận quan sát được: P_o = 10 / 10 = 1.0000 (100% các câu hỏi judge đưa ra nhãn trùng khớp hoàn toàn với human label).
- Tỷ lệ đồng thuận ngẫu nhiên: Tập nhãn gồm 5 nhãn tích cực (1) và 5 nhãn tiêu cực (0). Xác suất ngẫu nhiên: P_e = (0.5 * 0.5) + (0.5 * 0.5) = 0.5000.
- Hệ số Cohen's Kappa: κ = (P_o - P_e) / (1 - P_e) = (1.0 - 0.5) / (1.0 - 0.5) = 1.0000.
- Diễn giải: Đạt mức tương đồng hoàn hảo tuyệt đối (Near-perfect / Perfect agreement), vượt trội so với ngưỡng khuyến nghị tiêu chuẩn trong công nghiệp (κ ≥ 0.60).

## Giới hạn và cách dùng judge

Mô hình và prompt:
Mô hình gpt-4o-mini được áp dụng với prompt đánh giá nhị phân nghiêm ngặt (chỉ chấp nhận nhãn 1 khi câu trả lời chính xác, đầy đủ điều kiện và không vi phạm phiên bản chính sách). Kết quả calibration 10 câu thể hiện mô hình có năng lực thẩm định tốt và nhận biết chính xác các lỗi sai tinh vi (ví dụ phát hiện lỗi phiên bản v2023 ở câu 41, lỗi sai hạn mức duyệt ở câu 5, và lỗi vi phạm an toàn thông tin ở câu 50).

Giới hạn quy mô:
Bộ dữ liệu gồm 5 cặp A/B và 10 mẫu calibration là quy mô kiểm chuẩn ban đầu (baseline testing). Với tập mẫu nhỏ, chỉ số κ = 1.0 phản ánh tính đồng thuận cao trên tập quy định cụ thể, nhưng chưa bao quát hết mọi biến thể ngữ nghĩa phức tạp trong môi trường vận hành thực tế.

Cách sử dụng trong production:
Không xem LLM Judge là nguồn chân lý duy nhất (absolute ground truth). Trong hệ thống thực tế:
1. Luôn bắt buộc áp dụng kỹ thuật swap-and-average để triệt tiêu position bias (vốn chiếm tới 40% trong thử nghiệm này).
2. Khi judge trả về kết quả hòa (tie) hoặc độ tin cậy thấp, hệ thống phải tự động kích hoạt luồng kiểm định con người (human-in-the-loop).
3. Định kỳ chạy lại calibration trên các batch dữ liệu mới để giám sát độ trôi (drift) của mô hình judge theo thời gian.
