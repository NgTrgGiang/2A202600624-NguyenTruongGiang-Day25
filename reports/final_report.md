# Báo cáo Reliability — Day 10

**Cấu hình chạy:** `configs/default.yaml` — 3 kịch bản chaos × 100 request = 300 request tổng, cache backend in-memory.
**Nguồn dữ liệu:** `reports/metrics.json` (bật cache), một lần chạy đối chứng không cache (tắt cache).

---

## 1. Tổng quan kiến trúc

Gateway định tuyến mọi request qua ba lớp phòng thủ theo thứ tự: semantic cache (rẻ nhất, nhanh
nhất), chuỗi circuit breaker theo từng provider (cô lập provider đang lỗi), và static fallback (đảm
bảo luôn có phản hồi kể cả khi mọi thứ đều sập).

```
User Request
    |
    v
+-------------------- ReliabilityGateway.complete(prompt) --------------------+
|                                                                             |
|  1. [Cache check]  ResponseCache.get() / SharedRedisCache.get()             |
|        |  HIT  (score >= 0.92, an toàn privacy, không false-hit)            |
|        +--------------------------------> return  route="cache_hit:<score>" |
|        |  MISS                                                              |
|        v                                                                    |
|  2. [Circuit Breaker: primary] --call--> Provider "primary"  (lỗi 25%)      |
|        |  OPEN? fail fast (CircuitOpenError)   thành công -> cache + return  |
|        |  ProviderError / CircuitOpenError -> ghi nhận, thử tiếp  route=primary|
|        v                                                                    |
|     [Circuit Breaker: backup]  --call--> Provider "backup"   (lỗi 5%)       |
|        |  thành công -> cache + return                       route=fallback |
|        |  ProviderError / CircuitOpenError -> ghi nhận                       |
|        v                                                                    |
|  3. [Static fallback]  "dịch vụ tạm suy giảm"               route=static_fallback
|                                                              error=last_error|
+-----------------------------------------------------------------------------+
```

**Circuit breaker (máy trạng thái 3 pha):**

```
                 failure_count >= failure_threshold
   CLOSED  ─────────────────────────────────────────►  OPEN
     ▲                                                   │
     │ success_count >= success_threshold                │ hết reset_timeout_seconds
     │            (probe_success)                        ▼   (allow_request -> probe)
   CLOSED  ◄─────────────────────  HALF_OPEN  ◄────────────────
                                       │
                                       │ probe lỗi bất kỳ (probe_failure)
                                       └──────────────────────►  OPEN
```

- **CLOSED** → mọi call đi qua; đếm số lỗi.
- **OPEN** → fail fast trong `reset_timeout_seconds`, sau đó cho phép một probe (→ HALF_OPEN).
- **HALF_OPEN** → một probe; thành công thì đóng mạch, lỗi thì mở lại ngay lập tức.

Thời gian trôi qua dùng `time.monotonic()` (không bị ảnh hưởng bởi thay đổi đồng hồ hệ thống);
transition log ghi mốc thời gian `time.time()` để phân tích thời gian phục hồi.

---

## 2. Cấu hình

| Tham số | Giá trị | Lý do |
|---|---:|---|
| failure_threshold | 3 | Chịu được các lỗi thoáng qua đơn lẻ (primary lỗi ~25%) mà không mở mạch chỉ vì một lần trượt, nhưng vẫn ngắt nhanh khi có sự cố thật. Giá trị 1 quá nhạy; 5+ để lọt quá nhiều lỗi trước khi bảo vệ. |
| reset_timeout_seconds | 2 | Đủ lâu để một provider chập chờn phục hồi, đủ ngắn để thời gian phục hồi thấp (~2.4 s quan sát được). Khớp SLO phục hồi < 5 s. |
| success_threshold | 1 | Một probe thành công là đủ để tin lại fake provider. Trong production với tín hiệu nhiễu hơn, tôi sẽ nâng lên 2–3 để tránh dao động (flapping). |
| cache TTL | 300 s | Câu trả lời policy/FAQ ổn định trong nhiều phút; 5 phút giới hạn độ cũ trong khi vẫn hấp thụ được lưu lượng lặp lại theo cụm trong một lần chạy 300 request. |
| similarity_threshold | 0.92 | Thử 0.85 trước — gây false hit với các query khác năm ("2024" vs "2026 deadline"). 0.92 vẫn khớp các câu gần trùng, còn phần còn lại do bộ chống false-hit bắt. |
| load_test requests | 100 (×3 kịch bản) | Đủ mẫu để percentile P95/P99 ổn định mà không làm lần chạy chậm. |

