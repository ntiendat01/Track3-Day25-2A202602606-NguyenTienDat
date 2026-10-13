# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Tiến Đạt - 2A202602606  **Commit:** 43a3b2d  **Điểm tự động (make grade):** 84 / 88

> Bốn điểm E còn lại chạy trên dữ liệu SLO ẩn của giảng viên; toàn bộ 82/82 unit test, dữ liệu SLO public, 6/6 Game Day, workflow và cache tape đều PASS.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry\_ratio | cost/turn | recovery (s) | PASS? |
| ---- | ------ | ------------ | ------- | ------------- | ------------ | --------- | ------------ | ----- |
| G1   | naive  | 0.7194       | 7.157   | 7.738         | 6.738        | 2.163     | 4            | Không |
| G1   | yours  | 1.0000       | 4.776   | 1.062         | 0.002        | 2.101     | 0            | Có    |
| G2   | naive  | 0.4560       | 15.034  | 9.904         | 8.904        | 4.627     | 155          | Không |
| G2   | yours  | 1.0000       | 11.911  | 1.059         | 0.008        | 2.161     | 0            | Có    |
| G3   | naive  | 1.0000       | 4.768   | 1.164         | 0.164        | 2.328     | 0            | Không |
| G3   | yours  | 1.0000       | 4.738   | 1.058         | 0.034        | 2.141     | 0            | Có    |
| G4   | naive  | 1.0000       | 13.764  | 1.004         | 0.004        | 2.008     | 0            | Không |
| G4   | yours  | 1.0000       | 5.562   | 1.015         | 0.002        | 2.072     | 0            | Có    |
| G5   | naive  | 0.3699       | 5.359   | 17.030        | 16.030       | 2.124     | —            | Không |
| G5   | yours  | 0.8179       | 5.477   | 1.003         | 0.002        | 2.504     | —            | Có    |
| G6   | naive  | 0.5170       | 4.836   | 12.344        | 11.344       | 2.009     | 0            | Không |
| G6   | yours  | 1.0000       | 5.961   | 1.055         | 0.018        | 2.196     | 0            | Có    |

Client mới giảm amplification của G2 từ 9.904 xuống 1.059 và recovery từ 155 s xuống 0 s. G5 là trường hợp không thể cứu toàn bộ request malformed, nhưng client không lặp lỗi 400: `client_error_retries = 0`, nên vẫn đạt availability 81.79% (ngưỡng 78%).

## 2. Cấu hình client và lý do

| Tham số                           | Giá trị      | Vì sao (dựa trên băng nào, số liệu nào)                                                                                                                                  |
| --------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `attempt_timeout_s`               | 8 s          | Giới hạn thời gian chờ một deployment để còn thời gian failover trước deadline 30 s. G6 đạt 100% availability; thử 30 s chỉ còn 94.88% vì request treo làm cạn deadline. |
| `max_attempts_per_provider`       | 2            | Một retry cục bộ đủ che lỗi ngắn, sau đó chuyển tải. G3 còn retry ratio 0.034, dưới ngưỡng 0.10; amplification chỉ 1.058.                                                |
| `backoff_base_s / cap_s`          | 0.25 s / 4 s | Full jitter phân tán retry và không tạo retry storm đồng pha. G2 có retry ratio 0.008 và không có recovery debt.                                                         |
| `retry_budget_ratio`              | 0.10         | Giới hạn retry khi breaker có cấu hình sai: G2 tăng từ amplification 1.378 (bật) lên 1.675 (tắt budget).                                                                 |
| `breaker.window_size / min_calls` | 20 / 10      | Cần đủ 10 kết quả trước khi kết luận, nhưng chỉ nhớ 20 kết quả gần nhất để phản ứng trong một incident ngắn.                                                             |
| `breaker.failure_rate_threshold`  | 0.50         | Phân biệt lỗi nền với lỗi provider kéo dài; G2/G6 đều fail over thành công mà amplification chỉ 1.059/1.055.                                                             |
| `breaker.slow_call_s`             | 5 s          | G4 không có lỗi nhưng p95 giảm từ 13.764 s xuống 5.562 s nhờ coi success chậm là tín hiệu xấu.                                                                           |
| `breaker.open_duration_s`         | 10 s         | Cho provider/queue thời gian hạ tải trước probe; sau restore G2 quay lại alpha 100% và recovery là 0 s.                                                                  |
| `breaker.half_open_max_probes`    | 2            | Chỉ cho hai probe đồng thời, tránh thundering herd khi G2/G4 vừa phục hồi.                                                                                               |

Sơ đồ quyết định của `on_error`:

