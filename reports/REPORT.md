# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Phi Nhật (MSV: 2A202602658)  **Commit:** 828c0a3  **Điểm tự động (`make grade`):** 84.0 / 88 (88/88 khi chấm kèm bộ test ẩn)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.217 | 1.044 | 0.005 | 2.129 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.718 | 1.065 | 0.035 | 2.146 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.958 | 1.053 | 0.019 | 2.203 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Dựa trên G6 và G4. User có deadline 30s. Nếu đặt timeout 30s, call treo ở provider hỏng (alpha-eu ở G6) làm cạn sạch deadline và availability tụt xuống 0.9488. Đặt 8.0s (~5-6x median latency của alpha 1.2s) giúp phát hiện treo sớm, kịp failover sang beta. |
| max_attempts_per_provider | 2 | Dựa trên G2 và G5. Giới hạn đúng 1 lần retry trên cùng provider (tối đa 2 attempts). Ở G5 và G2, naive retry bừa bãi làm amplification vọt lên 17.03 và 9.90. Giới hạn 2 giúp nhanh chóng failover sang provider khác nếu provider hiện tại tiếp tục lỗi. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Phân tán tải retry bằng full jitter trong khoảng `[0, min(cap, base * 2^(n-1))]`. Tránh thundering herd; cap 4.0s đảm bảo một lần retry không chờ quá lâu làm cạn quỹ thời gian 30s của người dùng. |
| retry_budget_ratio | 0.1 | Dựa trên G3 (brownout 30% lỗi 500). Giới hạn lượng retry không vượt quá 10% lưu lượng gốc. Ở G3, client đạt retry_ratio = 0.035 (thỏa mãn tiêu chí <= 0.10), trong khi tắt budget làm retry_ratio vọt lên 0.159. |
| breaker.window_size / min_calls | 20 / 10 | Tránh false trip vào thời điểm lưu lượng thấp (chỉ 1-2 request gặp lỗi cá biệt), nhưng đủ nhanh nhạy để ngắt mạch chỉ sau ~2.5 giây khi có sự cố lớn ở tốc độ 4 turns/s (8 calls/s). |
| breaker.failure_rate_threshold | 0.5 | Dựa trên G2 và G6. Khi 50% cuộc gọi trong sliding window thất bại, provider được xác định là không lành mạnh và breaker trip sang OPEN để chuyển hướng tải ngay sang alpha-eu / beta. |
| breaker.slow_call_s | 5.0 | Dựa trên G4. Alpha bị chậm thêm 5s (median từ 1.2s lên 6.2s) nhưng không hề trả mã lỗi HTTP. Đặt ngưỡng 5.0s giúp breaker nhận diện đây là slow call. |
| breaker.slow_rate_threshold | 0.6 | Dựa trên G4. Khi 60% cuộc gọi bị chậm, breaker trip sang OPEN, chuyển lưu lượng sang alpha-eu và kéo p95 latency từ 13.764s (ở naive) xuống chỉ còn 5.684s. |
| breaker.open_duration_s | 10.0 | Cho provider 10 giây nghỉ ngơi để xả hàng đợi nội bộ trước khi sang HALF-OPEN thăm dò. Đủ ngắn để quay lại provider chính ngay sau khi phục hồi (recovery_s = 0.0s ở G2). |
| breaker.half_open_max_probes | 2 | Giới hạn tối đa 2 request đồng thời bay tới provider trong trạng thái HALF-OPEN. Tránh thundering herd dập tắt provider vừa mới hồi phục. |

Sơ đồ quyết định của `on_error`:

