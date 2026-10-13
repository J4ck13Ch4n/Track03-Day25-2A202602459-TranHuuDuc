# Lab 25B — Report

**Họ tên / nhóm:** Trần Hữu Đức — 2A202602459  **Commit:** (điền sau khi commit)  **Điểm tự động (`make grade`):** 84 / 88

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | ✗ |
| G1 | yours | 1.0 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | ✓ |
| G2 | naive | 0.456 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | ✗ |
| G2 | yours | 1.0 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | ✓ |
| G3 | naive | 1.0 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | ✗ (retry_ratio) |
| G3 | yours | 1.0 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | ✓ |
| G4 | naive | 1.0 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | ✗ (p95) |
| G4 | yours | 1.0 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | ✓ |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | ✗ |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | — | ✓ |
| G6 | naive | 0.517 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | ✗ |
| G6 | yours | 1.0 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | ✓ |

6/6 băng PASS với `ResilientClient`. Naive thất bại trên cả 6 — chủ yếu vì khuếch đại tải (amplification 7.7–17×) và không phân loại lỗi (G5: client_error_retries, G3: retry_ratio quá cao vì không phân biệt 500 nhất thời với lỗi cần failover).

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | service_median_s cao nhất là `beta` (1.8 s); 8 s đủ cho cả trường hợp chậm + hàng đợi mà không chờ hết 30 s deadline. Thử 30 s ở Exp 2: availability G6 giảm 1.0 → 0.9488 vì client "treo" quá lâu trên `alpha-eu` đang hang. |
| max_attempts_per_provider | 2 | 1 lần gửi + 1 retry trên cùng provider trước khi fail over: đủ để vượt lỗi tức thời (G3 brown-out: retry_ratio chỉ 0.034) mà không khuếch đại tải khi provider thật sự quá tải (G2 amplification 1.041). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Với max_attempts=2 retry duy nhất có jitter ∈ [0, 0.25 s] — đủ ngắn để p95 G3 vẫn 4.66 s, đủ để rải tải không đồng bộ. |
| retry_budget_ratio | 0.1 | Giữ retry ≤ ~10% traffic bất kể incident tệ ra sao. Exp 1 (breaker bị cấu hình sai) cho thấy budget chặn amplification ở 1.38 thay vì 1.68 khi tắt budget — lớp phòng thủ cuối khi breaker chưa/không trip kịp. |
| breaker.window_size / min_calls | 20 / 10 | Đủ lớn để không trip vì lỗi nền (base_error_rate 0,2%), đủ nhỏ để phản ứng trong vài giây ở traffic ~4 turn/s × 2 call/turn. |
| breaker.failure_rate_threshold | 0.5 | Cùng cấp độ với check G3 (error_rate 30%) — trip khi > nửa kết quả gần nhất fail, không trip vì brown-out 30%. |
| breaker.slow_call_s | 5.0 (slow_rate_threshold 0.6) | Bắt G4 (alpha chậm thêm 5 s → ~6–7 s) sớm, trước khi user chạm ngưỡng 30 s; p95 G4 chỉ 5.56 s nhờ failover trước khi user cảm nhận chậm. |
| breaker.open_duration_s | 10.0 | Khớp `max_queue_wait_s=10s` của provider — đủ để hàng đợi rút trước khi probe lại; recovery_s = 0 ở G2/G6. |
| breaker.half_open_max_probes / half_open_successes | 2 / 2 | Giới hạn probe để tránh thundering herd lên provider mới hồi phục; cần 2 probe liên tiếp thành công mới đóng, tránh đóng sớm rồi mở lại (flapping). |

Sơ đồ quyết định của `on_error`:

