# 🎤 Script Thuyết Trình — Observability trong Microservices
> **Dành cho người thuyết trình.** Mỗi slide gồm: lời nói gợi ý, điểm cần nhấn mạnh, và demo thực tế.

---

## 🏗️ Kiến trúc Hệ thống Hiện tại

### Business Services — Checkout Flow

```
Người dùng
    │
    ▼
┌─────────────────────────────────────┐
│         API Gateway  :8080          │  ← Entry point duy nhất
│   POST /checkout, GET /health       │
└──────────────┬──────────────────────┘
               │ gọi song song
       ┌───────┴────────┐
       ▼                ▼
┌──────────────┐  ┌──────────────────┐
│ Order Service│  │Inventory Service │
│    :8081     │  │      :8083       │
│ (PostgreSQL) │  │ /inventory/      │
└──────┬───────┘  └──────────────────┘
       │ gọi tiếp
       ▼
┌──────────────────┐
│ Payment Service  │
│     :8082        │
└──────┬───────────┘
       │ gọi bank
       ▼
┌──────────────────┐
│  FakeBank API    │
│     :8084        │  ← Chaos: /chaos/slow, /chaos/error
└──────────────────┘
```

### Observability Stack

```
5 Services (expose /metrics + OTLP traces)
       │                    │
       ▼                    ▼
 Prometheus            OTel Collector :4320
 (pull 5s)             (receive gRPC/HTTP)
       │                    │
       ▼                    ▼
   Grafana              Jaeger :16686
   :3000                (in-memory)
   2 dashboards

Stdout JSON → Filebeat → Elasticsearch :9200 → Kibana :5601
```

### Port Map

| Service | Port | Vai trò |
|---------|------|---------|
| API Gateway | 8080 | Entry point |
| Order Service | 8081 | Tạo đơn + PostgreSQL |
| Payment Service | 8082 | Thanh toán, retry |
| Inventory Service | 8083 | Trừ tồn kho |
| FakeBank API | 8084 | Bank giả, chaos endpoint |
| Prometheus | 9090 | Time-series metrics |
| Grafana | 3000 | Dashboard (2 boards) |
| Jaeger | 16686 | Distributed tracing |
| Elasticsearch | 9200 | Log storage |
| Kibana | 5601 | Log search & filter |
| Control Panel | 3100 | Demo UI |

---

## Slide 1 — Vì sao Observability quan trọng trong Microservices?

### 🎤 Lời nói
> *"Hãy tưởng tượng bạn đang vận hành hệ thống e-commerce. Người dùng bấm Checkout — request đó sẽ đi qua Gateway, Order, Payment, Inventory, và Bank API. Chỉ cần Payment chậm 200ms, hoặc DB quá tải — toàn bộ checkout sẽ bị ảnh hưởng. Câu hỏi là: làm sao bạn biết chậm ở đâu?"*

### 💡 Nhấn mạnh
- Monolith: lỗi xảy ra ở **1 chỗ**, debug dễ
- Microservices: lỗi có thể ở **bất kỳ service nào**, hoặc **giữa 2 service**
- Không có observability = **"vận hành trong bóng tối"**

### 🖥️ Demo
1. Mở **Jaeger** → chọn service `gateway` → Find Traces
2. Click vào 1 trace → show span tree: `gateway → order → payment → fake-bank → inventory`
3. **Nói:** *"Đây là toàn bộ hành trình của 1 request. Không có cái này, khi hệ thống chậm bạn không biết phải nhìn vào đâu."*

---

## Slide 2 — 3 Tín hiệu cốt lõi: Metrics – Logs – Traces

### 🎤 Lời nói
> *"Observability có 3 trụ cột. Mỗi cái trả lời một câu hỏi khác nhau. Và chỉ khi kết hợp cả 3, bạn mới thực sự hiểu hệ thống đang làm gì."*

### 💡 Nhấn mạnh
| Tín hiệu | Câu hỏi trả lời | Ví dụ |
|----------|----------------|-------|
| **Metrics** | **Có** vấn đề không? | p95 latency tăng từ 300ms → 2s |
| **Traces** | Vấn đề **ở đâu**? | 80% thời gian nằm ở Payment |
| **Logs** | Vấn đề **vì sao**? | Timeout khi gọi bank API + retry |

