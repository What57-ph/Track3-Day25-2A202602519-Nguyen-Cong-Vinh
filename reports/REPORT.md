# Lab 25A — Report

**Họ tên / nhóm:** Nguyễn Công Vinh **Commit:** `fa340f4bee7f44857c7a66f1fc5087bac4539815` **Điểm tự động (`make grade`):** 88 / 88

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
| ---- | ------ | -----------: | ------: | ------------: | ----------: | --------: | -----------: | :---: |
| G1   | naive  |       0.7194 |   7.157 |         7.738 |       6.738 |     2.163 |            4 | FAIL  |
| G1   | yours  |       1.0000 |   4.780 |         1.063 |       0.001 |     2.104 |            0 | PASS  |
| G2   | naive  |       0.4560 |  15.034 |         9.904 |       8.904 |     4.627 |          155 | FAIL  |
| G2   | yours  |       1.0000 |   9.922 |         1.035 |       0.004 |     2.106 |            0 | PASS  |
| G3   | naive  |       1.0000 |   4.768 |         1.164 |       0.164 |     2.328 |            0 | FAIL  |
| G3   | yours  |       1.0000 |   4.642 |         1.076 |       0.049 |     2.167 |            0 | PASS  |
| G4   | naive  |       1.0000 |  13.764 |         1.004 |       0.004 |     2.008 |            0 | FAIL  |
| G4   | yours  |       1.0000 |   5.626 |         1.014 |       0.002 |     2.072 |            0 | PASS  |
| G5   | naive  |       0.3699 |   5.359 |        17.030 |      16.030 |     2.124 |            — | FAIL  |
| G5   | yours  |       0.8179 |   5.477 |         1.003 |       0.002 |     2.504 |            — | PASS  |
| G6   | naive  |       0.5170 |   4.836 |        12.344 |      11.344 |     2.009 |            0 | FAIL  |
| G6   | yours  |       1.0000 |   5.949 |         1.052 |       0.017 |     2.210 |            0 | PASS  |

Resilient đạt 6/6 băng. G5 có availability 81.79%, cao hơn ngưỡng 78%; 413 lỗi 400 được kết thúc mà không retry hoặc failover.

## 2. Cấu hình client và lý do

| Tham số                         |    Giá trị | Vì sao (dựa trên băng nào, số liệu nào)                                                                                                                                   |
| ------------------------------- | ---------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| attempt_timeout_s               |        8 s | G4 p95 5.626 s, G6 p95 5.949 s và availability đều 100%. Đối chứng timeout 30 s làm G6 availability còn 94.88%, với 108 give-up `deadline` và 14 `no_provider_available`. |
| max_attempts_per_provider       |          2 | Cho phép một retry cùng provider. G3 đạt availability 100%, retry ratio 0.049, amplification 1.076.                                                                       |
| backoff_base_s / cap_s          | 0.25 / 4 s | Full jitter giãn retry và giới hạn thời gian chờ; G1 có 0 vi phạm `retry-after`, G2 amplification 1.035.                                                                  |
| retry_budget_ratio              |       0.10 | Khi breaker không hoạt động, budget làm G2 amplification từ 1.614 xuống 1.382 và G3 retry ratio từ 0.160 xuống 0.067.                                                     |
| breaker.window_size / min_calls |    20 / 10 | Xét 20 kết quả gần nhất, đợi đủ 10 mẫu trước khi mở để tránh phản ứng với ít lỗi ban đầu. G4 p95 còn 5.626 s.                                                             |
| breaker.failure_rate_threshold  |       0.50 | Mở breaker khi ít nhất một nửa cửa sổ là lỗi; hạn chế khuếch đại retry trong G2/G3.                                                                                       |
| breaker.slow_call_s             |        5 s | Thành công nhưng mất từ 5 s trở lên vẫn được xem là dấu hiệu provider suy giảm; G4 thêm 5 s vào alpha nhưng resilient p95 là 5.626 s.                                     |
| breaker.open_duration_s         |       10 s | Cho provider thời gian hồi phục trước probe; primary share sau phục hồi là 1.0 trong G2 và G4.                                                                            |
| breaker.half_open_max_probes    |          2 | Giới hạn hai probe đồng thời; cần hai probe thành công mới đóng breaker.                                                                                                  |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

