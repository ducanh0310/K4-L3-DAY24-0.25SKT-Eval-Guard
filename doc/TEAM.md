# Nhóm thực hiện K4 Track 3B

Tên nhóm: 0.25SKT

Repo chung: https://github.com/ducanh0310/K4-L3-DAY24-0.25SKT-Eval-Guard.git

Trưởng nhóm/người nộp: Nguyễn Đức Anh - 2A202602888

Ngày lab/deadline áp dụng: 2026-10-13 23:59 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: f2c781ef285ad7d2e18ce7874178e3bb02fb9a29

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.
Thêm/xóa rows để khớp nhóm thật; bài làm thực hiện độc lập một mình theo diện solo.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Nguyễn Đức Anh | 2A202602888 | Trưởng nhóm & thành viên duy nhất: thực hiện trọn vẹn toàn bộ các Phase A (RAGAS), Phase B (LLM Judge), Phase C (Guardrails) và tích hợp pipeline M1-M5 | artifacts/answers_50q.json, artifacts/ragas_50q.json, artifacts/judge_results.json, artifacts/guard_results.json, doc/reports/*.md |

Người tích hợp/chạy batch/nộp link: Nguyễn Đức Anh (MSSV 2A202602888) thực hiện toàn bộ việc tích hợp, chạy trọn vẹn các batch live experiments và nộp link repository.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows 11 Pro 64-bit (build 22631), Python 3.11.9, MSVC v143 runtime / pre-compiled wheels (Annoy 1.17.3 wheel cài thành công).
Virtualenv/dependencies/imports: Môi trường ảo `.venv` cấu hình đầy đủ các gói thư viện từ `requirements/constraints-core.txt` và `requirements/requirements.txt` (presidio-analyzer, presidio-anonymizer, nemoguardrails 0.9.0, spacy en_core_web_lg, ragas, langchain-openai, qdrant-client). Cấu hình `sitecustomize.py` hỗ trợ UTF-8 encoding và nạp trước PyTorch tránh xung đột DLL giữa c10.dll và pyarrow trên Windows. Lệnh `pip check` trả về 0 xung đột dependency.
Models/Qdrant/50 answers đã chuẩn bị: Qdrant Docker container `infra-qdrant-1` hoạt động trên port 6333; chạy script `scripts/setup_answers.py --live` sinh đủ 50 câu trả lời RAG thật với `gpt-4o-mini-2024-07-18` và `text-embedding-ada-002` vào `artifacts/answers_50q.json` đạt chuẩn 100%. Runtime NeMo Guardrails chạy thực tế bằng `LLMRails` với engine OpenAI (`text-embedding-ada-002` và `gpt-4o-mini`) trong `guardrails/config.yml`.
Nếu cài full NeMo chưa thành công, ghi lỗi thực tế; không dùng unit test pass hoặc pip resolve để khẳng định runtime đã chạy: NeMo Guardrails đã cài đặt và chạy thành công trên runtime thực tế với `LLMRails` cấu hình OpenAI embedding để tránh lỗi kết nối mạng TLS khi tải model từ Hugging Face.

## INDIVIDUAL

Sao chép mẫu cho mỗi thành viên; tiêu đề họ tên không dấu-MSSV.

### NguyenDucAnh-2A202602888

- Phần được giao và phần thực tế đã làm: Do là nhóm làm độc lập 1 người, thực hiện toàn diện tất cả các hạng mục:
  1) Tích hợp 5 modules pipeline (`m1_chunking.py`, `m2_search.py`, `m3_rerank.py`, `m4_eval.py`, `m5_enrichment.py`).
  2) Triển khai `src/phase_a_ragas.py` (Tasks 1-4) tính điểm RAGAS 50 câu, phân nhóm phân phối, phát hiện bottom 10 và failure clustering matrix.
  3) Triển khai `src/phase_b_judge.py` (Tasks 5-8) xây dựng pairwise LLM judge với kỹ thuật swap-and-average, phát hiện position bias, verbosity bias và tính Cohen's kappa cho 10 mẫu calibration.
  4) Triển khai `src/phase_c_guard.py` (Tasks 9-12) tích hợp Presidio PII detection (VN CCCD, phone, email) kết hợp NeMo Guardrails chặn jailbreak/prompt injection, kiểm tra an toàn output, và đo benchmark độ trễ sequential layer P50/P95/P99.
- Bằng chứng: Toàn bộ git commits trên nhánh `main`, các file mã nguồn hoàn thiện trong `src/`, 4 artifact JSON (`artifacts/answers_50q.json`, `artifacts/ragas_50q.json`, `artifacts/judge_results.json`, `artifacts/guard_results.json`) và 3 báo cáo chuyên sâu trong `doc/reports/`.
- Cách kiểm tra: Đã chạy `pytest tests/ -q` vượt qua 86/86 test cases, chạy toàn bộ runner live (`scripts/run_experiments.py --phase a --live`, `--phase b --live`, `--phase c --live --latency-samples 20`), và vượt qua bộ kiểm tra toàn diện `python scripts/check_lab.py --with-tests`.
- Điều đã học: Hiểu sâu sắc sự khác biệt giữa offline mock test và live production constraints; nắm vững các kỹ thuật hạn chế thiên vị của LLM Judge (position bias, verbosity bias) bằng swap-and-average; cơ chế kiểm định độ đồng thuận Cohen's Kappa; giải pháp bảo vệ 2 lớp (input sanitization qua Presidio PII kết hợp NeMo Guardrails topical/jailbreak rails); và cách tối ưu độ trễ P95 trong hệ thống real-time AI.
- Giới hạn/cách xử lý: Xử lý triệt để xung đột DLL giữa PyTorch c10.dll và pyarrow trên nền tảng Windows thông qua `sitecustomize.py`; giải quyết bài toán false positive của mô hình spaCy English đối với từ ngữ tiếng Việt bằng cách cấu hình danh mục thực thể mục tiêu trọng tâm (`VN_CCCD`, `VN_PHONE`, `EMAIL_ADDRESS`, `PHONE_NUMBER`), giúp đạt 20/20 attacks blocked và 8/8 clean allowed; khắc phục sự cố timeout HuggingFace qua việc chuyển sang embedding provider OpenAI.
- Nguồn/AI hỗ trợ nếu có: Sử dụng AI coding assistant hỗ trợ sinh và rà soát cấu trúc boilerplate code; tự tay kiểm chứng từng dòng logic, chạy trực tiếp test suite pytest và đối soát thủ công từng mẫu kết quả trên file artifact JSON.

## Cập nhật sau khi nộp

Bản nộp ban đầu hoàn chỉnh trước thời hạn; không có cập nhật phát sinh sau deadline.
