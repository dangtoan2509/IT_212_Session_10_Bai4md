# Bài 4: Xây Dựng Yêu Cầu Phi Chức Năng - NFR

## Phần 1: Prompt Đã Thiết Kế

```
=== PROMPT — Đặc Tả Yêu Cầu Phi Chức Năng (NFR) cho Guai-api ===

【Vai trò】
Bạn là một Solutions Architect / Performance Engineer với hơn 10 năm kinh nghiệm 
thiết kế kiến trúc hệ thống thương mại điện tử có lưu lượng truy cập cao (high-traffic). 
Bạn thành thạo về tối ưu hiệu năng ứng dụng Spring Boot, MySQL performance tuning, 
và bảo mật JWT. Bạn quen với các tiêu chuẩn SRS IEEE 830 và ISO 25010 
(Software Quality Requirements).

【Mục tiêu】
Viết phần Đặc tả Yêu cầu Phi chức năng (Non-Functional Requirements - NFR) hoàn chỉnh 
cho hệ thống Guai-api (Shop AI), chuẩn bị cho các sự kiện có traffic lớn 
(flash sale, campaign marketing). Tài liệu phải có chỉ tiêu kỹ thuật cụ thể, 
đo lường được (measurable), và khả thi (feasible) để đội DevOps/DBA có thể 
triển khai trực tiếp.

【Ngữ cảnh kỹ thuật】
- Hệ thống: Guai-api — Backend RESTful API cho sàn thương mại điện tử Shop AI
- Tech Stack: Spring Boot 3.x, Java 17, Spring Security, Spring Data JPA
- Database: MySQL 8.0, InnoDB engine
- Authentication: JWT (Access Token + Refresh Token)
- Deployment: Docker containers trên AWS (EC2/ECS), phía trước có Nginx reverse proxy
- Quy mô dự kiến: 
  + Ngày thường: ~5,000 concurrent users
  + Sự kiện flash sale: ~50,000 concurrent users (peak)
  + Catalog: ~500,000 sản phẩm
  + Database size: ~50GB (dự kiến tăng trưởng 20%/năm)

【Yêu cầu bắt buộc — 3 khía cạnh NFR】

▶ KHÍA CẠNH 1: Thời gian phản hồi (Response Time) — Endpoint tra cứu sản phẩm
Phân tích và đưa ra chỉ tiêu kỹ thuật cho:
- Response time tối đa (P95, P99) cho các API:
  + GET /api/v1/products (danh sách, có pagination)
  + GET /api/v1/products/{id} (chi tiết sản phẩm)
  + GET /api/v1/products/search?keyword=... (tìm kiếm full-text)
- Throughput (requests per second) yêu cầu tối thiểu
- Chiến lược caching (Redis/CDN) nếu cần
- Phân biệt chỉ tiêu giữa ngày thường và sự kiện flash sale

▶ KHÍA CẠNH 2: Hiệu suất cơ sở dữ liệu MySQL
Phân tích và đề xuất chiến lược Indexing chi tiết cho:
- Bảng `products` (500K+ records): Index nào cần tạo? Composite index hay single-column?
- Bảng `orders` và `order_items`: Tối ưu query cho báo cáo và thống kê
- Query optimization: Slow query threshold, explain plan monitoring
- Connection pooling configuration (HikariCP)
- Chiến lược Read Replica nếu cần thiết

▶ KHÍA CẠNH 3: Độ trễ và an toàn bảo mật JWT
Phân tích và đưa ra chỉ tiêu cho:
- Thời gian xử lý JWT validation (overhead trên mỗi request)
- Thời hạn tối ưu cho Access Token và Refresh Token
- Chiến lược Token Revocation (blacklist) và hiệu năng lookup
- Bảo vệ chống Token Theft, Replay Attack
- Rate limiting cho authentication endpoints

【Định dạng đầu ra】
- Mỗi NFR phải có mã định danh: NFR-PERF-xxx, NFR-DB-xxx, NFR-SEC-xxx
- Trình bày dưới dạng Markdown Table với các cột: Mã, Tên, Mô tả, Chỉ tiêu kỹ thuật, 
  Phương pháp đo lường, Mức ưu tiên
- Có phần đề xuất chiến lược Indexing dưới dạng SQL statements
- Có phần tóm tắt kiến trúc (Architecture Notes) cho mỗi khía cạnh

=== KẾT THÚC PROMPT ===
```