| Loại lỗi                  | Ghi breaker?          | Retry cùng provider?                            | Failover / cooldown                                          |
| ------------------------- | --------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| 400 / lỗi client          | Không, release permit | Không                                           | Give up; request sai ở mọi provider                          |
| 429 `insufficient_quota`  | Không, release permit | Không                                           | Cooldown provider theo `quota_cooldown_s`, rồi failover      |
| 429 rate limit            | Không, release permit | Không                                           | Tôn trọng `retry_after`, spillover sang provider khác        |
| 503 / 529 / 5xx / timeout | Có                    | Có nếu còn lượt, budget cho phép và đủ deadline | Nếu không retry được thì failover; breaker OPEN sẽ bị bỏ qua |
| Thành công                | Ghi latency           | —                                               | HALF-OPEN đóng sau đủ probe thành công                       |

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Giả lập breaker cấu hình sai (`breaker.min_calls: 1000000000`) và đặt
   `max_attempts_per_provider: 5`. Chạy G2, G3 hai lần: (a) giữ retry budget; (b) tắt budget
   (`retry_budget_ratio: 100`, `retry_budget_max_tokens: 1.0e9`, `retry_budget_min_per_s: 1.0e6`).
   So sánh amplification, retry_ratio, recovery. Khi breaker _hoạt động đúng_ thì budget còn quan trọng không — vì sao
   vẫn nên có cả hai?

   G2 khi giữ budget: availability 99.96%, amplification 1.382, retry ratio 0.058, recovery 0 s; khi tắt budget: 96.57%, 1.614, 0.404 và 30 s. G3 giữ budget có amplification 1.135, retry ratio 0.067; tắt budget tăng lên 1.162 và 0.160, trong khi availability vẫn 100%. Breaker giới hạn lỗi ở provider, còn budget chặn retry tăng vọt khi breaker cấu hình sai, chưa đủ mẫu hoặc bị bỏ qua; vì vậy nên dùng cả hai.

2. **`attempt_timeout_s: 30`.** Chạy G4, G6. Chuyện gì xảy ra với p95 và availability? Timeout "đúng" nên dựa vào đâu?

   G4 availability vẫn 100%, p95 đổi nhẹ từ 5.626 lên 5.683 s nhờ slow-call breaker. G6 availability giảm từ 100% xuống 94.88%; có 108 give-up do deadline và 14 `no_provider_available`, dù p95 của các lượt thành công là 5.296 s. Cần đánh giá availability và nguyên nhân give-up trước p95; timeout nên dựa trên deadline người dùng, latency/queue của provider và thời gian cần để failover.

3. **Bỏ lọc tool calling khỏi `ladder()`.** Chạy G6. Giải thích con số `incompatible_routes` bằng ngôn ngữ của
   một product manager.

   Bỏ lọc làm G6 có 238 `incompatible_routes` và availability chỉ 90.02%. Đây là 238 lượt gửi tác vụ cần tool sang `gamma-mini`, model không hỗ trợ tool calling, nên trả lời của nó không thể hoàn tất thao tác người dùng. Từ góc nhìn sản phẩm, chọn model rẻ nhưng không đủ khả năng tạo ra yêu cầu thất bại thay vì tiết kiệm chi phí.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds |   overpaid | wasted LLM / crash |
| -------- | ---------: | ------: | ----------------: | ---------------: | -----------: | ---------: | -----------------: |
| naive    |       100% |      81 |               128 |               81 |            0 | 1,574,200¢ |              1.123 |
| yours    |       100% |      66 |                 0 |                0 |            0 |         0¢ |              0.303 |

Trả lời: vì sao chỉ idempotency key là **chưa đủ**, và chỉ checkpoint cũng **chưa đủ**? Điều gì xảy ra nếu ghi
checkpoint _trước_ khi gọi payment?

Idempotency key ngăn payment/email thực thi trùng, nhưng không lưu kết quả các bước đã chạy nên worker khởi động lại vẫn tốn LLM/lookup. Checkpoint không thể tự biết payment đã commit hay chưa nếu worker crash trước khi ghi kết quả; retry khi đó có thể hoàn tiền hai lần. Vì vậy phải gọi side effect trước, rồi checkpoint kết quả; nếu response bị mất thì retry với cùng key. Ghi checkpoint trước payment có thể khiến worker restart bỏ qua payment chưa từng xảy ra, làm mất khoản hoàn tiền.

## 5. Alerting & error budget

| Sự cố            | Mong đợi | Naive alerter | Burn-rate alerter  | Độ trễ phát hiện |
| ---------------- | -------- | ------------- | ------------------ | ---------------: |
| checkout_bug     | page     | Page          | Page               |          27 phút |
| provider_outage  | page     | Page          | Page               |           4 phút |
| deploy_blip      | none     | False page    | Không page         |                — |
| slow_leak        | ticket   | Không ticket  | Ticket, không page |       1,337 phút |
| degraded_evening | page     | Page          | Page               |         291 phút |
| night_spike      | none     | False page    | Không page         |                — |
| small_bump       | none     | False page    | Không page         |                — |

