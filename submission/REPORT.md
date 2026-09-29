# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Viết Hoàng Hải
- **MSSV:** 2A202602967
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/nguyenviethoanghai/K4-L3-DAY13-NguyenVietHoangHai-2A202602967-Monitoring-LLMOps
- **Commit SHA cuối:**
- **Challenge ID:**
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602967`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 80/100 | 100/100 | Đạt toàn bộ 4 tiêu chí JSON schema, Correlation ID, Log enrichment, PII scrubbing |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Hợp lệ 6/6 panel contract theo dashboard.yaml |
| `pytest` | 22 passed | 22 passed | 100% unit tests pass |
| Số traces hợp lệ | 0 | 10 | Tối thiểu 10 traces trên Langfuse với đầy đủ root/child spans |
| Số PII leak | 0 | 0 | Đã scrub sạch Email, SĐT VN, CCCD, Thẻ thanh toán trước khi render log |
| Latency P95 / TTFT P95 | ~1590ms / ~50ms | ~530ms / ~50ms | P95 latency ổn định dưới ngưỡng SLO (3000ms) |
| Retrieval success rate | 100% | 100% | 10/10 requests truy xuất retrieval thành công |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware`, request trước tiên được `clear_contextvars()` để tránh rò rỉ giữa các request. Sau đó trích xuất `x-request-id` từ HTTP headers nếu có, hoặc tự sinh chuỗi định dạng `req-<8-hex>` (`f"req-{uuid.uuid4().hex[:8]}"`). ID này được bind vào contextvars (`bind_contextvars(correlation_id=correlation_id)`), lưu vào `request.state.correlation_id`, và gán vào header phản hồi (`x-request-id` cùng `x-response-time-ms`).
- **Các metadata được ghi vào structured log:** `user_id_hash` (băm sha256 12 ký tự), `session_id`, `feature`, `model`, `env`, `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Processor `scrub_event` trong `logging_config.py` được đặt trước `JsonlFileProcessor` và `JSONRenderer`. Nó đệ quy quét mọi giá trị chuỗi trong log dictionary và áp dụng regex cho Email, SĐT VN, CCCD, Credit Card để thay thế bằng các token `[REDACTED_...]`.
- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py` đạt điểm 100/100; kiểm tra file `data/logs.jsonl` không còn chứa số điện thoại hoặc email gốc dạng plain text.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Cấu hình `LANGFUSE_PUBLIC_KEY` và `LANGFUSE_SECRET_KEY` từ project `day13-k4-l3a-2A202602967` trong file `.env`. Traces hiển thị trong Langfuse Dashboard với user_id băm và tag tương ứng.
- **Cấu trúc root/retrieval/generation observations:** Root observation `@observe(name="lab-agent-run", as_type="agent")` bọc toàn bộ flow; child observation `@observe(name="retrieval", as_type="retriever")` đo thời gian truy xuất vector store; child observation `@observe(name="generation", as_type="generation")` ghi nhận prompt, model, input/output tokens, cost chi tiết.
- **Cách nối trace với log:** Gắn `correlation_id` vào trace metadata thông qua `propagate_attributes(metadata={"correlation_id": correlation_id})`. Khi phát hiện log lỗi qua `correlation_id`, ta tra cứu trực tiếp ID này trong metadata của Langfuse Trace.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (label: `baseline`, `production`)
- **Version/label candidate:** Version 2 (label: `candidate`)
- **Trace ID của mỗi version:** Ghi nhận từ các request chạy thử nghiệm tương ứng với từng label.
- **Cách promote và rollback `production`:** Vào Langfuse UI -> Prompts -> `day13-chat` -> chọn version -> gán/thay đổi label `production`. Để rollback, chỉ cần chuyển nhãn `production` về Version 1.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** 6 panel theo chuẩn `config/dashboard.yaml` gồm Latency (P50/P95/P99 & TTFT), Traffic (req/min), Errors (error rate & retrieval success), Cost (USD theo thời gian), Tokens (in/out tokens), Quality (heuristic score proxy).
- **SLO và lý do chọn:** `fast_successful_requests` với mục tiêu 99.5% requests thành công và có `latency_ms <= 3000ms` trong chu kỳ 28 ngày. Ngưỡng 3000ms đảm bảo trải nghiệm tương tác trực tuyến cho người dùng và tính toán dựa trên baseline P95 latency thông thường (<600ms).
- **Cách tính error budget:** Error budget = $100\% - 99.5\% = 0.5\%$. Với 100,000 requests trong chu kỳ, hệ thống cho phép tối đa 500 requests bị lỗi hoặc vượt quá 3000ms.
- **Ba alert và runbook tương ứng:**
  1. `high_p95_latency`: Severity Critical, `latency_p95 > 3000ms` trong 5 phút. Runbook tại `docs/alerts.md#alert-1`.
  2. `high_error_rate`: Severity Critical, `error_rate_pct > 2%` trong 5 phút. Runbook tại `docs/alerts.md#alert-2`.
  3. `low_quality_score`: Severity Warning, `quality_score_mean < 0.75` trong 5 phút. Runbook tại `docs/alerts.md#alert-3`.

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** 2026-09-29T09:54:12Z — 2026-09-29T09:54:36Z
- **Triệu chứng từ metrics:** P95 Latency tăng vọt từ ~200ms lên ~2660ms (vượt ngưỡng cảnh báo 2000ms). Trong khi đó, TTFT (Time To First Token) vẫn duy trì ổn định ở mức ~50-68ms, số lượng token và chất lượng câu trả lời không có biến động bất thường.
- **Log line và correlation ID liên quan:**
  ```json
  {"service": "api", "latency_ms": 2656, "ttft_ms": 50, "tokens_in": 34, "tokens_out": 139, "cost_usd": 0.002187, "quality_score": 0.9, "tool_name": "retrieval", "tool_success": true, "payload": {"answer_preview": "Starter answer..."}, "event": "response_sent", "user_id_hash": "aae0b94055a9", "session_id": "k4-l3a-challenge-s02", "model": "claude-sonnet-4-5", "correlation_id": "req-3a8ea469", "env": "dev", "feature": "monitoring", "level": "info", "ts": "2026-09-29T10:04:03.937982Z"}
  ```
  Correlation IDs bị ảnh hưởng: `req-3a8ea469`, `req-c785feea`, `req-7f181637`, `req-47f5ab14`.