> ⚡ *"Metrics cho biết CÓ vấn đề. Traces cho biết Ở ĐÂU. Logs cho biết VÌ SAO."* — **câu này nói chậm, rõ**

### 🖥️ Demo
1. Admin panel → **Run Check** → terminal in ra TraceId
2. Copy TraceId → Kibana: search `@tr: "TRACE_ID"` → thấy logs
3. Mở Jaeger → thấy trace tương ứng
4. **Nói:** *"Cùng 1 request — nhìn từ 3 góc: Metric (latency), Trace (flow), Log (chi tiết)."*

---

## Slide 3 — 4 Golden Signals

### 🎤 Lời nói
> *"Google SRE Book đề xuất: nếu chỉ theo dõi được vài thứ, hãy chọn 4 cái này. Chúng đủ để phát hiện 90% vấn đề sản xuất."*

### 💡 Nhấn mạnh
| Signal | Ý nghĩa | Alert khi |
|--------|---------|-----------|
| **Latency** | Tốc độ phản hồi | p95 > 1.5s |
| **Traffic** | Lưu lượng | Đột biến bất thường |
| **Errors** | Tỉ lệ lỗi | Error rate > 1% |
| **Saturation** | Mức độ bão hòa | CPU > 80%, queue đầy |

> ⚡ *"4 chỉ số này là nền tảng. Nếu không nắm được chúng, KHÔNG nên nói đến tối ưu sâu hơn."*

### 🖥️ Demo
1. Admin → **Start Load Test** (20 req, 300ms delay)
2. Grafana → **Golden Signals dashboard** → chỉ vào 4 panels đang live update
3. **Nói:** *"Mỗi panel là 1 signal. Nhìn vào đây, trong 5 giây tôi biết hệ thống đang healthy hay không."*

---

## Slide 4 — Monitoring Stack: Prometheus + Grafana

### 🎤 Lời nói
> *"Prometheus scrape metrics từ mỗi service mỗi 5 giây và lưu vào time-series database. Grafana đọc từ đó và vẽ biểu đồ. Quan trọng là: thu thập chưa đủ — phải trực quan hóa mới có ý nghĩa."*

### 💡 Nhấn mạnh
- Prometheus **pull-based**: chủ động hỏi service, không phải service đẩy lên
- Grafana: dashboard **không chỉ để nhìn** — để **ra quyết định**
- Dashboard tốt phải có: p95/p99 latency, error rate, RPS, CPU/memory

> ⚡ *"Thu thập là chưa đủ. Phải trực quan hóa và nhìn được xu hướng theo thời gian."*

### 🖥️ Demo
1. Mở **Prometheus** → `/targets` → show 5 service đều `UP`, scrape interval 5s
2. Mở **Grafana** → show dashboard tự động load từ config (provisioning)
3. **Nói:** *"Toàn bộ dashboard này được define bằng code, deploy xong là có luôn, không cần setup tay."*

---

## Slide 5 — Alerting: Ít nhưng chất lượng

### 🎤 Lời nói
> *"Alert là công cụ quan trọng nhất trong vận hành — và cũng là công cụ bị lạm dụng nhất. Alert quá nhiều thì người ta sẽ bỏ qua, mất đi tác dụng. Nguyên tắc: mỗi alert phải có người chịu trách nhiệm và phải dẫn đến hành động cụ thể."*

### 💡 Nhấn mạnh
- **Alert storm:** 1 DB die → 10 service lỗi → 10 alert khác nhau → người on-call bị overwhelmed
- **Alertmanager grouping:** gom thành 1 incident duy nhất
- Alert không nên fire khi: CPU 80% lúc 3am không ảnh hưởng user

> ⚡ *"Alert quá nhiều = không có alert. Chỉ alert khi cần hành động ngay."*