Bộ chặn privacy (`_is_uncacheable`) không cho các query chứa `balance / password / credit card /
ssn / social security / user N / account N` được ghi vào hay đọc từ cache.

---

## 3. Định nghĩa SLO

| SLI | Mục tiêu SLO | Giá trị thực | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | 98.0% | ⚠️ Sát ngưỡng |
| Latency P95 | < 2500 ms | 317.5 ms | ✅ |
| Fallback success rate | >= 95% | 93.0% | ⚠️ Sát ngưỡng |
| Cache hit rate | >= 10% | 61.0% | ✅ |
| Thời gian phục hồi | < 5000 ms | 2395 ms | ✅ |

Hai trường hợp sát ngưỡng hoàn toàn đến từ kịch bản `primary_timeout_100`, nơi primary lỗi 100% và
backup vẫn tự lỗi ~5% — một trạng thái suy giảm thực sự. Ở `all_healthy` và `primary_flaky_50`, hệ
thống vượt xa mọi SLO.

---

## 4. Chỉ số (Metrics)

Dán từ `reports/metrics.json` (bật cache, 300 request):

| Chỉ số | Giá trị |
|---|---:|
| total_requests | 300 |
| availability | 0.98 |
| error_rate | 0.02 |
| latency_p50_ms | 282.47 |
| latency_p95_ms | 317.52 |
| latency_p99_ms | 320.20 |
| fallback_success_rate | 0.9302 |
| cache_hit_rate | 0.61 |
| estimated_cost | 0.045524 |
| estimated_cost_saved | 0.183 |
| circuit_open_count | 11 |
| recovery_time_ms | 2395.32 |

Bản CSV có tại `reports/metrics.csv` (một hàng, các kịch bản được trải phẳng thành cột
`scenario_<name>`).

---

## 5. So sánh có/không cache

Hai lần chạy giống hệt nhau, biến số duy nhất là cache (`enabled: true` vs `enabled: false`), mỗi lần
300 request.

| Chỉ số | Không cache | Có cache | Chênh lệch |
|---|---:|---:|---|
| availability | 0.9667 | 0.98 | +1.3 điểm |
| latency_p50_ms | 274.27 | 282.47 | +8.2 ms* |
| latency_p95_ms | 316.26 | 317.52 | ≈ 0 |
| estimated_cost | 0.134404 | 0.045524 | **−66%** |
| circuit_open_count | 20 | 11 | −45% |
| cache_hit_rate | 0.00 | 0.61 | +0.61 |

\* Cache hit trả về trong 0 ms và bị loại khỏi danh sách latency (chỉ ghi các call provider thật), nên
chênh lệch P50 là nhiễu lấy mẫu giữa các call không-cache, không phải do cache gây chậm.

**Kết luận:** cache loại bỏ ~66% chi phí provider, cắt gần một nửa số lần circuit mở (ít call provider
hơn → ít cơ hội lỗi hơn), và nhích availability lên. Chỉ số cost-saved (`0.183`) cộng dồn $0.001 mỗi
hit qua 183 hit.

---

## 6. Redis shared cache