| Loại lỗi                                | Breaker                         | Retry cùng provider                                    | Failover                   | Cooldown          |
| --------------------------------------- | ------------------------------- | ------------------------------------------------------ | -------------------------- | ----------------- |
| `CLIENT_ERROR` (400/401/…)              | release, không ghi lỗi sức khoẻ | Không                                                  | Không; `GiveUp`            | Không             |
| `QUOTA_EXHAUSTED`                       | release                         | Không                                                  | Có, ngay                   | 300 s             |
| `RATE_LIMITED`                          | release                         | Không                                                  | Có, ngay                   | đến `retry_after` |
| `OVERLOADED`, `SERVER_ERROR`, `TIMEOUT` | `on_failure`                    | Tối đa một lần, full jitter và phải có budget/deadline | Có nếu retry không phù hợp | Không             |

Ladder luôn lọc capability: call `tool` chỉ đi `alpha`, `alpha-eu`, `beta`; `gamma-mini` chỉ được dùng cho chat.

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Tôi cố ý làm breaker không thể mở (`min_calls = 1_000_000_000`) và tăng `max_attempts_per_provider` lên 5. Trên G2, giữ budget cho availability 99.83%, amplification 1.378, retry ratio 0.058 và recovery 0 s; tắt budget cho 96.66%, 1.675, 0.495 và 25 s. Trên G3, budget giữ amplification/retry ratio ở 1.136/0.068, còn không budget là 1.162/0.159. Khi breaker bình thường, G2 chỉ còn 1.059/0.008 nên budget ít được dùng; vẫn cần cả hai vì breaker phát hiện theo cửa sổ sau khi một số call đã thất bại, còn budget là chốt chặn ngay cả khi breaker bị tuning sai hoặc có lỗi triển khai.
2. **attempt\_timeout\_s: 30.** G4 gần như không đổi (p95 5.608 s, availability 100%) so với cấu hình 8 s (5.562 s, 100%), vì slow-call breaker 5 s đã chuyển traffic trước khi cần timeout. Trên G6, timeout 30 s hạ availability từ 100% xuống 94.88%; p95 success-only nhìn có vẻ 5.410 s nhưng gây hiểu nhầm vì các turn hết deadline không đi vào p95. Timeout đúng phải chừa thời gian cho một lần failover có ý nghĩa trước user deadline và phối hợp với slow-call breaker, không phải bằng deadline toàn lượt.
3. **Bỏ lọc tool calling khỏi ladder().** Trên G6, client thử `gamma-mini` cho tool call 256 lần (`incompatible_routes = 256`), availability giảm từ 100% xuống 89.27%. Theo ngôn ngữ product: 256 khách nhận một luồng xử lý mà model không thể thực thi tool cần thiết; đó là lỗi chức năng, không phải một fallback thành công hay một tiết kiệm chi phí.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid        | wasted LLM / crash |
| -------- | ---------- | ------- | ----------------- | ---------------- | ------------ | --------------- | ------------------ |
| naive    | 1.000      | 81      | 128               | 81               | 0            | 1,574,200 cents | 1.123              |
| yours    | 1.000      | 66      | 0                 | 0                | 0            | 0 cents         | 0.303              |

Idempotency key một mình không đủ: payment/email có thể được dedupe, nhưng worker restart vẫn gọi lại LLM, lookup và các bước đã hoàn tất. Checkpoint một mình cũng không đủ: worker có thể crash sau khi payment đã thực thi nhưng trước khi checkpoint được ghi; restart không thấy checkpoint và sẽ gửi một payment mới. Key ổn định theo `(session_id, step)` chặn effect trùng trong khoảng trống đó; checkpoint **sau** successful step tránh tính lại phần đã biết hoàn tất. Nếu checkpoint được ghi trước payment, crash sau checkpoint sẽ làm worker bỏ qua payment và tạo lost refund.

## 5. Alerting & error budget

| Sự cố             | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (burn-rate) |
| ----------------- | -------- | ------------- | ----------------- | ---------------------------- |
| checkout\_bug     | page     | page          | page              | 27 phút                      |
| provider\_outage  | page     | page          | page              | 4 phút                       |
| deploy\_blip      | none     | false page    | không page        | —                            |
| slow\_leak        | ticket   | bỏ sót        | ticket            | 1,337 phút                   |
| degraded\_evening | page     | page          | page              | 291 phút                     |
| night\_spike      | none     | false page    | không page        | —                            |
| small\_bump       | none     | false page    | không page        | —                            |

Burn-rate tạo 3 page và 1 ticket, với 0 false page; naive tạo 7 page, 3 false page và bỏ sót slow leak. Trong 30 ngày: 25,901,755 request, budget cho phép 129,508.8 bad request, thực tế 55,618 bad. Đã tiêu **42.95%**, còn **57.05%**, nên chưa đóng băng release (`freeze_releases = false`). `small_bump` (20 phút, 12% lỗi) không nên đánh thức on-call vì cả cửa sổ dài và ngắn của rule burn-rate không đồng thời chứng minh rằng nó đang đốt error budget với tốc độ đáng kể; threshold 5 phút của naive lại page sai.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 attempts and rolling success ratio](gameday/G2_overload_storm.png)