### 🖥️ Demo
1. Admin → **⚡ Slow Bank** (bank sleep 3s)
2. Chạy Load Test 10 req
3. Prometheus → `/alerts` → thấy `CheckoutLatencyHigh` đang **FIRING**
4. **Nói:** *"Alert này có nghĩa: có người phải xử lý ngay. Không phải chỉ để xem."*

---

## Slide 6 — Bẫy High-Cardinality Labels

### 🎤 Lời nói
> *"Prometheus lưu mỗi tổ hợp metric + labels thành 1 time-series riêng. Nếu bạn thêm user_id vào label — 1 triệu user = 1 triệu time-series — Prometheus sẽ hết memory."*

### 💡 Nhấn mạnh
```
# ❌ SAI:
http_requests_total{user_id="u123456"}  ← mỗi user = 1 series mới

# ✅ ĐÚNG:
http_requests_total{service="order", route="/checkout", status="200"}
```
- **High cardinality** = number of unique label values rất lớn
- Tuyệt đối KHÔNG dùng: `user_id`, `email`, `request_id`, `order_id` làm label

> ⚡ *"Không bao giờ đưa user_id, email, request_id vào Prometheus label. Đưa vào Log thì được."*

### 🖥️ Demo
1. Admin → **⚡ High Cardinality**
2. Prometheus → `/metrics` endpoint của fake-bank → show `bank_calls_total{user_id="user-xxxxx"}` với hàng trăm series khác nhau
3. **Nói:** *"Đây là anti-pattern hay gặp nhất trong production. Hệ thống sẽ chậm dần rồi OOM."*

---

## Slide 7 — Logging tập trung: Elasticsearch + Kibana

### 🎤 Lời nói
> *"Trong Kubernetes hoặc Docker, container có thể restart bất kỳ lúc nào — và log mất theo. Giải pháp: ghi log tập trung vào Elasticsearch, xem qua Kibana. Không cần ssh vào từng server."*

### 💡 Nhấn mạnh
- Container log là **ephemeral** — restart là mất
- Không thể `grep` trên 50 container cùng lúc
- Centralized logging + structured format = **có thể filter theo bất kỳ field nào**

> ⚡ *"Trong microservices, không có centralized logging = không thể debug production."*

### 🖥️ Demo
1. Kibana → Discover → `filebeat-*` → Last 15 minutes
2. Search: `fields.service: payment`
3. Expand 1 log entry → show JSON fields
4. **Nói:** *"Log có cấu trúc — tôi có thể filter theo service, level, trace_id, bất cứ thứ gì trong vài giây."*

---

## Slide 8 — Logging Pipeline thực tế

### 🎤 Lời nói
> *"Log text thuần rất khó xử lý tự động. Log JSON cho phép Elasticsearch index từng field — có thể filter, aggregate, visualize ngay. Pipeline của chúng ta: App viết JSON ra stdout → Filebeat thu thập → Elasticsearch → Kibana."*

### 💡 Nhấn mạnh
```json
// ❌ Log text — khó filter
"Payment failed after 3 retries for user abc123"

// ✅ Log JSON — searchable theo mọi field
{
  "service": "payment",
  "level": "ERROR",
  "trace_id": "ce072d63...",
  "message": "Timeout calling bank API",
  "retry_count": 3
}
```

> ⚡ *"Log PHẢI có cấu trúc. Text log chỉ có ích cho người đọc — JSON có ích cho cả người lẫn máy."*

---

## Slide 9 — Distributed Tracing: OpenTelemetry + Jaeger

### 🎤 Lời nói
> *"Distributed tracing theo dõi toàn bộ hành trình của 1 request qua nhiều service. Mỗi bước gọi là 1 Span. Tổng hợp lại thành Trace. Nhìn vào trace, bạn thấy ngay: bottleneck nằm ở đâu, service nào đang chậm."*

### 💡 Nhấn mạnh
```
Trace: Checkout request
├── gateway        50ms
│   ├── order      80ms
│   │   └── payment   1200ms ← BOTTLENECK
│   │       └── fake-bank  1150ms
│   └── inventory  30ms
Total: ~1400ms
```
- **Trace** = hành trình đầy đủ
- **Span** = 1 bước trong hành trình
- **Parent-child relationship** = biết service A gọi service B