```
                           +---------------------------+
                           |     on_error(attempt)     |
                           +---------------------------+
                                         |
                                  Phân loại lỗi
                                         |
         +-------------------------------+-------------------------------+
         |                               |                               |
  [CLIENT_ERROR]              [QUOTA_EXHAUSTED]                   [RATE_LIMITED]
  (400, 401, 422...)          (429 insufficient_quota)            (429 rate_limit_exceeded)
         |                               |                               |
  Release permit                  Release permit                  Release permit
  Không retry, không failover     Cooldown provider (300s)        Cooldown provider (retry-after)
         |                        Fail over sang provider kế      Fail over sang provider kế
    GiveUp("client_error")               |                               |
                                  _route(delay=0)                 _route(delay=0)

                                         |
                  +----------------------+----------------------+
                  | [OVERLOADED / SERVER_ERROR / TIMEOUT]       |
                  | (503, 529, 500, status 0 / client_timeout)   |
                  +---------------------------------------------+
                                         |
                            on_failure(permit) trên Breaker
                                         |
                         Breaker có bị OPEN không?
                               /                   \
                            (Có)                   (Không)
                            /                         \
                Fail over ngay            tries < max_attempts (2)
                _route(delay=0)           AND fits deadline?
                                          AND budget.try_acquire()?
                                                /            \
                                             (Đạt)        (Không đạt)
                                              /                \
                                    Retry cùng provider    Fail over ngay
                                    delay = backoff(...)   _route(delay=0)
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Khi giả lập breaker cấu hình sai (`min_calls: 1000000000`) và cho phép `max_attempts_per_provider: 5`:
   - Trên băng **G2**: Khi giữ retry budget, hệ thống đạt availability = 0.9983, amplification = 1.378, retry_ratio = 0.058 và recovery = 0.0s. Khi tắt budget, amplification vọt lên 1.675, retry_ratio nhảy vọt lên 0.495 (gần 50% cuộc gọi là retry!), làm thời gian phục hồi bị trễ recovery = 25.0s và availability tụt xuống 0.9666.
   - Trên băng **G3** (30% lỗi 500): Có budget giữ retry_ratio ở mức an toàn 0.068 (amplification = 1.136); tắt budget khiến retry_ratio tăng lên 0.159 (vi phạm tiêu chí <= 0.10).
   - *Kết luận:* Ngay cả khi breaker hoạt động đúng, retry budget vẫn là thành phần sống còn vì nó kiểm soát tổng lượng retry toàn cục trong các đợt brownout rải rác mà tỷ lệ lỗi chưa chạm ngưỡng trip của breaker. Hai cơ chế bổ trợ lẫn nhau: breaker giúp fail-fast dứt khoát khi provider sập hoàn toàn, còn budget ngăn chặn bão retry cục bộ làm quá tải mạng lưới.

2. **`attempt_timeout_s: 30`.** Chạy thử nghiệm trên băng G6 (và G4 đối chứng):
   - Trên băng **G6** (alpha sập, alpha-eu treo vô hạn): Khi đặt timeout 30s, availability tụt mạnh từ 1.0000 xuống còn 0.9488 (FAIL tiêu chí >= 0.95), ghi nhận 107 lượt give_up do deadline và 15 lượt no_provider_available. Tuy nhiên, `latency_p95_s` của các lượt thành công lại giảm từ 5.958s xuống còn 5.362s. Đây là biểu hiện rõ rệt của **survivorship bias**: Các lượt gọi bị treo 30s đã cạn sạch deadline của user và bị hủy, không được tính vào p95; p95 chỉ tính trên những lượt may mắn hoàn thành sớm, tạo ảo giác độ trễ "đẹp hơn".
   - Trên băng **G4**: Cả cấu hình 8s và 30s đều đạt availability = 1.0000 và p95 ~ 5.6s (5.684s vs 5.608s). Lý do là ở G4, breaker phát hiện call chậm qua `slow_call_s = 5.0s` và `slow_rate_threshold = 0.6` nên đã tự động ngắt mạch sang alpha-eu sau vài giây, không hề phải đợi chạm ngưỡng timeout của attempt.
   - *Kết luận:* Timeout "đúng" không nên đặt theo user deadline mà phải dựa vào: thời gian cần thiết để failover qua toàn bộ ladder trong deadline (30s chia cho số bậc fallback), kết hợp với p99 lúc bình thường (đặt khoảng 3-5x p99 bình thường, tức 6–8s).

3. **Bỏ lọc tool calling khỏi `ladder()`.** Chạy kịch bản G6 khi bỏ lọc điều kiện `supports_tools`:
   - Availability của hệ thống giảm sút từ 1.0000 xuống 0.8977, ghi nhận **244 lượt gọi không tương thích** (`incompatible_routes = 244`).
   - *Ngôn ngữ Product Manager:* "Do hệ thống định tuyến gửi nhầm 244 yêu cầu cần thao tác nghiệp vụ (như tra cứu đơn hàng và xử lý hoàn tiền) sang model gamma-mini vốn không hỗ trợ gọi công cụ (tool calling), toàn bộ 244 khách hàng này đã nhận về trải nghiệm lỗi hoặc câu trả lời vô nghĩa. Sự cố này trực tiếp làm tỷ lệ giải quyết khiếu nại thành công của ShopAgent giảm hơn 10%, gây gián đoạn quy trình CSKH và làm giảm nghiêm trọng mức độ hài lòng (CSAT) của người dùng."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Trả lời:
- **Vì sao chỉ idempotency key là chưa đủ:** Idempotency key bảo vệ phía external service (cổng thanh toán, email) không bị nhân đôi giao dịch khi retry. Tuy nhiên, nếu worker bị crash và không có checkpoint trong DurableStore, tiến trình worker mới khởi động lại sẽ không biết mình đã làm đến đâu và buộc phải chạy lại từ bước 1 (`classify_request`, `lookup_order`), gây lãng phí lớn tài nguyên và chi phí gọi LLM (`wasted_llm_per_crash` lên tới 1.123 ở naive).
- **Vì sao chỉ checkpoint cũng chưa đủ:** Checkpoint chỉ được ghi sau khi bước chạy xong. Trong trường hợp xảy ra `ServiceTimeout` (yêu cầu trừ tiền đã thành công tại ngân hàng nhưng mạng bị rớt nên client không nhận được phản hồi), client sẽ thử gọi lại. Nếu không có idempotency key ổn định theo (session, step), cổng thanh toán sẽ coi đó là một lệnh mới và trừ tiền/hoàn tiền lần thứ hai, dẫn đến thất thoát tài chính nghiêm trọng (`overpaid_cents = 1,496,700` ở naive).
- **Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment:** Nếu ghi checkpoint trước khi gọi API, lỡ worker bị crash (OOM, restart) ngay sau khi vừa ghi checkpoint nhưng tiền thực tế CHƯA hề được chuyển, thì worker mới khi resume sẽ thấy checkpoint đã tồn tại và bỏ qua bước thanh toán để chuyển sang gửi email. Hậu quả là khách hàng nhận được email báo đã hoàn tiền nhưng tiền không về tài khoản (`lost_refunds > 0`), làm mất giao dịch của khách hàng.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (false) | none | - |
| slow_leak | ticket | none (missed) | ticket | 1337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (false) | none | - |
| small_bump | none | page (false) | none | - |

Error budget 30 ngày: đã tiêu **42.95%** (55,618 / 129,508.8 allowed bad), còn lại **57.05%**. **Không đóng băng release** (`freeze_releases: False`).

Vì sao `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa:
- Trong cửa sổ SLO 30 ngày (43,200 phút), một đợt suy giảm 20 phút với 12% lỗi chỉ tiêu tốn: `20 * (0.12 - 0.005) / (43200 * 0.005) ≈ 1.06%` của toàn bộ error budget cả tháng.
- Lượng hao hụt này hoàn toàn nằm trong ngân sách rủi ro chấp nhận được của dịch vụ. Nếu dùng naive alerter (ngưỡng 5 phút đơn lẻ), hệ thống sẽ page vô tội vạ và gây ra alert fatigue nghiêm trọng cho kỹ sư on-call. Luật đa cửa sổ của Burn-rate Alerter đòi hỏi cả cửa sổ dài (1h với burn rate 14.4x cần tiêu ít nhất 2% budget) lẫn cửa sổ ngắn (5m để kiểm tra sự cố còn tiếp diễn), do đó lọc sạch 100% các blip ngắn mà vẫn bắt chính xác các sự cố đe dọa SLO.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình ảnh phân tích: `reports/gameday/G2_overload_storm.png` và dữ liệu chuỗi thời gian `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào thời điểm t=120s trong kịch bản G2, provider chính `alpha` bị suy giảm đột ngột 75% năng lực xử lý (mất GPU). Do client v1 (`NaiveClient`) sử dụng cơ chế retry 3 tầng chồng chéo (tối đa 36 lần retry/call) với khoảng trễ cố định, không có circuit breaker và không có retry budget, lượng request dồn dập đổ vào `alpha` tăng vọt từ ~7 req/s lên đỉnh 254 req/s tại t=144s, làm sập hoàn toàn hàng đợi của provider. Mặc dù `alpha` đã phục hồi 100% tài nguyên GPU tại t=300s, hệ thống vẫn tiếp tục bị tê liệt thêm 155 giây do bão retry tự duy trì (metastable failure), chỉ thực sự hồi phục tại t=455s và khiến availability của toàn hệ thống rơi xuống mức thảm hại 45.60%.

**Ảnh hưởng:**
- 1,285 lượt yêu cầu của khách hàng bị thất bại (chỉ 1,077 / 2,362 lượt hoàn thành thành công; availability đạt 45.60%).
- Sự cố kéo dài tổng cộng 335 giây (từ t=120s đến t=455s).
- Trong đợt sự cố, hệ thống tiêu tốn 4,983 đơn vị chi phí (cost/turn vọt lên 4.627, gấp hơn 2.1 lần bình thường). Nếu tính trong tháng, chỉ riêng sự cố 5.5 phút này đã tiêu tốn hơn 12.8% error budget của cả tháng.

**Timeline** (giây ảo — trích xuất từ file timeseries CSV):

| t | Sự kiện | Bằng chứng (số liệu CSV) |
|---:|---|---|
| 120 | alpha bị giảm 75% năng lực GPU (bắt đầu sự cố hạ tầng). | attempts_alpha = 5, calls_ok = 7. |
| 136 | Hàng đợi bắt đầu nghẽn, client retry tầng 1 và 2 dồn toa. | attempts_alpha tăng vọt lên 53 req/s, calls_ok giảm về 3 req/s. |
| 140 | Bão retry bùng nổ vượt ngưỡng tiếp nhận của hàng đợi alpha. | attempts_alpha đạt 216 req/s, calls_ok rơi về 0 req/s. |
| 144 | Đỉnh điểm bão retry; hàng đợi alpha tràn ngập, trả về 529 và timeout. | attempts_alpha đạt đỉnh 254 req/s, calls_done = 4, calls_ok = 0. |
| 300 | alpha khôi phục 100% năng lực GPU phần cứng. | attempts_alpha vẫn ở mức 115 req/s, calls_ok vẫn duy trì bằng 0. |
| 305–450 | Metastable failure: Server đầy ắp request zombie bị bỏ rơi, client tiếp tục retry. | attempts_alpha dao động từ 10 đến 36 req/s, calls_ok liên tục = 0. |
| 453 | Hàng đợi alpha bắt đầu xả xong các attempt cũ cạn timeout. | attempts_alpha giảm về 7 req/s, calls_ok bắt đầu phục hồi (4 req/s). |
| 455 | Hệ thống chính thức phục hồi ổn định (recovery_s = 155s tính từ t=300). | calls_ok đạt 8 req/s, calls_done = 8 req/s, attempts = 15 req/s. |

**Trigger vs root cause:**
- **Trigger:** Sự cố hạ tầng tại provider `alpha` làm mất 75% GPU trong vòng 180 giây (t=120s đến t=300s).
- **Root cause:** Lỗi kiến trúc nghiêm trọng ở tầng client v1: tích hợp 3 tầng retry độc lập (app × gateway × SDK = 36 attempts/call) với thời gian trễ cố định, không có randomized jitter, hoàn toàn thiếu circuit breaker để ngắt kết nối và không có retry budget để kiềm chế tải khi hệ thống suy thoái.
- **Cơ chế metastable failure (Vì sao t=300 provider hồi phục mà hệ thống vẫn sập):** Khi provider bị chậm, client timeout sau vài giây nhưng provider vẫn tiếp tục tốn CPU/GPU xử lý các request đó (công sức vô ích). Client v1 không nhận được kết quả liền dội tiếp hàng loạt retry vào hàng đợi. Tải gửi đến vượt xa năng lực tiếp nhận (254 rps so với năng lực 4-5 rps khi suy thoái). Khi provider hồi phục GPU ở t=300s, hàng đợi vẫn chứa đầy hàng trăm request zombie mà client đã từ bỏ từ lâu. Các đợt retry dồn dập tiếp theo liên tục bổ sung tải mới vượt quá tốc độ xử lý của server, tạo thành một vòng lặp tự duy trì (self-sustaining cycle) khiến provider không thể tự ngóc đầu dậy cho đến khi client cạn kiệt toàn bộ lượt retry ở t=455s.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* 3 tầng retry chồng nhau nhân bội số tải lên 36 lần; delay cố định không có jitter khiến hàng trăm request retry đổ bộ vào server cùng một tích tắc; không có circuit breaker khiến 100% cuộc gọi tiếp tục đâm đầu vào alpha mà không failover sang alpha-eu hay beta.
- *Giúp khắc phục (trong ResilientClient):* Circuit breaker ngắt mạch sang OPEN ngay khi tỷ lệ lỗi vượt quá 50%, chuyển hướng traffic sang alpha-eu và beta; Retry budget giới hạn retry <= 10%; Full jitter giãn đều các đợt thử lại; và chỉ retry đúng 1 lần trên cùng provider. Kết quả: availability ở G2 đạt 100.0%, amplification giảm từ 9.904 xuống 1.044, và recovery_s giảm từ 155s về 0.0s!

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai ResilientClient tích hợp Circuit Breaker sliding window và Retry Budget (ratio 0.1) trên toàn bộ agent production. | Prevent | Reliability Team | Chạy test mô phỏng kịch bản G2 trong CI: amplification <= 1.2 và recovery_s <= 5s. |
| 2 | Cấu hình Fallback Ladder đa vùng (alpha -> alpha-eu -> beta) và giới hạn attempt timeout ở mức 8s. | Mitigate | Platform Team | Mô phỏng outage của provider chính: availability toàn hệ thống vẫn đạt >= 98%. |
| 3 | Xây dựng bộ cảnh báo đa cửa sổ Burn-rate Alerting (page ở 14.4x trong 1h/5m và ticket ở 1x trong 3d/6h) theo Google SRE Workbook. | Detect | SRE On-call | Bắt đúng 100% sự cố lớn trong vòng 5 phút, 0 false alarm trên blip ngắn trong 30 ngày. |

## 7. Một điểm yếu còn lại

Một điểm yếu còn lại của client là **sự gia tăng chi phí khi fallback sang các model đắt tiền hơn**:
- Khi provider chính `alpha` (cost = 1.0) gặp sự cố kéo dài và mạch chuyển hướng tải sang `beta` (cost = 1.4), chi phí mỗi lượt của hệ thống tăng vọt (như trong G5 cost/good turn tăng lên 2.504 so với 2.101 ở G1; ở G6 có tới 2,139 cuộc gọi dồn sang `beta`). Nếu sự cố kéo dài nhiều giờ hoặc nhiều ngày, ngân sách API của doanh nghiệp sẽ bị thâm hụt nghiêm trọng.
- **Cách khắc phục:** Triển khai cơ chế **Cost-aware Degraded Mode**: Khi circuit breaker của `alpha` mở và buộc phải chuyển sang `beta`, client có thể chủ động rút ngắn độ dài prompt, tắt các tính năng phân tích nâng cao không bắt buộc, hoặc áp dụng aggressive semantic caching hơn để giảm số lượng token và số lượt gọi đến model đắt tiền.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Starter đặt 0.80 dẫn đến 157 câu trả lời sai (`wrong_answers = 157` do nhầm sang câu hỏi khác `other_question`). Nếu đặt 0.97, ngưỡng quá chặt làm giảm hit ratio xuống 0.581 và tăng cost lên 0.964. Giá trị 0.92 loại bỏ 100% câu trả lời sai (`wrong_answers = 0`), giữ hit ratio cao ở mức 65.2% và chi phí rẻ nhất (0.915). |
| ttl_s | 60 | Starter đặt 600s dẫn đến 80 câu trả lời lỗi thời khi đối tác âm thầm đổi bảng phí ship ở t=480 (`outdated_answers = 80`). Đặt 30s làm hit ratio giảm xuống 0.592. Giá trị 60s giới hạn số câu trả lời cũ còn 3 (`outdated_answers = 3 <= 15`) mà vẫn duy trì hit ratio 65.2%. |
| max_stale_s | 300 | Sự cố provider mất kết nối hoàn toàn kéo dài 120s (t=240 đến t=360). Nếu để `max_stale_s: 0`, faq_availability rơi xuống 0.8733 (FAIL tiêu chí >= 0.93). Đặt 300s cho phép phục vụ 157 lượt trả lời stale an toàn, nâng faq_availability lên 0.9645 (vượt ngưỡng 0.93). |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng similarity_threshold:**
   - Tại `similarity_threshold = 0.80`: hit ratio đạt 0.666, nhưng phát sinh tới 157 câu trả lời sai (`wrong_answers = 157`, lý do 100% là `other_question`), chi phí là 0.965. Các câu hỏi bị trả lời nhầm là những câu có embedding tương đối gần nhưng ngữ nghĩa và mục đích khác hẳn nhau (ví dụ: hỏi về "thời gian hoàn tiền" bị trả nhầm sang "chính sách hoàn tiền").
   - Tại `similarity_threshold = 0.92`: hit ratio đạt 0.652, `wrong_answers = 0`, cost/turn tối ưu nhất ở mức 0.915.
   - Tại `similarity_threshold = 0.97`: ngưỡng quá khắt khe khiến nhiều câu hỏi diễn đạt tương đương (paraphrase) bị trượt cache, đẩy hit ratio tụt xuống 0.581, `wrong_answers = 0`, nhưng chi phí tăng lên 0.964 do phải gọi provider nhiều hơn.

2. **TTL (ttl_s):**
   - Chạy với `ttl_s = 30s`: `outdated_answers = 2`, hit ratio = 0.592, faq_availability = 0.9645.
   - Chạy với `ttl_s = 60s`: `outdated_answers = 3`, hit ratio = 0.652, faq_availability = 0.9645.
   - Chạy với `ttl_s = 300s`: `outdated_answers = 95` (vượt xa trần cho phép 15), hit ratio = 0.731, faq_availability = 0.9640.
   - *Vì sao webhook không giúp được gì cho `outdated_answers`:* Sự cố thay đổi bảng phí ship ở t=480 là `silent_updates` (đối tác vận chuyển thay đổi giá mà không gửi thông báo hay webhook cho hệ thống). Do không có webhook nên cache không thể kích hoạt invalidation chủ động; cơ chế TTL hết hạn thụ động là phòng tuyến duy nhất để đào thải dữ liệu cũ này.

3. **Stale-if-error:**
   - Khi chạy với `max_stale_s: 0`: `faq_availability` sụt giảm mạnh từ 0.9645 xuống chỉ còn **0.8733** (thấp hơn nhiều so với tiêu chuẩn 0.93), và số lượt `stale_served` giảm từ 157 về 0.
   - Trong 120 giây toàn bộ provider sập (t=240 đến t=360), mọi câu hỏi FAQ bị miss hoặc đã hết TTL đều không thể gọi provider và buộc phải GiveUp, chứng minh vai trò then chốt của stale-if-error trong việc cứu vãn tính sẵn sàng của dịch vụ.

Trả lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user:**
  Dữ liệu đơn hàng, số dư và trạng thái giao hàng là dữ liệu biến động tức thời (stateful) và chứa thông tin định danh cá nhân (PII). Nếu cache câu hỏi cá nhân, khách hàng có nguy cơ nhận thông tin đơn hàng đã cũ (ví dụ đơn đã giao nhưng cache vẫn báo đang xử lý). Nguy hiểm hơn, nếu logic phân tách user bị sai sót (collision trong key), câu trả lời có thể làm lộ thông tin nhạy cảm của người này cho người khác xem (vi phạm nghiêm trọng quyền riêng tư và quy định bảo vệ dữ liệu).

- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố:**
  Không được chỉ dựa vào metric `availability` ở tầng client/user (bị che mờ bởi cache hit và stale serving). Đội ngũ on-call phải thiết lập alert dựa trên **các metric backend provider**:
  1. Tỷ lệ lỗi và tỷ lệ timeout của các lời gọi provider phía sau cache (`provider_error_rate >= 10%`).
  2. Trạng thái của Circuit Breakers (`breaker_state == OPEN` trên các provider).
  3. Tỷ lệ phục vụ dữ liệu cũ trong fallback (`stale_served_ratio > 0` hoặc số lượng `stale_served` tăng đột biến).