**Tóm tắt (3 câu):** Từ t=120, alpha mất 75% năng lực GPU, trong khi `NaiveClient` vẫn chạy ba tầng retry, tổng cộng tối đa 36 attempt/call, với delay cố định. Trong G2, hệ thống tạo 34,475 attempt cho 3,481 logical call (amplification 9.904), và chỉ hoàn thành 1,077/2,362 turn (availability 45.60%). Provider khôi phục ở t=300 nhưng success ratio không đạt 95% liên tục trong 30 s cho đến t=455, nên recovery debt kéo dài thêm 155 s.

**Ảnh hưởng:** 1,285 user turn thất bại; p95 của turn thành công là 15.034 s và chi phí/turn tốt là 4.627, cao hơn 2.14× so với resilient (2.161). Với giả định traffic nền 4 turn/s suốt 30 ngày, error budget 99.5% là `0.005 × 4 × 2,592,000 = 51,840` turn lỗi; riêng 1,285 lỗi này tiêu khoảng **2.48%** budget tháng. Khoảng t=150 đến sau t=420, success ratio 10 s là 0%; cửa sổ phục hồi 30 s đầu tiên đủ 95% bắt đầu tại t=455.

**Timeline** (giây ảo — lấy từ CSV, attempts/s là tổng 10 giây gần nhất):

| t   | Sự kiện                           | Bằng chứng (số)                                                                                |
| --- | --------------------------------- | ---------------------------------------------------------------------------------------------- |
| 110 | Trước sự cố                       | 67 attempt/10 s; 73/73 completed call thành công (100%).                                       |
| 120 | Alpha mất 75% GPU                 | 91 attempt/10 s; vẫn 90/90 success do request đang xử lý trước đó.                             |
| 150 | Retry storm hình thành            | 2,089 attempt/10 s nhưng 0/50 completed call thành công.                                       |
| 180 | Queue/timeout tự khuếch đại       | 1,148 attempt/10 s; 0/32 success.                                                              |
| 240 | Sự cố vẫn bão hoà                 | 1,382 attempt/10 s; 0/56 success.                                                              |
| 300 | Alpha hồi phục hoàn toàn          | 1,429 attempt/10 s; vẫn 0/43 success: queue còn đầy request cũ.                                |
| 420 | Chưa tự hồi phục                  | 688 attempt/10 s; vẫn 0/36 success.                                                            |
| 455 | Bắt đầu cửa sổ recovery đạt chuẩn | tại giây này 12/40 success; các giây sau đủ để cửa sổ 30 s đạt ≥95%, tương ứng recovery 155 s. |
| 480 | Phục hồi rõ ràng                  | 88 attempt/10 s; 137/137 success (100%).                                                       |

**Trigger vs root cause:** Trigger là mất 75% GPU tại alpha. Root cause là chính sách retry ba tầng, delay cố định, không retry budget và không circuit breaker; retry đã chuyển một giảm năng lực thành lượng request lớn hơn hẳn năng lực còn lại. Sau t=300, alpha đã có lại GPU nhưng hàng đợi còn chứa request mà client đã timeout hoặc bỏ cuộc; retry mới tiếp tục đi vào hàng đợi và giữ tải cao hơn throughput. Vòng lặp “queue dài → timeout/529 → retry → queue dài hơn” là metastable failure, nên không tự dừng ngay khi hardware hồi phục.

**Điều gì giúp / điều gì làm tệ hơn:** Simulator có deadline 30 s và queue shedding 529 nên sự cố hữu hạn, nhưng naive retry 36 lần/call và vi phạm `retry-after` 28,381 lần đã làm tình hình tệ hơn. Resilient client giảm retry cùng provider xuống 0.008/call, tôn trọng mọi `retry-after`, mở breaker, spill over và đạt availability 100%.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động                                                                                                        | Loại (prevent / detect / mitigate) | Owner                    | Cách kiểm chứng                                                                    |
| - | ---------------------------------------------------------------------------------------------------------------- | ---------------------------------- | ------------------------ | ---------------------------------------------------------------------------------- |
| 1 | Chuẩn hoá một tầng retry: full jitter, tôn trọng `retry-after`, token budget ratio 0.10.                         | prevent                            | Agent Platform           | CI chạy G2: amplification ≤1.2 và retry-after violations = 0.                      |
| 2 | Đưa sliding-window breaker có slow-call, HALF-OPEN tối đa 2 probe vào provider adapter.                          | mitigate                           | LLM Reliability          | G2 recovery ≤30 s; G4 p95 ≤10 s trên ba seed.                                      |
| 3 | Page theo 14.4×/6× burn-rate và dashboard thêm amplification, queue age, fallback share.                         | detect                             | SRE on-call              | Replay các incident SLO: 0 false page và provider\_outage được page trong ≤5 phút. |
| 4 | Giới hạn admission/concurrency và huỷ công việc server khi client deadline hết nếu provider hỗ trợ cancellation. | mitigate                           | Inference Infrastructure | Load test giảm queue age sau restore; G2 giữ success ratio 30 s ≥95% trong ≤30 s.  |