```
err -> classify(err)
 ├─ CLIENT_ERROR        -> breaker.release  -> GiveUp("client_error")           [không retry, không fail over]
 ├─ QUOTA_EXHAUSTED      -> breaker.release  -> cooldown[provider]=now+300s     -> fail over ngay
 ├─ RATE_LIMITED         -> breaker.release  -> cooldown[provider]=now+retry_after -> fail over ngay
 └─ OVERLOADED / SERVER_ERROR / TIMEOUT
         -> breaker.on_failure(permit)
         -> tries < max_attempts_per_provider  AND  budget.try_acquire_retry()?
              ├─ có    -> Send(cùng provider, delay = backoff.delay(tries))
              └─ không -> fail over (idx += 1, tries = 0, delay = 0)
```

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối.** Với breaker bị cấu hình sai (`min_calls=1e9` — không bao giờ trip) và `max_attempts_per_provider=5`:

   | Băng | | amplification | retry_ratio | availability | recovery (s) |
   |---|---|---:|---:|---:|---:|
   | G2 | có budget | 1.383 | 0.058 | 0.9987 | 0 |
   | G2 | không budget | 1.677 | 0.503 | 0.9606 | 32 |
   | G3 | có budget | 1.136 | 0.068 | 1.0 | 0 |
   | G3 | không budget | 1.163 | 0.162 | 1.0 | 0 |

   Khi breaker *hoạt động đúng* (cấu hình gốc), có/không budget cho số gần như giống nhau (G2 amplification 1.040 vs 1.029; G3: 1.059 vs 1.056) — breaker đã chặn provider lỗi trước khi lượng retry chạm 10% ngân sách. Vẫn nên giữ cả hai: breaker là tuyến phòng thủ chính (phản ứng theo tỷ lệ lỗi, biết provider nào đang khoẻ), còn budget là lưới an toàn độc lập không phụ thuộc cấu hình breaker — khi breaker sai (ví dụ `min_calls` đặt nhầm quá lớn như trên), budget vẫn tự nó giữ retry_ratio ở 0.058–0.068 thay vì 0.16–0.50, và giữ recovery ở 0s thay vì 32s.

2. **`attempt_timeout_s: 30`.**

   | Băng | | p95 (s) | availability |
   |---|---|---:|---:|
   | G4 | timeout=8 | 5.604 | 1.0 |
   | G4 | timeout=30 | 5.683 | 1.0 |
   | G6 | timeout=8 | 6.016 | 1.0 |
   | G6 | timeout=30 | 5.306 | 0.9488 |

   G4 gần như không đổi: alpha vẫn trả lời (không lỗi), nên timeout dài không có tác dụng — vấn đề của G4 là breaker bắt call chậm (`slow_call_s`), không phải timeout. G6 thì tệ hẳn: `alpha-eu` đang "hang" (không bao giờ trả lời) trong sự cố; timeout 30 s khiến client chờ gần hết 30 s deadline của user mới phát hiện và fail over, availability rơi từ 1.0 xuống 0.9488. Timeout "đúng" nên dựa trên `service_median_s` + vài độ lệch chuẩn của provider chậm nhất trong ladder (beta ≈ 1.8 s × (1+3×0.35) ≈ 3.7 s), cộng biên an toàn — không nên dựa theo deadline của user, vì khi provider hang, mỗi giây timeout dài thêm là một giây user chờ vô ích.

3. **Bỏ lọc tool calling khỏi `ladder()`, chạy G6.**

   `incompatible_routes`: 0 → 258; `availability`: 1.0 → 0.8918.

   Góc nhìn PM: khi `alpha`/`alpha-eu` sập, hệ thống sẽ route ~258 yêu cầu cần "gọi tool" (ví dụ tra đơn hàng, xử lý hoàn tiền) sang `gamma-mini` — model rẻ nhưng không hỗ trợ tool calling. Model này sẽ trả lời bằng văn bản thay vì thực thi hành động, nên ~11% người dùng (availability giảm 100% → 89%) nhận một câu trả lời *trông như thành công* nhưng thực chất sai định dạng và không làm được việc họ cần — một lỗi im lặng, khó phát hiện qua log lỗi thông thường, và tốn chi phí xử lý khiếu nại về sau nhiều hơn một lỗi báo động rõ ràng (ví dụ "hệ thống đang bận, thử lại sau").

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1.574.200 ₫* | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

\*đơn vị `amount_cents`, tức 15.742 "đơn vị tiền" quy đổi. Số `worker_crashes` khác nhau giữa hai lần chạy vì executor tiêu thụ RNG theo nhịp khác nhau (số lần gọi `_retry` khác nhau), không phải do thực thi sai.

Chỉ idempotency key là **chưa đủ**: nó chỉ bảo vệ *một lệnh gọi service cụ thể* khỏi bị thực thi hai lần, nhưng worker vẫn không biết bước đó đã **xong** để khỏi gọi lại nguyên chuỗi (`classify_request`, `lookup_order`...) từ đầu sau mỗi lần restart — tốn thêm lệnh gọi LLM vô ích (chính là `wasted_llm_per_crash`). Chỉ checkpoint cũng **chưa đủ**: nếu không có key ổn định theo (session, step), một lần gọi lại sau crash sẽ mang một payload/khoá khác → service thanh toán coi là một thao tác mới → double-charge.