---

## Phần 2: Tài Liệu NFR Do AI Sinh Ra

### 📋 KHÍA CẠNH 1: THỜI GIAN PHẢN HỒI (RESPONSE TIME)

#### Bảng NFR — Performance

| Mã | Tên yêu cầu | Mô tả chi tiết | Chỉ tiêu kỹ thuật | Phương pháp đo lường | Ưu tiên |
|---|---|---|---|---|---|
| NFR-PERF-001 | Response Time — Danh sách sản phẩm | API `GET /api/v1/products` trả về danh sách sản phẩm có phân trang (default 20 items/page) phải đáp ứng trong thời gian quy định. | **Ngày thường:** P95 ≤ 200ms, P99 ≤ 500ms | Load test bằng JMeter/Gatling với 5,000 concurrent users. Đo từ lúc Nginx nhận request đến khi client nhận response. | 🔴 Critical |
| | | | **Flash sale:** P95 ≤ 500ms, P99 ≤ 1,000ms | Load test với 50,000 concurrent users. | |
| NFR-PERF-002 | Response Time — Chi tiết sản phẩm | API `GET /api/v1/products/{id}` trả về thông tin chi tiết một sản phẩm (bao gồm hình ảnh, đánh giá, thông số). | **Ngày thường:** P95 ≤ 150ms, P99 ≤ 300ms | Load test tương tự NFR-PERF-001. Cache hit ratio phải ≥ 90%. | 🔴 Critical |
| | | | **Flash sale:** P95 ≤ 300ms, P99 ≤ 700ms | | |
| NFR-PERF-003 | Response Time — Tìm kiếm sản phẩm | API `GET /api/v1/products/search` hỗ trợ full-text search trên tên, mô tả sản phẩm. | **Ngày thường:** P95 ≤ 500ms, P99 ≤ 1,000ms | Load test với dataset 500K products. Đo riêng query time và total response time. | 🟠 High |
| | | | **Flash sale:** P95 ≤ 1,000ms, P99 ≤ 2,000ms | | |
| NFR-PERF-004 | Throughput tối thiểu | Hệ thống phải xử lý được số lượng request tối thiểu mỗi giây mà không degradation. | **Ngày thường:** ≥ 1,000 RPS (tổng tất cả product APIs) | Đo bằng Prometheus + Grafana dashboard. | 🔴 Critical |
| | | | **Flash sale:** ≥ 5,000 RPS | | |
| NFR-PERF-005 | Caching Strategy | Áp dụng Redis caching cho dữ liệu sản phẩm thường xuyên truy cập để giảm tải database. | **Cache TTL:** Product list = 5 phút, Product detail = 10 phút, Search result = 2 phút. **Cache hit ratio mục tiêu:** ≥ 85% | Monitor qua Redis INFO stats và custom metrics. Cache invalidation khi product update. | 🟠 High |
| NFR-PERF-006 | CDN cho tài nguyên tĩnh | Hình ảnh sản phẩm, CSS, JS phải được phục vụ qua CDN (CloudFront). | **Cache hit ratio:** ≥ 95%. **TTFB từ CDN:** ≤ 50ms (cùng region). | CloudFront analytics dashboard. | 🟡 Medium |

#### 🏗️ Architecture Notes — Performance

```
                    ┌─────────────┐
                    │   CloudFront│ ← Hình ảnh, static assets (NFR-PERF-006)
                    │     CDN     │
                    └──────┬──────┘
                           │
┌──────────┐      ┌────────┴────────┐      ┌──────────────┐
│  Client  │─────→│  Nginx (LB)     │─────→│  Spring Boot │
│          │      │  Rate Limiting   │      │  Application │
└──────────┘      └─────────────────┘      └──────┬───────┘
                                                   │
                                          ┌────────┴────────┐
                                          │                  │
                                   ┌──────┴──────┐   ┌──────┴──────┐
                                   │    Redis    │   │   MySQL     │
                                   │   Cache     │   │  (Primary)  │
                                   │ (TTL-based) │   │             │
                                   └─────────────┘   └──────┬──────┘
                                                            │
                                                     ┌──────┴──────┐
                                                     │   MySQL     │
                                                     │  (Replica)  │
                                                     └─────────────┘
```