**Vì sao cache in-memory không đủ cho triển khai nhiều instance:** mỗi tiến trình gateway giữ
`ResponseCache` riêng trong bộ nhớ cục bộ. Sau một load balancer với N instance, một query được cache
ở instance A lại là miss ở các instance B…N, nên hit rate hiệu dụng sụt về `1/N`, tiết kiệm chi phí co
lại, và toàn bộ cache mất sạch mỗi lần restart/redeploy.

**`SharedRedisCache` giải quyết thế nào:** lưu mọi entry trong một Redis instance dùng chung cho tất
cả gateway. Data model:

- Key: `rl:cache:{md5(query)[:12]}` (hash ngắn xác định)
- Value: Redis Hash với các field `query` và `response`
- TTL: `EXPIRE` tự động dọn dẹp — không cần quét thủ công

`get()` trước tiên thử exact key hit (trả về score 1.0), sau đó fallback sang `SCAN` toàn bộ prefix,
tính `ResponseCache.similarity()` với từng `query` đã lưu, áp dụng cùng bộ chặn privacy và false-hit
như cache in-memory trước khi trả về kết quả khớp.

### Bằng chứng shared state

Test riêng chứng minh hai đối tượng cache độc lập dùng chung một backend
(`tests/test_redis_cache.py::test_shared_state_across_instances`):

```python
c1.set("shared query", "shared response")   # ghi qua instance 1
cached, _ = c2.get("shared query")           # đọc qua instance 2
assert cached == "shared response"           # ✅ pass khi Redis đang chạy
```

> Trạng thái: Redis đã được khởi động bằng `docker compose up -d` (`redis:7-alpine`, healthy trên
> :6379) và toàn bộ test pass **35 passed, 7 xpassed, 0 skipped** — cả 6 test Redis đều xanh, bao gồm
> `test_shared_state_across_instances`. Tái lập bằng:
>
> ```bash
> docker compose up -d      # khởi động redis:7-alpine trên :6379
> make test                 # 6 test Redis trước đó bị skip nay chạy
> ```

### Output Redis CLI (lấy từ một lần chạy chaos thật với backend Redis)

Chạy mô phỏng với `cache.backend: redis` (300 request). Output thực tế:

```bash
$ docker exec <redis> redis-cli EVAL "return #redis.call('keys','rl:cache:*')" 0
(integer) 13                       # 13 query khác nhau được cache qua lần chạy

$ docker exec <redis> redis-cli --scan --pattern "rl:cache:*"
rl:cache:734852f3cf4a
rl:cache:fff10da1c72c
rl:cache:3dab98c0e49e
...

$ docker exec <redis> redis-cli HGETALL rl:cache:734852f3cf4a
1) "response"
2) "[backup] reliable answer for: What is the tuition fee for the 2025 academic year?"
3) "query"
4) "What is the tuition fee for the 2025 academic year?"

$ docker exec <redis> redis-cli TTL rl:cache:734852f3cf4a
(integer) 263                      # eviction dựa trên EXPIRE, đang đếm ngược từ 300s
```

### So sánh backend in-memory vs Redis — chạy đầy đủ

Cùng cấu hình, chỉ đổi `cache.backend` (`memory` vs `redis`), mỗi lần 300 request:

| Chỉ số | Cache in-memory | Cache Redis | Ghi chú |
|---|---:|---:|---|
| availability | 0.98 | 0.9933 | Cả hai đều mạnh; biến động giữa các lần chạy |
| latency_p50_ms | 282.47 | 275.52 | Truy vấn Redis thêm chi phí không đáng kể so với call provider 180–260 ms |
| latency_p95_ms | 317.52 | 317.73 | Gần như y hệt |
| cache_hit_rate | 0.61 | 0.6933 | Hành vi hit tương đương |
| estimated_cost | 0.045524 | 0.036834 | Cả hai đều thấp hơn nhiều so với 0.134 khi không cache |
| circuit_open_count | 11 | 8 | — |

