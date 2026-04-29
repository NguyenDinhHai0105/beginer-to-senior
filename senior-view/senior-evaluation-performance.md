# Hướng dẫn Senior: Đánh giá hệ thống (Backend → Frontend) — Tiêu chí hiệu năng và chất lượng

## Mục đích
Tài liệu này hướng dẫn cách một kỹ sư senior đánh giá toàn diện một hệ thống: từ backend, hạ tầng, tới frontend. Bao gồm các tiêu chí, chỉ số hiệu năng (performance), checklist kiểm tra và cách báo cáo kết quả.

## Tổng quan đánh giá
Mục tiêu đánh giá:
- Correctness: hệ thống hoạt đúng chức năng
- Performance: nhanh, ổn định dưới tải thực tế
- Scalability: có thể mở rộng theo nhu cầu
- Reliability: ít lỗi, phục hồi nhanh
- Maintainability: dễ bảo trì, có test
- Security: tuân thủ các nguyên tắc bảo mật

Kết quả mong muốn: danh sách điểm mạnh/khuyết điểm + đề xuất ưu tiên cải tiến.

---

## Backend
1. Kiến trúc & Thiết kế
- Rõ ràng, tách rời trách nhiệm (separation of concerns)
- Microservices vs monolith: trade-offs, ranh giới dịch vụ rõ
- API design: versioning, pagination, idempotency, hợp đồng (contract)

2. Hiệu năng & Tối ưu
- Latency targets: p50/p95/p99 cho mỗi endpoint
- Throughput (RPS) và khả năng xử lý đồng thời
- DB: schema hợp lý, index đúng chỗ, tránh N+1
- Queries: từng query phải phân tích (EXPLAIN), tránh full table scan
- Caching: layer cache (Redis, in-memory), cache invalidation rõ ràng
- Connection pooling, throttling, backpressure

3. Batching & Background Jobs
- Đẩy tác vụ nặng ra background (queue), retry & dead-letter xử lý
- Idempotence cho job consumer

4. Observability
- Metrics: latency, error rate, saturation, throughput
- Tracing phân tán (ví dụ OpenTelemetry) để root-cause
- Logs có cấu trúc, correlation IDs
- Dashboards & SLO/SLI

5. Testing
- Unit, integration, contract tests (consumer-driven pact nếu cần)
- Performance tests & load tests
- Chaos testing cho trường hợp failure modes

6. Bảo mật
- Authentication, authorization, input validation
- Rate limiting, audit logging
- Secret management, TLS

---

## Hạ tầng & DevOps
- Autoscaling, health checks, graceful shutdown
- Load balancer và sticky session (nếu cần)
- CI/CD: build reproducible, canary/blue-green deployments
- IaC (Terraform/CloudFormation) và versioning
- Backups & DR plan

---

## Frontend
1. Chỉ số hiệu năng người dùng
- FCP, LCP, TTFB, TTI, TBT, CLS
- Perceived performance: skeleton screens, progressive rendering

2. Bundling & Asset
- Bundle size budgets, code-splitting, tree-shaking
- HTTP/2, HTTP/3, compression, preconnect/prefetch
- CDN, cache headers, cache-busting chiến lược

3. Runtime & Rendering
- SSR/SSR-hybrid khi cần (SEO, TTFB)
- Hydration performance và tránh reflows nặng
- Web workers để xử lý tasks nặng

4. Network
- Throttling scenarios: low bandwidth, high-latency
- Graceful fallback khi API chậm/hỏng

5. Observability & Testing
- Real User Monitoring (RUM) + synthetic tests
- E2E tests, visual regression, performance budgets

6. Accessibility & UX
- ARIA, keyboard navigation, color contrast
- Progressive enhancement

---

## Tiêu chuẩn hiệu năng (gợi ý ngưỡng)
Lưu ý: ngưỡng có thể tùy theo domain; đây là tham khảo phổ biến:
- API p95 latency: < 300 ms
- API p99 latency: < 1 s
- Error rate: < 0.1% (sau kiểm tra tải thực tế)
- Throughput: đáp ứng RPS mục tiêu với headroom 2x
- LCP (real users): < 2.5 s
- CLS: < 0.1
- Bundle initial JS: < 200–300 KB gzipped (tùy app)
- First Contentful Paint (FCP): < 1 s

---

## Checklist nhanh (Go/No-Go)
- [ ] Có metrics cho mọi dịch vụ quan trọng (latency, errors, throughput)
- [ ] Tracing xuyên suốt request flow
- [ ] DB queries được profiling & có index
- [ ] Caching đúng chỗ với invalidation strategy
- [ ] Load test trên realistic scenarios
- [ ] CI/CD có smoke test & rollback plan
- [ ] Frontend có performance budgets và RUM
- [ ] Runbooks & alerting rõ ràng

---

## Cách trình bày báo cáo đánh giá
1. Executive summary (vài dòng): trạng thái tổng quan + top 3 rủi ro
2. Dữ liệu: metrics, biểu đồ latency, error rate, throughput
3. Root-causes: những endpoint/queries/asset gây tắc nghẽn
4. Prioritized recommendations: quick wins vs infra/architectural work
5. Plan hành động: owner, estimate, impact

---

## Mẫu tóm tắt (1 trang)
- Hạng mục: Backend / DB / Infra / Frontend
- Trạng thái: OK / Cần chú ý / Critical
- Vấn đề chính: ...
- Gợi ý giải pháp & ưu tiên

---

## Ghi chú cuối
Số liệu luôn quan trọng hơn cảm nhận; đưa ra các phép đo, test reproducible và thresholds rõ ràng. Senior nên tập trung vào root cause, trade-offs, và roadmap cải tiến có thể đo lường được.