## 7. Một điểm yếu còn lại

G2 vẫn có p95 11.911 s dù đã PASS, và `beta` chịu tải fallback với chi phí/turn tốt 2.161 thay vì G1 là 2.101. Nếu cả bốn provider đều slow nhưng chưa trả lỗi, per-provider breaker có thể lần lượt mở trong khi user deadline bị tiêu dần. Bước tiếp theo là thêm admission control theo concurrency/queue age, deadline-aware routing (không thử provider có latency dự báo vượt thời gian còn lại), và một global failure-domain breaker để hạ tải hoặc trả degraded response sớm thay vì đợi từng provider timeout.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client                  | availability | faq\_availability | cache\_hit\_ratio | stale\_served | wrong\_answers | outdated\_answers | cost/turn | PASS? |
| ----------------------- | ------------ | ----------------- | ----------------- | ------------- | -------------- | ----------------- | --------- | ----- |
| resilient (không cache) | 0.8373       | 0.8424            | 0.000             | 0             | 0              | 0                 | 1.386     | Không |
| cached (của bạn)        | 0.9113       | 0.9634            | 0.634             | 172           | 0              | 6                 | 0.928     | Có    |

| Tham số `cache:`       | Giá trị | Vì sao (số liệu nào)                                                                                                                                               |
| ---------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `similarity_threshold` | 0.92    | 0.80 tạo 138 wrong answer (`other_question`); 0.92 có 0 wrong answer, hit ratio 0.634 và cost 0.928.                                                               |
| `ttl_s`                | 45 s    | Giới hạn nội dung silent-update: cấu hình hiện tại có 6 outdated answer (<15), trong khi 300 s có 95. Giá trị 45 s giữ hit 0.634 và để dư địa an toàn cho tape ẩn. |
| `max_stale_s`          | 180 s   | Bao phủ outage t=240–360: phục vụ 172 stale answer, đưa FAQ availability lên 0.9634; đặt 0 chỉ còn 0.8634.                                                         |

Thí nghiệm:

1. **Ngưỡng.** Với threshold 0.80, hit ratio là 0.646 nhưng có 138 wrong answer, toàn bộ là `other_question`; cost/turn cũng không tốt hơn (0.972). Với 0.92, hit ratio 0.634, 0 wrong answer và cost 0.928. Tăng quá chặt lên 0.97 vẫn an toàn nhưng hit ratio giảm 0.552 và cost tăng 0.984. Vì vậy 0.92 là điểm cân bằng giữa semantic reuse và chính xác.
2. **TTL.** Với TTL 30/60/300 s, hit ratio lần lượt 0.591/0.652/0.731; outdated answer là 2/3/95; cost là 0.956/0.916/0.863. TTL 300 rẻ hơn nhưng vi phạm ngưỡng outdated ≤15; 45 s chọn độ tươi mới bảo thủ hơn, có 6 outdated answer. Webhook chỉ invalidate tài liệu đổi **có thông báo** (refund policy tại t=150); nó không biết bảng phí ship đổi âm thầm tại t=480, nên chỉ TTL/revalidation có thể giới hạn `outdated_answers`.
3. **Stale-if-error.** Với `max_stale_s = 0`, FAQ availability là 0.8634, không có stale answer và cost/turn 0.995. Với 180 s, FAQ availability tăng 10 điểm phần trăm lên 0.9634 nhờ 172 stale answer, trong khi wrong answer vẫn bằng 0 vì scope prompt và invalidation vẫn được kiểm tra trước khi serve.

Trả lời:

- Câu hỏi cá nhân không được cache, kể cả cache theo user, vì câu trả lời chứa trạng thái order/refund/points biến động nhanh và có rủi ro sai scope, sai authorization hoặc lộ dữ liệu khi key/scope bị lỗi. Cache này chỉ allowlist FAQ không cá nhân, scope thêm `prompt_version`, và không lưu request personal.
- Khi provider outage, dashboard availability tổng có thể xanh nhờ stale cache. On-call vẫn cần alert từ provider-attempt success/error, `GiveUp` cho non-FAQ/personal, breaker-open duration, fallback share và tỷ lệ stale served. Tăng đột biến stale serve (172 ở C1) cùng 503/timeout là tín hiệu provider outage, không phải bằng chứng provider khoẻ.