- **Trace ID và span gây ảnh hưởng:** Mở trace `9b9fdbdcba9f8f77ee4270489e1dd713` (tương ứng với `req-3a8ea469` / user `aae0b94055a9`) trên Langfuse. Phân tích Waterfall cho thấy:
  - Root observation `lab-agent-run`: tổng thời gian ~2.66s.
  - Child observation `generation`: chỉ mất ~0.15s (hoàn toàn bình thường).
  - Child observation `retrieval`: mất ~2.50s (chiếm ~94% tổng thời gian xử lý request).
- **Root cause:** Độ trễ cao xuất phát từ bước truy xuất ngữ cảnh (`retrieval`) trong module RAG (`mock_rag.py` gặp sự cố `rag_slow` mô phỏng vector database connection latency / disk I/O bottleneck).
- **Fix action:** Tắt sự cố mô phỏng (`python scripts/inject_incident.py --disable`), cấu hình connection pooling và timeout 1500ms cho vector store.
- **Preventive measure:**
  1. Cấu hình Alert `high_p95_latency` khi latency P95 vượt ngưỡng 2000ms trong 5 phút.
  2. Bổ sung In-memory Cache (Redis) cho các query vector retrieval thường gặp để giảm tải cho vector database.
  3. Áp dụng Circuit Breaker pattern: nếu retrieval chậm vượt ngưỡng timeout, tự động fallback sang câu trả lời tổng quát thay vì giữ request chờ lâu.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Đặt processor `scrub_event` duyệt đệ quy ở vị trí đầu chuỗi processor trước khi serialize JSON và ghi file log. Quyết định này triệt để ngăn chặn rò rỉ dữ liệu PII (Email, Phone VN, CCCD, Credit card) ở mọi cấp cấu trúc log mà không làm thay đổi logic xử lý nghiệp vụ của API.
- **Một lỗi/blocker đã gặp:** Quản lý đồng bộ giữa correlation ID trong log và metadata trong Langfuse trace khi gọi qua các sub-functions.
- **Cách tìm nguyên nhân và xử lý:** Sử dụng contextvars (`bind_contextvars` / `propagate_attributes`) để correlation ID tự động kế thừa xuyên suốt từ Middleware đến Root Observation và các Child Spans.
- **Cách hiểu luồng Metrics → Logs → Traces:** 
  1. **Metrics:** Phát hiện triệu chứng vĩ mô và thời điểm bắt đầu sự cố (ví dụ P95 latency tăng vọt > 2000ms).
  2. **Logs:** Sử dụng log để lọc ra request cụ thể bị ảnh hưởng và lấy `correlation_id`.
  3. **Traces:** Dùng `correlation_id` mở Trace chi tiết trên Langfuse để soi từng span con (waterfall), xác định chính xác dòng code/dịch vụ nào (ở đây là span `retrieval`) là nguyên nhân gốc rễ.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Quản lý prompt version và rollback giúp đảm bảo hệ thống có thể khôi phục tức thì khi prompt mới gây suy giảm chất lượng hoặc tăng token bất thường mà không cần redeploy code. Theo dõi token/cost và định nghĩa SLO/Error budget giúp bảo vệ ngân sách vận hành và duy trì chất lượng dịch vụ cam kết với người dùng.
- **Điều quan trọng nhất đã học:** Nắm vững toàn diện quy trình quan sát hệ thống LLMOps hiện đại: Structured Logging, Data Redaction (PII), Tracing phân tán đa tầng, và điều tra sự cố theo chuỗi bằng chứng Metric ➔ Log ➔ Trace.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Không có. Hệ thống đã hoàn thiện 100% tất cả các bài kiểm tra và yêu cầu đặt ra.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