> ⚡ *"Không có tracing, debug microservices = đoán mò."*

### 🖥️ Demo
1. Bật **Slow Bank** → chạy 1 checkout
2. Jaeger → tìm trace đó → click vào → show timeline
3. **Chỉ vào span `fake-bank` chiếm 3000ms**
4. **Nói:** *"Không cần xem log, không cần đoán. Một cái nhìn vào đây — Payment → Bank là vấn đề."*

---

## Slide 10 — Context Propagation

### 🎤 Lời nói
> *"Để trace không bị đứt giữa chừng, mỗi service khi nhận request phải đọc header `traceparent` và truyền tiếp sang service tiếp theo. Nếu 1 service quên truyền — trace bị cắt, bạn mất 1 phần picture."*

### 💡 Nhấn mạnh
```
HTTP Header: traceparent: 00-abc123-def456-01
                              ↑         ↑     ↑
                           TraceId   SpanId  Flags
```
- **W3C TraceContext standard** — OpenTelemetry tự động inject/extract
- Nếu không propagate: mỗi service tạo trace riêng → không thấy end-to-end

> ⚡ *"Context propagation là điều kiện bắt buộc — thiếu nó, tracing vô nghĩa."*

---

## Slide 11 — Kết nối Logs và Traces

### 🎤 Lời nói
> *"Đây là kỹ thuật mạnh nhất và hay bị bỏ qua nhất. Khi log có chứa trace_id, bạn có thể nhảy thẳng từ trace sang toàn bộ log của request đó — và ngược lại. Thay vì mất hàng giờ tìm log, bây giờ chỉ mất vài giây."*

### 💡 Nhấn mạnh
- Mỗi log line phải chứa: `trace_id`, `span_id`
- Trong code: inject từ OpenTelemetry context vào Serilog properties
- **Correlation = superpower** của observability

> ⚡ *"Kỹ thuật này rút ngắn thời gian điều tra từ hàng giờ xuống vài phút."*

### 🖥️ Demo
1. Jaeger → copy TraceId của 1 trace (sau khi chạy Slow Bank)
2. Kibana Discover → search: `@tr: "PASTE_TRACE_ID_HERE"`
3. Show toàn bộ log của request đó từ gateway → order → payment
4. **Nói:** *"Một TraceId — xem được toàn bộ log của request đó qua tất cả services. Đây là log-trace correlation."*

---

## Slide 12 — Vận hành theo SLO

### 🎤 Lời nói
> *"Alert kiểu cũ: CPU > 80%. Nhưng CPU 80% trong 3am khi không có traffic — có ảnh hưởng user không? Không. SLO-based alerting tốt hơn: alert dựa trên trải nghiệm người dùng thực sự."*

### 💡 Nhấn mạnh
| Khái niệm | Ý nghĩa | Ví dụ |
|-----------|---------|-------|
| **SLI** | Chỉ số đo | Checkout success rate |
| **SLO** | Mục tiêu | 99.9% trong 30 ngày |
| **Error Budget** | Ngân sách lỗi | 0.1% = ~43 phút/tháng |

```
❌ Alert yếu:  CPU > 80%
✅ Alert tốt:  Checkout success rate < 99.9% trong 10 phút
```

> ⚡ *"Vận hành nên xoay quanh trải nghiệm người dùng, không phải chỉ số kỹ thuật."*

### 🖥️ Demo
1. Grafana → **SLO Dashboard** → show gauge đang xanh (~100%)
2. Admin → **⚡ Drop SLO** — gửi request lỗi
3. Đợi 30s → gauge chuyển đỏ (< 99%)
4. Admin → **Reset** → gauge dần xanh trở lại
5. **Nói:** *"SLO gauge đỏ = người dùng đang bị ảnh hưởng. Đây mới là alert đáng để action."*

---

## Slide 13 — Incident & Postmortem

### 🎤 Lời nói
> *"Sự cố là điều tất yếu — không thể tránh. Điều quan trọng là: phát hiện nhanh, khowanh vùng đúng, giảm thiểu ảnh hưởng, và sau đó không lặp lại. Postmortem tốt không tìm người đổ lỗi — tìm cách cải thiện hệ thống."*