Error budget 30 ngày: đã tiêu **42.95%**, còn **57.05%**. Không cần đóng băng release vì `freeze_releases=false`. Naive alerter có 7 page, 0 ticket, 3 false page; burn-rate có 3 page, 1 ticket, 0 false page. `small_bump` (20 phút ở 12% lỗi) chiếm xấp xỉ 1% budget tháng theo dữ liệu bài, không cho thấy burn kéo dài nên không nên đánh thức on-call lúc 11 giờ trưa.

## 6. Postmortem — Băng G2 với client naive (blameless)

_Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`._

**Tóm tắt (3 câu):** Tại t=120, alpha mất 75% năng lực và client tiếp tục retry nhiều tầng vào cùng provider. Alpha hồi phục ở t=300 nhưng hàng đợi còn request của các attempt đã timeout, trong khi retry mới tiếp tục đến. Hệ thống chỉ đạt lại ngưỡng 95% thành công sau phục hồi ở t=455, với tổng cộng 1,285 lượt user thất bại.

**Ảnh hưởng:** 1,285/2,362 lượt user thất bại (54.4%); suy giảm provider kéo dài 180 giây và ảnh hưởng khách hàng kéo dài khoảng 335 giây từ t=120 đến recovery t=455. Theo tải cấu hình 4 lượt/giây trong 30 ngày, tháng có 10,368,000 lượt và ngân sách lỗi 0.5% là 51,840 lượt; sự cố này tiêu thụ khoảng **2.48%** ngân sách tháng.

**Timeline** (giây ảo — lấy từ CSV):

|       t | Sự kiện                              | Bằng chứng (số)                                                                                                                                  |
| ------: | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
|     120 | alpha mất 75% năng lực               | Từ t=120–149 có 2,723 attempt (90.77/s), peak 254/s; 51/98 logical calls thành công.                                                             |
| 150–299 | retry storm duy trì quá tải          | 24,702 attempt (137.23/s); chỉ 51/729 call hoàn tất thành công (7.0%); riêng t=150 có 163 attempt, 0 thành công.                                 |
|     300 | alpha hồi phục hoàn toàn             | CSV vẫn ghi 115 attempt alpha và 0/5 call thành công trong giây đó.                                                                              |
| 300–454 | hệ thống vẫn chưa hồi phục — vì sao? | 7,625 attempt (49.19/s); chỉ 8/554 call thành công (1.4%). Timeout không hủy công việc provider đang xử lý, backlog và retry mới tự duy trì tải. |
|     455 | đạt recovery                         | Recovery metric là 155 s sau t=300; cửa sổ 30 s bắt đầu tại t=455 đạt ít nhất 95% call thành công.                                               |

**Trigger vs root cause:** Trigger là alpha mất 75% GPU. Root cause là ba tầng retry chồng nhau, delay cố định, không retry budget và không breaker. Timeout phía client không dừng request phía provider; retry lại đổ vào alpha, tạo vòng lặp queue dài → timeout → retry → queue dài hơn. Tổng 34,475 attempt đều vào alpha; có 3,194 `overloaded_error`, 2,737 `client_timeout`, availability 45.60% và amplification 9.904.

**Điều gì giúp / điều gì làm tệ hơn:** Khôi phục GPU ở t=300 xử lý trigger nhưng không xóa backlog; tải retry vẫn trung bình 49.19 attempt/s trong t=300–454. Thiết kế resilient dùng một tầng retry, jitter, budget, breaker và fallback nên trên G2 đạt availability 100%, amplification 1.035 và recovery 0 s.

**Action items** (cụ thể, có owner, đo được):

| #   | Hành động                                                                              | Loại (prevent / detect / mitigate) | Owner              | Cách kiểm chứng                                                                          |
| --- | -------------------------------------------------------------------------------------- | ---------------------------------- | ------------------ | ---------------------------------------------------------------------------------------- |
| 1   | Chỉ cho client reliability layer sở hữu retry; bắt buộc full jitter và retry budget.   | prevent                            | Agent Platform     | CI: G2 amplification ≤1.2, G3 retry ratio ≤0.10 trên 3 seed.                             |
| 2   | Cảnh báo theo success ratio và attempt rate của provider, không chỉ availability cuối. | detect                             | SRE                | Diễn tập G2: cảnh báo trong 30 s khi success dưới 95% hoặc alpha attempt vượt công suất. |
| 3   | Giữ timeout hữu hạn, breaker slow-call và fallback tương thích.                        | mitigate                           | Inference Platform | G2 availability ≥95%, recovery ≤30 s, alpha share sau phục hồi ≥80% trên 3 seed.         |

## 7. Một điểm yếu còn lại

G2 resilient vẫn có p95 9.922 s, khá cao khi một lượt user cần cả tool và chat; G6 cost/turn là 2.210 do spillover sang beta (so với 2.072 ở G4). Có thể thêm admission/concurrency control dựa trên queue và đo latency theo provider để spillover sớm hơn; xác nhận G2 p95 giảm mà availability vẫn ≥95% và recovery ≤30 s.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client                  | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
| ----------------------- | -----------: | ---------------: | --------------: | -----------: | ------------: | ---------------: | --------: | :---: |
| resilient (không cache) |       0.8373 |           0.8424 |           0.000 |            0 |             0 |                0 |     1.386 | FAIL  |
| cached (của bạn)        |       0.9124 |           0.9645 |           0.652 |          157 |             0 |                3 |     0.915 | PASS  |

| Tham số `cache:`     |          Giá trị | Vì sao (số liệu nào)                                                                                   |
| -------------------- | ---------------: | ------------------------------------------------------------------------------------------------------ |
| similarity_threshold |  0.93 (hiện tại) | Ngưỡng 0.93 loại các câu gần nghĩa nhưng khác ý; C1 có 0 câu sai, hit ratio 0.652.                     |
| ttl_s                |  60 s (hiện tại) | C1 có 3 câu outdated; TTL 30 s còn 2, TTL 300 s tăng lên 95.                                           |
| max_stale_s          | 300 s (hiện tại) | Stale-if-error giúp FAQ availability đạt 0.9645, phục vụ 157 câu stale; outdated là 3, dưới ngưỡng 15. |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Chạy C1 với `similarity_threshold` = 0,80 / giá trị của bạn / 0,97. Ghi hit ratio, `wrong_answers`, cost.
   Câu nào bị trả nhầm ở 0,80 (gợi ý: `wrong_answers_by_reason`, các cặp "gần mà khác")?

   Trong phép so sánh giữ TTL=60 s và `max_stale_s=300`, ngưỡng 0.80 có hit 0.666, 157 câu sai và cost/turn 0.965. Ngưỡng 0.93 có hit 0.652, 0 câu sai, cost 0.915; ngưỡng 0.97 có hit 0.581, 0 câu sai, cost 0.964. Các near miss như refund policy/refund time và phí ship nội thành/đi tỉnh bị gộp nhầm ở 0.80; cấu hình hiện tại 0.80 còn cho 181 câu sai do TTL/stale khác phép so sánh.

2. **TTL.** Chạy với `ttl_s` = 30 / 60 / 300. Vì sao webhook không giúp được gì cho `outdated_answers`, và TTL là thứ
   duy nhất giới hạn nó?

   Giữ threshold=0.93 và stale=300 s: TTL 30/60/300 cho hit ratio 0.592/0.652/0.731, outdated 2/3/95 và cost/turn 0.955/0.915/0.863. Webhook tại t=150 invalidate nguồn refund, nhưng shipping đổi tại t=480 không có webhook; TTL là giới hạn duy nhất cho câu trả lời shipping cũ, nên 300 s vượt ngưỡng outdated 15.

3. **Stale-if-error.** Chạy với `max_stale_s: 0`. `faq_availability` thay đổi thế nào?

   Với threshold=0.93, TTL=60 s, bỏ stale window làm FAQ availability giảm từ 0.9645 xuống 0.8733, stale served từ 157 xuống 0 và hit ratio từ 0.652 xuống 0.561. Cấu hình YAML hiện tại cũng đặt stale=0 nhưng threshold/TTL khác nên FAQ availability là 0.8587.

Trả lời:

- Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") **không** được cache, kể cả khi cache theo từng user?

  Vì trạng thái đơn, hoàn tiền và điểm có thể thay đổi nhanh; key theo user chỉ ngăn lộ chéo, không làm dữ liệu cũ thành đúng hay bảo đảm mọi thay đổi đều có webhook. Cache cá nhân còn có nguy cơ phục vụ thông tin riêng tư đã stale.

- Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?

  Theo dõi provider success/error rate, thời gian breaker OPEN, fallback rate và số/tỷ lệ stale-served cùng availability. Availability được cache giữ cao không thay thế tín hiệu sức khỏe provider; ở C1 cấu hình thử đạt FAQ availability 0.9645 nhưng vẫn phục vụ 157 câu stale khi outage.