---

### 📋 KHÍA CẠNH 2: HIỆU SUẤT CƠ SỞ DỮ LIỆU MYSQL

#### Bảng NFR — Database Performance

| Mã | Tên yêu cầu | Mô tả chi tiết | Chỉ tiêu kỹ thuật | Phương pháp đo lường | Ưu tiên |
|---|---|---|---|---|---|
| NFR-DB-001 | Query Response Time | Mọi query truy vấn sản phẩm phải hoàn thành trong thời gian quy định. | **Single record lookup:** ≤ 10ms. **List query (paginated):** ≤ 50ms. **Full-text search:** ≤ 200ms. **Slow query threshold:** > 500ms → ghi log cảnh báo. | Enable MySQL slow query log (`long_query_time = 0.5`). Monitor qua Percona PMM hoặc MySQL Enterprise Monitor. | 🔴 Critical |
| NFR-DB-002 | Connection Pooling | HikariCP connection pool phải được cấu hình tối ưu cho tải peak. | **Pool size:** minimum = 10, maximum = 50. **Connection timeout:** 30 giây. **Idle timeout:** 10 phút. **Max lifetime:** 30 phút. | Monitor `HikariPoolMXBean` metrics. Alert khi active connections > 80% max pool size. | 🔴 Critical |
| NFR-DB-003 | Read Replica Strategy | Tách biệt read/write queries để giảm tải Primary DB trong sự kiện flash sale. | **Read queries** (product listing, search, detail) → Read Replica. **Write queries** (order, cart, payment) → Primary. **Replication lag tối đa:** ≤ 1 giây. | Monitor `Seconds_Behind_Master` metric. Alert khi lag > 1s. | 🟠 High |
| NFR-DB-004 | Query Optimization | Mọi query truy vấn sản phẩm phải sử dụng index. Không cho phép full table scan trên bảng > 10K records. | **Index hit ratio:** ≥ 99%. **Không có query nào `type = ALL`** trong EXPLAIN output cho bảng products. | Chạy `EXPLAIN ANALYZE` định kỳ cho top 20 queries. CI/CD gate kiểm tra EXPLAIN plan. | 🟠 High |

#### 📊 Chiến Lược Indexing — SQL Statements

```sql
-- ═══════════════════════════════════════════════════════════════
-- BẢNG PRODUCTS (500K+ records)
-- ═══════════════════════════════════════════════════════════════

-- Index 1: Tra cứu sản phẩm theo danh mục + trạng thái (dùng cho product listing)
-- Use case: GET /api/v1/products?categoryId=5&status=ACTIVE&page=1&size=20
CREATE INDEX idx_products_category_status 
ON products (category_id, status, created_at DESC);

-- Index 2: Tra cứu sản phẩm theo giá (dùng cho bộ lọc giá)
-- Use case: GET /api/v1/products?minPrice=100&maxPrice=500
CREATE INDEX idx_products_price_status 
ON products (status, price, category_id);

-- Index 3: Full-text search trên tên và mô tả sản phẩm
-- Use case: GET /api/v1/products/search?keyword=áo thun
ALTER TABLE products 
ADD FULLTEXT INDEX idx_products_fulltext (name, description);

-- Index 4: Tra cứu sản phẩm theo slug (URL SEO-friendly)
-- Use case: GET /api/v1/products/slug/ao-thun-cotton-xanh
CREATE UNIQUE INDEX idx_products_slug 
ON products (slug);

-- Index 5: Sắp xếp sản phẩm theo đánh giá và doanh số
-- Use case: GET /api/v1/products?sort=rating,desc
CREATE INDEX idx_products_rating_sales 
ON products (status, average_rating DESC, total_sales DESC);

-- ═══════════════════════════════════════════════════════════════
-- BẢNG ORDERS
-- ═══════════════════════════════════════════════════════════════

-- Index 6: Tra cứu đơn hàng theo user (dùng cho "Đơn hàng của tôi")
CREATE INDEX idx_orders_user_created 
ON orders (user_id, created_at DESC);

-- Index 7: Tra cứu đơn hàng theo trạng thái (dùng cho admin dashboard)
CREATE INDEX idx_orders_status_created 
ON orders (status, created_at DESC);

-- Index 8: Báo cáo doanh thu theo thời gian
CREATE INDEX idx_orders_date_status 
ON orders (order_date, status, total_amount);

-- ═══════════════════════════════════════════════════════════════
-- BẢNG ORDER_ITEMS
-- ═══════════════════════════════════════════════════════════════

-- Index 9: Tra cứu items theo order
CREATE INDEX idx_order_items_order 
ON order_items (order_id);

-- Index 10: Thống kê sản phẩm bán chạy
CREATE INDEX idx_order_items_product 
ON order_items (product_id, quantity);

-- ═══════════════════════════════════════════════════════════════
-- BẢNG CART_ITEMS
-- ═══════════════════════════════════════════════════════════════

-- Index 11: Tra cứu giỏ hàng theo user + product (unique constraint)
CREATE UNIQUE INDEX idx_cart_user_product 
ON cart_items (user_id, product_id);
```

