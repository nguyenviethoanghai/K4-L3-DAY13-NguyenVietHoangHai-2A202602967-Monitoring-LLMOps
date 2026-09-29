# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: high_p95_latency
- Severity: critical
- Duration: 5m
- Kênh thông báo: Slack (#alerts-llmops)
- SLI/SLO liên quan: `fast_successful_requests` (latency P95 <= 3000ms)
- Điều kiện và thời gian duy trì: `latency_p95 > 3000ms` kéo dài liên tục trên 5 phút
- Ảnh hưởng tới người dùng: Người dùng phản hồi chậm hoặc bị timeout khi gửi câu hỏi qua chat API.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra dashboard panel **Latency** để xác định độ trễ tăng ở TTFT hay tổng thời gian phản hồi.
  2. Lọc file `data/logs.jsonl` tìm các request có `latency_ms > 3000`, trích xuất `correlation_id`.
  3. Mở Langfuse trace tương ứng với `correlation_id` để phân tích span waterfall (xem bottleneck tại retrieval hay LLM generation).
- Mitigation tạm thời: Bật cơ chế caching kết quả RAG, giảm kích thước top-k retrieval hoặc chuyển sang fallback model có latency thấp hơn.
- Owner: oncall-llmops

## Alert 2

- Tên: high_error_rate
- Severity: critical
- Duration: 5m
- Kênh thông báo: Slack (#alerts-llmops)
- SLI/SLO liên quan: Guardrail `error_rate_pct_max <= 2%`
- Điều kiện và thời gian duy trì: Tỷ lệ lỗi `error_rate_pct > 2%` trong cửa sổ 5 phút
- Ảnh hưởng tới người dùng: Người dùng nhận thông báo lỗi hệ thống, gián đoạn phiên chat.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra dashboard panel **Errors** để phân loại mã lỗi (`error_type`) và `tool_success_rate`.
  2. Tra cứu structured log cho event `request_failed` trong `data/logs.jsonl` để xem stack trace và error message.
  3. Mở Langfuse trace để kiểm tra bước bị fail (vector store connection timeout, context overflow, v.v.).
- Mitigation tạm thời: Kích hoạt fallback handler trả về câu trả lời mặc định khi tool/retrieval lỗi; restart service nếu xảy ra memory/connection leak.
- Owner: oncall-llmops

## Alert 3

- Tên: low_quality_score
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-llmops)
- SLI/SLO liên quan: Guardrail `quality_score_avg_min >= 0.75`
- Điều kiện và thời gian duy trì: Điểm chất lượng trung bình `quality_score_mean < 0.75` trong 5 phút
- Ảnh hưởng tới người dùng: Câu trả lời kém chất lượng, không đúng trọng tâm hoặc không khớp ngữ cảnh RAG.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra dashboard panel **Quality proxy** và số lượng tài liệu trích xuất (`doc_count`).
  2. Tra cứu logs `response_sent` có `quality_score < 0.75` và lấy `correlation_id`.
  3. Mở trace trên Langfuse để xem prompt template version hiện tại và generation output.
- Mitigation tạm thời: Rollback prompt version trên Langfuse về version ổn định (`production` -> `v1`/`baseline`); kiểm tra kho dữ liệu RAG.
- Owner: llm-prompt-team