### 💡 Nhấn mạnh — Simulate Incident

| Bước | Action | Công cụ |
|------|--------|---------|
| **Phát hiện** | Alert firing | Prometheus Alertmanager |
| **Khoanh vùng** | Latency tăng ở đâu? | Grafana Golden Signals |
| **Root cause** | Span nào chậm? | Jaeger trace |
| **Xác nhận** | Log nói gì? | Kibana |
| **Khôi phục** | Reset chaos | Admin panel |
| **Verify** | SLO xanh lại | Grafana SLO |

> ⚡ *"Văn hóa không đổ lỗi (blameless postmortem) cực kỳ quan trọng. Mục tiêu là cải thiện hệ thống."*

### 🖥️ Demo — Full incident flow
1. Admin → **⚡ Retry Storm** (payment retry 3x)
2. Load Test 20 req
3. Grafana → thấy **Saturation tăng**, **Latency tăng**
4. Jaeger → thấy nhiều span retry trong payment
5. Kibana → search `fields.service: payment` → thấy log retry
6. Admin → **Reset** → metrics normalize
7. **Nói:** *"Từ phát hiện đến root cause — 60 giây. Không có observability, có thể mất 1-2 giờ."*

---

## Slide 14 — Tổ chức đội ngũ

### 🎤 Lời nói
> *"Conway's Law: kiến trúc hệ thống phản chiếu cấu trúc tổ chức. Microservices thành công khi có 2 loại team phối hợp đúng cách."*

### 💡 Nhấn mạnh
- **Stream-aligned team**: phụ trách 1 domain (Order, Payment, Inventory...)
  - Tự build, deploy, vận hành service của mình
  - "You build it, you run it" — Amazon
- **Platform team**: cung cấp nền tảng observability dùng chung
  - Template log chuẩn, Dashboard Grafana mặc định, OTel Collector sẵn sàng
  - Team feature chỉ cần "cắm vào" là dùng được

> ⚡ *"Platform team giỏi = feature team tập trung vào business logic, không lo infrastructure."*

---

## Slide 15 — Tổng kết Kiến trúc

### 🎤 Lời nói
> *"Đây là toàn bộ kiến trúc observability của hệ thống demo. Mỗi nhánh là 1 trong 3 trụ cột. Tất cả được code hóa — deploy là xong."*

### 💡 Nhấn mạnh
```
Microservices
  ├── Metrics  → Prometheus (scrape 5s) → Grafana (dashboard)
  ├── Logs     → Stdout JSON → Filebeat → Elasticsearch → Kibana
  └── Traces   → OTel SDK → OTel Collector → Jaeger
```

### 🎤 Câu kết
> *"Microservices tăng tính linh hoạt — nhưng đánh đổi bằng độ phức tạp vận hành. Observability không phải là tùy chọn. Đây là điều kiện bắt buộc để hệ thống tồn tại và mở rộng an toàn. Không có nó, bạn đang bay mù."*

---

## ⏱️ Gợi ý thời lượng

| Phần | Slide | Thời gian gợi ý |
|------|-------|----------------|
| Giới thiệu vấn đề | 1-2 | 5 phút |
| Monitoring | 3-4 | 7 phút |
| Alerting & Bẫy | 5-6 | 5 phút |
| Logging | 7-8 | 5 phút |
| Tracing + Correlation | 9-11 | 8 phút |
| SLO + Incident | 12-13 | 8 phút |
| Tổ chức + Tổng kết | 14-15 | 5 phút |
| **Demo live** | — | **7 phút** |
| **Q&A** | — | **5 phút** |
| **Tổng** | | **~50 phút** |

---

## 🔗 Links demo nhanh trong thuyết trình

| Tool | URL public |
|------|-----------|
| **Control Panel** | https://demo.hoaghai.me |
| **Grafana** | https://grafana.hoaghai.me |
| **Jaeger** | https://jaeger.hoaghai.me |
| **Kibana** | https://kibana.hoaghai.me |
| **Prometheus** | https://prometheus.hoaghai.me |