---

### 📋 KHÍA CẠNH 3: ĐỘ TRỄ VÀ AN TOÀN BẢO MẬT JWT

#### Bảng NFR — Security & JWT Performance

| Mã | Tên yêu cầu | Mô tả chi tiết | Chỉ tiêu kỹ thuật | Phương pháp đo lường | Ưu tiên |
|---|---|---|---|---|---|
| NFR-SEC-001 | JWT Validation Overhead | Thời gian xử lý JWT validation (parse, verify signature, check claims) trên mỗi request phải ở mức tối thiểu. | **Overhead per request:** ≤ 5ms (P95). **Thuật toán ký:** HMAC-SHA256 (nhanh) hoặc RSA-256 (an toàn hơn, chấp nhận ≤ 10ms). | Đo bằng Spring Boot Actuator custom metrics. Profile bằng `JwtAuthenticationFilter` timing. | 🔴 Critical |
| NFR-SEC-002 | Token Lifetime Configuration | Cấu hình thời hạn token tối ưu giữa bảo mật và trải nghiệm người dùng. | **Access Token TTL:** 15 phút (900 giây). **Refresh Token TTL:** 7 ngày (604,800 giây). **Refresh Token Rotation:** Enabled (mỗi lần refresh cấp Refresh Token mới). | Monitor số lượng token refresh per session. Alert nếu refresh rate bất thường (> 100 refresh/user/ngày). | 🔴 Critical |
| NFR-SEC-003 | Token Blacklist Performance | Kiểm tra token blacklist (đã logout) phải nhanh, không trở thành bottleneck. | **Lookup time:** ≤ 2ms (P95). **Storage:** Redis SET với TTL = remaining token lifetime. **Max blacklist size:** ~100K tokens (auto-expire). | Monitor Redis `GET` latency cho blacklist keys. | 🟠 High |
| NFR-SEC-004 | Token Theft Protection | Bảo vệ chống lại các tấn công đánh cắp token (Token Theft, Replay Attack). | **Biện pháp bắt buộc:** 1) HTTPS only (HSTS header). 2) Token binding với IP + User-Agent fingerprint. 3) Refresh Token Rotation + Automatic Reuse Detection. 4) Secure, HttpOnly cookie cho Refresh Token. | Penetration testing hàng quý. Automated security scan bằng OWASP ZAP. | 🔴 Critical |
| NFR-SEC-005 | Rate Limiting — Auth Endpoints | Giới hạn tần suất request đến các endpoint xác thực để chống Brute Force và DDoS. | **POST /api/v1/auth/login:** Max 10 req/phút/IP. **POST /api/v1/auth/refresh:** Max 30 req/phút/user. **POST /api/v1/auth/register:** Max 3 req/giờ/IP. | Monitor qua Nginx `limit_req_zone` hoặc Spring bucket4j. Alert khi rate limit triggered > 100 lần/giờ. | 🔴 Critical |
| NFR-SEC-006 | JWT Payload Minimization | JWT payload phải chứa thông tin tối thiểu cần thiết để giảm kích thước token và giảm rủi ro Information Disclosure. | **Payload tối đa:** ≤ 500 bytes. **Claims bắt buộc:** sub (userId), roles, iat, exp. **Không chứa:** email, fullName, phone hoặc bất kỳ PII nào. | Decode JWT bằng jwt.io và kiểm tra payload size. CI/CD gate reject token > 500 bytes. | 🟠 High |