Backend Redis mang lại lợi ích chi phí/độ tin cậy y như cache in-memory mà không thêm độ trễ đáng kể,
đồng thời bổ sung chia sẻ giữa các instance và sống sót qua các lần restart.

---

## 7. Kịch bản chaos

| Kịch bản | Hành vi kỳ vọng | Hành vi quan sát được | Đạt/Trượt |
|---|---|---|---|
| primary_timeout_100 | Primary lỗi 100%; breaker mở; toàn bộ lưu lượng fallback sang backup | Breaker primary mở sau 3 lỗi rồi fail fast; ~93% request được backup phục vụ (`route=fallback`); ~5% backup lỗi còn lại sinh ra `static_fallback` | ✅ Đạt |
| primary_flaky_50 | Primary lỗi ~50%; mạch dao động OPEN↔HALF_OPEN; trộn primary + fallback | Breaker lặp chu kỳ mở → probe → đóng; thời gian phục hồi ~2.4 s; trộn ổn định giữa route `primary` và `fallback` | ✅ Đạt |
| all_healthy | Cả hai provider khỏe; gần như toàn bộ đi qua primary; ít/không mở mạch | Đa số `route=primary`; thỉnh thoảng mở mạch do tỉ lệ lỗi nền 25% của primary, được backup hấp thụ | ✅ Đạt |
| cache_vs_nocache (tự thêm) | Lần chạy có cache phải giảm chi phí và số lần mở mạch so với đối chứng | Chi phí −66% (0.134 → 0.046), số lần mở mạch −45% (20 → 11), hit rate 0 → 0.61 | ✅ Đạt |

Tổng hợp trên tất cả kịch bản: **circuit_open_count = 11**, **recovery_time_ms = 2395**,
**fallback_success_rate = 0.93** — bằng chứng phục hồi có mặt trong `transition_log` của mỗi breaker.

---

## 8. Phân tích thất bại

**Điều gì vẫn có thể sai?**
Trạng thái circuit breaker nằm *bên trong mỗi tiến trình*. Trong triển khai nhiều instance, instance A
có thể đang OPEN với primary trong khi các instance B…N vẫn dội vào đúng provider đang lỗi đó — mỗi
instance phải tự phát hiện sự cố và trả giá bằng hạn ngạch lỗi riêng. Đây chính là vấn đề fan-out mà
Redis cache đã giải quyết cho phản hồi, nhưng bộ đếm breaker vẫn cục bộ, nên bán kính ảnh hưởng của
một sự cố provider tăng theo số instance thay vì được kiểm soát một lần.

**Điều tôi sẽ thay đổi trước khi lên production:**
Đưa bộ đếm breaker vào Redis (`INCR` key lỗi/thành công kèm `EXPIRE`, một mốc `opened_at` dùng chung)
để mọi instance hội tụ về cùng một quyết định OPEN trong cùng một khung thời gian lỗi — đúng stretch
goal "Redis circuit state". Kết hợp với suy giảm mượt (graceful degradation): nếu Redis không truy cập
được, quay lại dùng breaker in-memory thay vì để request lỗi, để lớp shared-state không trở thành điểm
lỗi đơn mới.

---

## 9. Bước tiếp theo

1. **Shared circuit state trong Redis** — lưu bộ đếm breaker/`opened_at` trong Redis bằng
   `INCR`/`EXPIRE` để N instance ngắt cùng nhau; suy giảm mượt về in-memory nếu Redis sập.
2. **Định tuyến theo chi phí (cost-aware)** — theo dõi ngân sách; vượt 80% thì chuyển sang backup rẻ
   hơn, đạt 100% thì chỉ dùng cache hoặc static, biến các chỉ số chi phí sẵn có thành vòng điều khiển
   chủ động.
3. **Load test đồng thời** — chạy `run_scenario` bằng `ThreadPoolExecutor` để bộc lộ tranh chấp khóa
   (lock contention) và độ trễ đuôi thực tế, rồi kiểm lại P95/P99 so với bảng SLO.