Nếu ghi checkpoint **trước** khi gọi payment: một crash xảy ra đúng lúc giữa "ghi checkpoint" và "thực thi payment thật" sẽ để lại một checkpoint nói bước đã xong, nhưng tiền chưa hề được hoàn — worker sau khi resume sẽ **bỏ qua** bước này (vì thấy checkpoint tồn tại) → khách hàng mất tiền hoàn mà không ai biết (`lost_refunds` tăng). Vì vậy thứ tự đúng luôn là: **thực thi xong rồi mới checkpoint**.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (burn-rate) |
|---|---|---|---|---:|
| checkout_bug | page | page (trễ 1′) | page | 27′ |
| provider_outage | page | page (trễ 0′) | page | 4′ |
| deploy_blip | none | **false page** | không page | — |
| slow_leak | ticket | không phát hiện (naive không có ticket) | ticket | 1337′ |
| degraded_evening | page | page (trễ 214′) | page | 291′ |
| night_spike | none | **false page** | không page | — |
| small_bump | none | **false page** | không page | — |

Naive (ngưỡng 5%/5 phút cố định): 7 page, **3 false page** → fail. Burn-rate (đa cửa sổ): 3 page + 1 ticket, **0 false page** → pass.

Error budget 30 ngày (dữ liệu `data/sli_30d.csv`, SLO 99,5%): đã tiêu **42,95%**, còn **57,05%** — `freeze_releases = False`. `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa vì: (a) quá ngắn để cả cửa sổ dài (60 phút/6 giờ/3 ngày) của mọi rule vượt ngưỡng burn-rate — chỉ là một "blip"; (b) chỉ tiêu ~1% ngân sách lỗi cả tháng (theo chú thích trong `data.py`) — không đáng kể so với 57% ngân sách còn lại. Page cho những sự cố nhỏ như vậy chỉ gây mệt mỏi cảnh báo (alert fatigue) khiến on-call lơ là những page thật sự quan trọng.

## 6. Postmortem — Băng G2 với client naive (blameless)

**Tóm tắt:** `alpha` mất 75% công suất GPU trong 3 phút ảo (t=120–300 s). `NaiveClient` (3 tầng retry chồng nhau, không jitter, không breaker) tiếp tục dội toàn bộ traffic vào `alpha` dù nó đang quá tải. Khi `alpha` hồi phục hoàn toàn ở t=300, hệ thống **không** tự hồi phục ngay — availability vẫn bằng 0 cho tới khoảng t≈455 (155 s sau khi provider đã khỏe), biến một sự cố hạ tầng 3 phút thành gần 5,6 phút sập hoàn toàn cho người dùng.

**Ảnh hưởng:** trong toàn băng (duration 600 s), availability đo được chỉ **45,6%** (yêu cầu ≥95%), amplification **9,9×** (yêu cầu ≤1,6), `cost_per_good_turn` = 4,627 — gấp **2,2×** so với bản `ResilientClient` trên cùng băng (2,119): mỗi lượt *thành công* phải trả tiền cho rất nhiều attempt bị bỏ phí ở những lượt khác.

**Timeline** (từ `reports/gameday/naive_G2_overload_storm_timeseries.csv`):

| t (s) | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 110 | Trước sự cố, hệ thống bình thường | calls_ok/calls_done = 9/9 (100%), attempts_alpha = 5/s |
| 120 | `alpha` mất 75% công suất (capacity_factor=0,25) | effect bắt đầu theo `scenarios.yaml` |
| 130 | Retry bắt đầu chồng chéo | attempts_alpha nhảy lên 10/s trong khi calls_done vẫn chỉ 3 |
| 150 | Availability rơi về 0 | calls_ok=0/3; attempts_alpha=163/s (retry storm toàn tốc) |
| 200–299 | Provider ngập trong retry vô ích | attempts_alpha duy trì 130–184/s, calls_ok=0 suốt — mọi request timeout trước khi được xử lý xong (hàng đợi > `max_queue_wait_s=10s`) |
| **300** | `alpha` hồi phục hoàn toàn (effect kết thúc) | capacity_factor trở lại 1,0 — nhưng attempts_alpha vẫn 115/s, calls_ok vẫn 0 |
| 301–450 | Hệ thống **chưa** hồi phục dù provider đã khỏe | attempts_alpha dao động 6–122/s, calls_ok=0 liên tục |
| ≈455 | Availability bắt đầu hồi phục | `recovery_s = 155` (155 s sau t=300) theo `naive.json` |
| 480+ | Hồi phục hoàn toàn | calls_ok/calls_done ≈ 1,0 |

**Trigger vs root cause:** Trigger = `alpha` mất 75% GPU trong 3 phút — sự cố hạ tầng phía provider, không phải lỗi của ShopAgent. Root cause = `NaiveClient` retry 3 tầng chồng nhau (tối đa 36 attempt/call), không jitter, không retry budget, không circuit breaker để ngắt traffic khỏi provider đang quá tải. Lượng retry do chính client tạo ra vượt xa năng lực còn lại của `alpha`, nên hàng đợi của provider không bao giờ kịp rút ngắn: mỗi request mới vẫn chờ lâu hơn 10 s rồi bị 529, client vẫn retry gần như ngay lập tức → vòng lặp tự duy trì (metastable failure, Bronson et al. HotOS 2021) ngay cả sau khi nguyên nhân gốc (mất GPU) đã biến mất ở t=300, vì chính cơ chế retry là thứ bây giờ giữ tải vượt công suất — cộng với việc server "vẫn tiếp tục xử lý" các request mà client đã bỏ cuộc, nên mọi công sức đó bị lãng phí hoàn toàn (theo đúng cơ chế mô tả ở mục 4 README).

**Điều gì làm tệ hơn:** (1) 3 tầng retry độc lập không ai biết tầng khác đang retry → nhân nhau thành 36 attempt/call; (2) delay cố định không jitter → tất cả client quay lại đồng loạt cùng lúc (thundering herd); (3) không nhận biết deadline/timeout của server → mỗi retry là công sức thuần lãng phí, không giải phóng được capacity thật.

**Điều gì giúp (nếu có):** `alpha` vẫn còn 25% công suất gốc — nếu không có retry storm, hệ thống lẽ ra chỉ cần giảm nhẹ throughput tương ứng (~25%), không sập hoàn toàn.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai circuit breaker sliding-window trên mọi provider (Part B/C) | prevent | Reliability/Platform | `make compare`: G2 amplification ≤1,6 trong CI (đã đạt 1,041, recovery_s=0) |
| 2 | Thêm retry budget (token-bucket, ratio≤10%) làm lớp phòng thủ cuối khi breaker bị cấu hình sai | mitigate | SRE/On-call | Lặp lại Exp 1 (breaker vô hiệu) trong CI, assert retry_ratio ≤0,1 bất kể cấu hình breaker |
| 3 | Thêm burn-rate alerting đa cửa sổ (Part E) để page trong vài phút khi amplification/retry_ratio vượt ngưỡng | detect | SRE | `make slo`: page một sự cố dạng overload trong ≤6 phút, giống `fast_burn_2pct_1h` |
| 4 | Gộp retry về đúng một tầng (ứng dụng), bỏ retry ở gateway/SDK nếu không kiểm soát được | prevent | Platform | Audit code, đếm số tầng retry độc lập còn lại = 1 |

## 7. Một điểm yếu còn lại

`ResilientClient` vẫn có p95 cao hơn hẳn các băng khác ở G2 (10,92 s so với 4,66–5,96 s ở G1/G3/G4/G6), dù availability vẫn đạt 100%. Nguyên nhân: breaker cần đủ `min_calls=10` kết quả gần nhất trong window mới xét trip, nên một số lượt *đầu* của sự cố (ngay sau t=120) vẫn phải nếm hàng đợi gần chạm `max_queue_wait_s=10s` hoặc một vòng backoff trước khi breaker kịp mở và client failover — kéo dài đuôi p95 dù đa số lượt vẫn thành công. Chi phí cũng nhích lên (cost_per_good_turn 2,119 so với 2,101 ở G1) vì một phần traffic trong lúc đó đổ sang `beta` (cost_per_call 1,4 so với `alpha` 1,0).

Cách sửa: thêm một "fast trip" riêng cho các lỗi cấp bách (529/overloaded liên tiếp) độc lập với sliding window chung — ví dụ trip ngay nếu N lỗi 529 *liên tiếp* bất kể window hiện có bao nhiêu kết quả, để phản ứng nhanh hơn trong vài giây đầu của một đợt mất năng lực đột ngột. Đánh đổi: rủi ro trip giả cao hơn với các brown-out ngắn hoặc nhiễu nền (vài lỗi 529 ngẫu nhiên) — cần kiểm tra lại không vi phạm `retry_ratio_max` của G3.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.0 | 0 | 0 | 0 | 1.386 | ✗ (faq_availability, cache_hit_ratio) |
| cached (của bạn) | 0.9113 | 0.9628 | 0.651 | 154 | 0 | 3 | 0.917 | ✓ |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Starter 0,80 cho 171 `wrong_answers` (toàn bộ `other_question`): các cặp intent "gần mà khác" (refund_policy ↔ refund_time, shipping_city ↔ shipping_province…) có cosine centroid ~0,86, vượt 0,80 nên bị trộn lẫn câu trả lời. 0,92 cho `wrong_answers=0`, hit ratio 0,651 — tốt hơn 0,97 (hit chỉ 0,576, cost/turn cao hơn: 0,966) vì 0,97 loại luôn cả vài paraphrase hợp lệ (cosine 0,95–0,97). |
| ttl_s | 60 | Sweep 30/60/300 cho `outdated_answers` = 2/3/95 (faq_availability gần như không đổi, ~0,963). Silent update (bảng phí ship đổi ở t=480, không webhook) chỉ bị TTL giới hạn — ttl=300 vượt xa ngưỡng 15 cho phép, ttl=60 giữ ở 3, có biên an toàn rộng. |
| max_stale_s | 150 | Outage kéo dài 120 s (t=240–360). Với `max_stale_s=0` (như starter), `faq_availability` rơi từ 0,9628 xuống **0,8733** vì mọi FAQ trong outage mất chỗ bám hoàn toàn. 150 s (tổng cửa sổ tươi+stale = 210 s) dư so với 120 s outage mà không giữ dữ liệu cũ quá lâu. |

Thí nghiệm:

1. **Ngưỡng.** `similarity_threshold` = 0,80 → hit=0,659, `wrong_answers`=171 (100% `other_question` — các cặp câu hỏi "gần mà khác" như refund_policy/refund_time có cosine ~0,86, vượt ngưỡng 0,80 nên bị trộn lẫn). Giá trị của tôi (0,92) → hit=0,651, wrong=0. 0,97 → hit=0,576 (giảm vì loại luôn một số paraphrase hợp lệ cosine 0,95–0,97), wrong=0, cost/turn tăng nhẹ (0,966 so với 0,917) vì phải gọi provider nhiều hơn.
2. **TTL.** ttl_s = 30/60/300 → `outdated_answers` = 2/3/95; hit ratio tăng dần theo ttl (0,591/0,651/0,731) — đánh đổi trực tiếp giữa hit ratio và độ tươi dữ liệu. Webhook (`content_updates`) chỉ invalidate các source được CMS báo trước (policy/refund ở t=150); bảng phí ship đổi "âm thầm" ở t=480 không đi qua đường này, nên TTL là chốt chặn **duy nhất** cho loại thay đổi không báo trước.
3. **Stale-if-error.** `max_stale_s=0` → `faq_availability` giảm từ 0,9628 xuống **0,8733**, `stale_served=0` (đúng vì không còn cửa sổ stale). Đúng như dự đoán ở README (mục 11 FAQ): không có stale-if-error thì mọi FAQ rơi vào lúc 4 provider cùng sập (240–360 s) đều thất bại hoàn toàn.

**Vì sao câu hỏi cá nhân không được cache, kể cả theo từng user?** Hai lý do: (a) dữ liệu cá nhân (đơn hàng/tiền hoàn/điểm) thay đổi liên tục, không có tín hiệu invalidation nào từ hệ thống đặt hàng/thanh toán — TTL không đảm bảo tươi; (b) rủi ro **lộ dữ liệu**: hai câu hỏi của hai user khác nhau ("đơn #123" vs "đơn #124") có cosine ~0,99 (do cách sinh embedding trong `faq.py`), rất dễ vượt ngưỡng similarity và trả nhầm thông tin đơn hàng/tiền hoàn của **người khác** — đây là rò rỉ dữ liệu, nghiêm trọng hơn một câu trả lời sai thông thường. Vì vậy `SemanticCache.lookup`/`store` chặn personal ngay từ đầu, không đặt allowlist theo user.

**Alert dựa trên số nào khi dashboard vẫn "xanh" nhờ cache lúc outage?** `faq_availability` tổng thể có thể vẫn cao nhờ cache trong khi mọi provider đã sập — không phải tín hiệu đáng tin trong lúc outage. Nên theo dõi riêng: (1) availability của các lượt **không phải FAQ** (cần tool-call, luôn phải đi qua provider thật) — nó sẽ rơi ngay khi outage xảy ra; (2) `stale_served` / tỷ lệ stale trong `cache_hit_ratio` tăng vọt bất thường — dấu hiệu cache đang "che" một outage; (3) breaker state / `errors_by_code` của các provider tăng vọt trực tiếp. Dựa một mình vào `faq_availability` hay `availability` tổng có thể khiến on-call không biết có sự cố thật đang diễn ra phía sau các câu trả lời "đúng nhưng cũ".