#### 🏗️ Architecture Notes — JWT Security

```
Request Flow với JWT Validation:

Client ──HTTPS──→ Nginx ──→ JwtAuthenticationFilter ──→ Controller
                              │
                              ├── 1. Extract token từ header (< 1ms)
                              ├── 2. Check blacklist (Redis GET) (≤ 2ms) [NFR-SEC-003]
                              ├── 3. Validate signature (HMAC-SHA256) (≤ 3ms) [NFR-SEC-001]  
                              ├── 4. Check expiration (< 1ms)
                              ├── 5. Extract claims & set SecurityContext (< 1ms)
                              │
                              └── Total overhead: ≤ 5ms (P95)

Token Theft Detection (Refresh Token Rotation):

1. User đăng nhập → Nhận AT₁ + RT₁
2. AT₁ hết hạn → Gửi RT₁ để refresh → Nhận AT₂ + RT₂ (RT₁ bị vô hiệu)
3. Nếu attacker dùng RT₁ (đã bị vô hiệu) → PHÁT HIỆN REPLAY ATTACK
   → Vô hiệu hóa TOÀN BỘ token family → Buộc user đăng nhập lại
   → Ghi Security Alert log
```

---

### 📋 BẢNG TỔNG HỢP TOÀN BỘ NFR

| Mã | Khía cạnh | Tên | Chỉ tiêu chính | Ưu tiên |
|---|---|---|---|---|
| NFR-PERF-001 | Performance | Response Time — Product List | P95 ≤ 200ms (thường), P95 ≤ 500ms (peak) | 🔴 Critical |
| NFR-PERF-002 | Performance | Response Time — Product Detail | P95 ≤ 150ms (thường), P95 ≤ 300ms (peak) | 🔴 Critical |
| NFR-PERF-003 | Performance | Response Time — Search | P95 ≤ 500ms (thường), P95 ≤ 1000ms (peak) | 🟠 High |
| NFR-PERF-004 | Performance | Throughput | ≥ 1,000 RPS (thường), ≥ 5,000 RPS (peak) | 🔴 Critical |
| NFR-PERF-005 | Performance | Redis Caching | Cache hit ratio ≥ 85% | 🟠 High |
| NFR-PERF-006 | Performance | CDN Static Assets | Cache hit ratio ≥ 95%, TTFB ≤ 50ms | 🟡 Medium |
| NFR-DB-001 | Database | Query Response Time | Single lookup ≤ 10ms, List ≤ 50ms | 🔴 Critical |
| NFR-DB-002 | Database | Connection Pooling | Pool max = 50, timeout = 30s | 🔴 Critical |
| NFR-DB-003 | Database | Read Replica | Replication lag ≤ 1s | 🟠 High |
| NFR-DB-004 | Database | Query Optimization | Index hit ratio ≥ 99% | 🟠 High |
| NFR-SEC-001 | Security | JWT Validation Overhead | ≤ 5ms per request (P95) | 🔴 Critical |
| NFR-SEC-002 | Security | Token Lifetime | AT = 15min, RT = 7 days | 🔴 Critical |
| NFR-SEC-003 | Security | Token Blacklist | Lookup ≤ 2ms (P95) | 🟠 High |
| NFR-SEC-004 | Security | Token Theft Protection | HTTPS + Rotation + Reuse Detection | 🔴 Critical |
| NFR-SEC-005 | Security | Rate Limiting | Login: 10 req/min/IP | 🔴 Critical |
| NFR-SEC-006 | Security | JWT Payload Size | ≤ 500 bytes | 🟠 High |
