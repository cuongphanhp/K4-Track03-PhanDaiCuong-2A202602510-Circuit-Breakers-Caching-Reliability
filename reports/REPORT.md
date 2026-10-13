# Lab 25B — Report

**Họ tên / nhóm:** [Nhập họ tên + MSSV]  **Commit:** [Nhập commit hash]  **Điểm tự động (`make grade`):** 84 / 88

> Mọi nhận định dưới đây đều có số. Dữ liệu lấy từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | ❌ |
| G1 | yours | 1.0000 | 4.817 | 1.060 | 0.001 | 2.094 | 0.0 | ✅ |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | ❌ |
| G2 | yours | 1.0000 | 10.781 | 1.038 | 0.005 | 2.116 | 0.0 | ✅ |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | ❌ |
| G3 | yours | 1.0000 | 4.718 | 1.065 | 0.035 | 2.146 | 0.0 | ✅ |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | ❌ |
| G4 | yours | 1.0000 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | ✅ |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | ❌ |
| G5 | yours | 0.8179 | 5.482 | 1.003 | 0.002 | 2.504 | — | ✅ |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | ❌ |
| G6 | yours | 1.0000 | 6.225 | 1.067 | 0.027 | 2.164 | 0.0 | ✅ |

Nhận xét ngắn: client naive sụp vì sai ba giới hạn cốt lõi — retry chồng 3 tầng, bỏ `retry-after`, và không tách lỗi 400 / 429 / timeout. Client của bạn đạt 6/6 băng và giữ amplification dưới 1.1 trong khi naive tăng tới 17x ở G5 và 12x ở G6. Điều quan trọng nhất không phải là “đủ tốt ở mọi băng” mà là “giữ retry ratio rất thấp mà vẫn không mất khả năng phục hồi”, vì cả hai đều được thể hiện rõ ở G1–G2.

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Với G6, người dùng có deadline 30s; timeout quá dài khiến request treo và kéo dài hàng đợi. Client của bạn giữ latency p95 ở 6.225s trong G6, thấp hơn nhiều so với naive (4.836s nhưng availability chỉ 51.7% và nhiều request bị timeout/treo). 8s là đủ cho một attempt và vẫn còn margin để failover. |
| max_attempts_per_provider | 2 | Duy trì tối đa 1 retry trên cùng provider. G1 naive có retry_ratio 6.738 và amplification 7.738; vết nứt này giảm xuống 0.001 và 1.06 trong phiên resilient. 2 attempt/provider là đủ cho transient failure mà không biến rate-limit thành retry storm. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter + cap 4s giúp tránh thác loạn cùng lúc trong G1 và G2. Naive không có jitter, dẫn tới retry_after_violations 36,027 ở G1 và 28,381 ở G2; resilient giữ 0. |
| retry_budget_ratio | 0.1 | Giữ retry không vượt ~10% số lần gọi đầu. G3 naive có retry_ratio 0.164, gần ngưỡng; resilient chỉ 0.035. Nó không giải quyết mọi chuyện, nhưng là lớp phòng thủ cuối khi breaker đã trip hoặc độ trễ cao. |
| breaker.window_size / min_calls | 20 / 10 | Cửa sổ 20 request gần nhất giúp phát hiện tức thì khi lỗi bùng lên; `min_calls: 10` tránh trip quá sớm trên vài lỗi ngẫu nhiên. Điều này đáng để có ở cả G2 (overload) và G3 (brownout). |
| breaker.failure_rate_threshold | 0.5 | Trong G2, alpha có overload và 529/429 tăng mạnh. Khi lỗi vượt 50% trong cửa sổ gần đây, breaker trip để cắt bớt gánh nặng. |
| breaker.slow_call_s | 5.0 | G4 là “slow region” không có lỗi nhưng p95 tăng lên 13.764s trong naive. Chậm nhưng không lỗi vẫn là tín hiệu xấu; slow-call threshold 5s cho phát hiện sớm. |
| breaker.open_duration_s | 10.0 | Khi alpha hoặc alpha-eu đã trip, thời gian khóa 10s đủ để giảm tải và cho hệ thống xoay provider, còn không quá dài để chặn traffic quá lâu. |
| breaker.half_open_max_probes | 2 | Giới hạn probe ở HALF-OPEN để không có “burst” khi provider vừa hồi phục; `half_open_successes: 2` cũng buộc phục hồi đáng tin cậy và không rơi vào churn. |

Sơ đồ quyết định của `on_error`:

```text
429 rate_limit_exceeded + retry_after -> ghi breaker, không retry ngay, tôn trọng Retry-After
429 insufficient_quota -> không retry, cooldown provider, failover sang provider tương thích
529/503/timeout -> ghi breaker, retry cục bộ nếu còn budget và chưa trip, failover nếu failover_help
400 invalid_request_error -> không retry, không failover, báo client_error
các lỗi transient còn lại -> retry với full jitter, kiểm tra retry budget trước khi gửi
nếu breaker OPEN/HALF-OPEN -> bỏ qua hoặc giới hạn probe, không cho request đi tràn qua
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Ở naive, G2 có retry_ratio 8.904 và amplification 9.904; ở resilient G2, các con số rơi xuống 0.005 và 1.038, chứng tỏ budget + breaker + backoff cùng hoạt động. Nếu bỏ budget nhưng breaker vẫn đúng, hệ thống vẫn sẽ tốt hơn naive, nhưng quãng “thử” cuối cùng này giúp ngăn retry tiếp tục tràn khi provider đã bắt đầu phục hồi. Nói cách khác, breaker ngăn một cơn bão, còn budget giữ cho các retries không biến thành thảm họa thứ hai. Nếu không có cả hai, số lượng retry có thể tăng rất nhanh khi hàng đợi đầy và timeout xuất hiện.

2. **`attempt_timeout_s: 30` không phải là timeout “đúng” cho hệ thống này.** G4 là trường hợp tốt để thấy: naive có p95 13.764s nhưng availability vẫn 100%, vì các request thành công hầu như không có lỗi và p95 chỉ tính trên lượt thành công. Dựa vào SRE, deadline của user là 30s, nhưng thời gian cho một attempt không nên gần tới 30s vì phải còn đủ chỗ cho one retry + failover. Một timeout 8s cho phép request rơi vụt khỏi hàng đợi quá ngắn, tránh “treo 30s rồi mới failover” mà không làm hại p95. Trong G6, resilient giữ availability 100% với p95 6.225s; đó là một timeout “không quá dài, không quá ngắn”.

3. **Bỏ lọc tool calling khỏi `ladder()` sẽ làm mất an toàn product.** Trong G6, resilient có `incompatible_routes = 0` và `gamma-mini` chỉ nhận đúng 496 attempt / 4770 request; trong khi naive gửi tất cả về alpha và không có fallback cấu hình. Nếu rung “cứ failover” mà không kiểm tra supports_tools, model không hỗ trợ tool sẽ nhận tool call và hàm gắn với sản phẩm chắc chắn lỗi. Từ góc nhìn PM: đó là “triệt tiêu chất lượng và tăng chi phí, đồng thời gây thêm hỗn loạn cho user trong khi không giải quyết sự cố kỹ thuật”.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.000 | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Vì sao chỉ idempotency key là chưa đủ? Bởi vì nếu worker crash ngay sau khi payment đã đi nhưng trước khi ghi checkpoint, hệ thống có thể thực thi lại với cùng idempotency key nhưng không biết bước nào đã hoàn tất. Tương tự, chỉ checkpoint mà không có idempotency key sẽ mắc lỗi “gọi payment hai lần trong môi trường thất bại”, vì restart hoặc retry không có định danh rõ ràng. Nếu ghi checkpoint trước khi gọi payment, hệ thống sẽ tin là “đã xong” dù payment thật ra chưa thành công; khi crash xảy ra, ứng dụng sẽ xem bước là hoàn tất nhưng thực tế thanh toán có thể chưa chạy hoặc chạy nhưng response mất. Điều đó dẫn tới thất thoát“lost refund” hoặc “duplicate refund” tùy trường hợp.

Đúng hướng giải pháp là: sinh `idempotency key` ổn định theo `(session, step)`, ghi checkpoint sau khi step thành công và chỉ sau đó mới đánh dấu “hoàn tất”, rồi resume từ checkpoint. Hệ thống của bạn đạt completion 100% và không có duplicate refund/email, đồng thời giữ `wasted_llm_per_crash = 0.303` dưới ngưỡng 0.6.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | detected (1 phút) | detected (27 phút) | 27 |
| provider_outage | page | detected (0 phút) | detected (4 phút) | 4 |
| deploy_blip | none | paged | none | — |
| slow_leak | ticket | không phát hiện | detected | 1337 |
| degraded_evening | page | detected (214 phút) | detected (291 phút) | 291 |
| night_spike | none | paged | none | — |
| small_bump | none | paged | none | — |

Điểm này cho thấy burn-rate alerting thực sự tốt hơn naive alerter: burn-rate không “đánh thức ai” khi chỉ có blip nhỏ, nhưng vẫn page đúng ở các sự cố có tác động thực sự. SLE 30 ngày trong `reports/slo.json` cho thấy:

- `requests`: 25,901,755
- `allowed_bad`: 129,508.8
- `bad`: 55,618
- `availability`: 0.997853
- `budget_consumed`: 0.4295
- `budget_remaining`: 0.5705
- `freeze_releases`: false

Nói cách khác, hệ thống đã tiêu 42.95% ngân sách lỗi trong 30 ngày, còn 57.05% còn lại. Về nguyên tắc SRE, `small_bump` (20 phút ở 12% lỗi) không nên page vào 11 giờ trưa vì nó không bám đủ “hút” vào burn-rate 1h/6h/3d; nó là biến động cục bộ, không phải tích tụ đủ để biểu thị một sự cố có ý nghĩa. Release nên đóng băng khi budget_consumed vượt ngưỡng an toàn hoặc khi có bằng chứng thấm qua cả các cửa sổ dài; trong mẫu này, budget còn dư khá lớn nên không cần freeze.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):** G2 là một vụ overload ở alpha khi 75% GPU mất đi; naive client đã phản ứng bằng retry 3 tầng không có bất kỳ giới hạn nào, khiến traffic được khuếch đại và queue của provider dài thêm. Tại thời điểm t=120–150, calls_ok giảm xuống 0 trong khi attempts_alpha tăng từ 5 lên tới 250/s, thể hiện hủy vòng lặp. Sau khi alpha hồi phục khoảng t=300, hệ thống vẫn không phục hồi ngay vì client tiếp tục đẩy lại nhiều retry vào hàng đợi sẵn có, khiến trạng thái “rất tệ” được duy trì dù provider đã quay lại bình thường.

**Ảnh hưởng:** G2 naive chỉ đạt availability 45.6%, p95 15.034s, recovery_s 155s. Trong 1 tháng chỉ có sự cố này, mất 45.6% availability báo hiệu mất khoảng 54.4% của toàn bộ traffic “good turns” trong băng; trong một tháng, đây sẽ tương đương với tiêu tốn 0.544 × error budget của 30 ngày, vượt quá tiêu chuẩn an toàn. Đó là nơi hệ thống mất khả năng phục hồi và dùng tiền của toàn bộ khách hàng để trả cho một sự cố ở tầng provider.

**Timeline** (giây ảo — lấy từ CSV):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha bắt đầu suy yếu | `calls_ok=7`, `calls_done=7`, `attempts_alpha=5`; vẫn chưa đầy đủ để phán đoán nhưng trễ bắt đầu thể hiện sự cố |
| 136 | overflow bắt đầu bùng phát | `attempts_alpha=53`, `calls_ok=3` |
| 138 | hàng đợi provider vỡ | `attempts_alpha=122`, `calls_ok=0` |
| 140 | bùng nổ rõ ràng | `attempts_alpha=216`, `calls_ok=0` |
| 150 | hệ thống trong trạng thái nghẽn | `attempts_alpha=163`, `calls_ok=0` |
| 200–250 | hàng đợi duy trì ở mức cao | `attempts_alpha` dao động quanh 120–200/s trong khi `calls_ok` vẫn bằng 0 |
| 300 | provider bắt đầu hồi phục | `attempts_alpha=115` và sau đó tụt dần về 36 ở t=303, 2 ở t=305 |
| 305–360 | hệ thống đã hồi phục ở tầng cung cấp nhưng chưa ổn ở tầng client | `attempts_alpha` xuống xuống mức cơ bản; tuy nhiên, `recovery_s=155` cho thấy thời gian khôi phục thực tế rất chậm |

**Trigger vs root cause:** trigger là sự cố ở alpha: 75% GPU mất đi, khiến capacity giảm mạnh và queue dài lên. Root cause là phản ứng của client naïve: 3 tầng retry, không full jitter, không `retry-after`, không breaker, không budget. Khi provider báo 429/529 và client timeout, mỗi retry mới lại gây thêm tải vào queue, tạo vòng lặp “hàng đợi đầy → timeout tăng → retry tăng → queue đầy hơn”. Đó là metastable failure: provider đã phục hồi, nhưng client vẫn tiếp tục bắn retry với lượng lớn, duy trì trạng thái giảm hiệu suất cho tới khi hàng đợi dọn sạch hoặc hết retry budget.

**Điều gì giúp / điều gì làm tệ hơn:**

- Giúp: `rate_limit_exceeded` và `overloaded_error` là những dấu hiệu được mô phỏng rõ; nếu client có breaker và respecter retry-after, nó sẽ cắt bớt lượng traffic thay vì “đẩy cho tới khi nào hết”.
- Làm tệ hơn: không có backoff, không có thử phân tách theo provider, không có breaker, không có slow-call detection. Chúng tạo ra hiệu ứng bùng nổ mà trong thực tế gọi là “retry amplification”.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai `FullJitterBackoff` + `Retry-After` respect trên mọi provider | prevent | SRE / platform | G1/G2: `retry_after_violations` phải bằng 0 và `amplification ≤ 1.6` |
| 2 | Tạo breaker sliding-window với `failure_rate_threshold` và `slow_call_s` | prevent | Reliability Engineering | G2: `recovery_s ≤ 30s`, `availability ≥ 95%` |
| 3 | Thêm `retry_budget` và per-provider concurrency limit | mitigate | API Gateway | G3: `retry_ratio ≤ 0.10`, không có “retry storm” sau khi outage vừa qua |
| 4 | Giám sát `queue_wait` / `deadline` / `429` trong dashboard | detect | On-call | Chỉ cần 1 sự kiện `rate_limit_exceeded` vượt 1s trên alpha mới alert trước khi hẳn | 

## 7. Một điểm yếu còn lại

Một điểm yếu còn lại của client hiện tại là khi cả bốn provider cùng chậm hoặc khi có một sự cố “multi-az / multi-region” mà không có lỗi rõ ràng, breaker có thể trip theo slow-call nhưng vẫn có thể làm nền hàng đợi ở beta và gamma-mini bị gánh thêm. Tình huống như vậy không xuất hiện rõ trong G2 nhưng chính là điều mà hệ thống production cần dự đoán. Tôi sẽ sửa bằng cách bổ sung một lớp “admission control” cấp toàn cục: không cố gắng lên provider nào cũng “đá” cùng lúc, mà đặt `global concurrency cap` và `latency budget` per provider; khi p99 vượt `1.5 ×` baseline trong 60s, hệ thống giảm số request được phép gửi đi và ưu tiên FAQ / user-critical path trước. Cách này giảm chi phí khi beta gánh tải và đặc biệt có lợi nếu tất cả provider cùng chậm, nơi việc “failover theo kiểu cứng” không còn là giải pháp đủ.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | ❌ |
| cached (của bạn) | 0.9117 | 0.9645 | 0.568 | 189 | 0 | 2 | 0.971 | ✅ |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Chọn ở mức này để tránh gần nhưng khác, ví dụ “Chính sách hoàn tiền” và “Phí ship đi tỉnh” có thể có embedding tương đồng nhưng không phải cùng intent. Threshold thấp hơn sẽ tăng hit ratio nhưng có nguy cơ `wrong_answers` và `other_question`; threshold cao hơn trong khi tăng độ an toàn lại làm hit ratio giảm. |
| ttl_s | 25 | Duy trì TTL ngắn để câu trả lời không tồn tại quá lâu sau khi chính sách bất ngờ đổi. Khi C1 có thay đổi chính sách ở t=150 và prompt v2 ở t=420, TTL ngắn hạn giúp mới nhất được ưu tiên, đồng thời `outdated_answers = 2` thấp hơn nhiều so với không có TTL. |
| max_stale_s | 300 | Duy trì cửa sổ stale đủ dài để phục vụ khi mọi provider sập (t=240–360) mà không ép cache phải cập nhật ngay. Nó giúp `faq_availability` tăng từ 84.24% lên 96.45%, trong khi vẫn giữ `wrong_answers = 0` và `outdated_answers = 2`. |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Nếu `similarity_threshold` đặt 0.80, cache sẽ match quá rộng và có nguy cơ trả nhầm câu hỏi “gần như cùng chủ đề nhưng khác intent”. Ở mức 0.92, hệ thống vẫn giữ `cache_hit_ratio = 0.568` nhưng `wrong_answers = 0`; đó là một điểm cân bằng giữa độ an toàn và tỷ lệ hit. Mức quá thấp dễ tạo `wrong_answers_by_reason` kiểu `other_question` hoặc nhầm giữa các câu hỏi có cùng từ khóa nhưng khác dữ liệu người dùng. 

2. **TTL.** TTL quá dài làm `outdated_answers` tăng vì webhook chỉ invalidate khi nội dung cập nhật được phát hiện; nếu system đã cache một response cũ quá lâu, webhook không “lùi ngược” câu trả lời đó về trạng thái tươi. Do đó TTL là cơ chế duy nhất giới hạn stale trong một khoảng không đổi; trong C1, TTL 25s và max_stale 300s cho ra `outdated_answers = 2`, vẫn thấp nhưng đủ để tối ưu dự phòng outage. 

3. **Stale-if-error.** Nếu `max_stale_s = 0`, câu trả lời cũ không còn được phục vụ trong khi mọi provider sập, dẫn tới `faq_availability` chụm về mức của client không cache (0.8424). Chỉ khi stale-if-error còn mở, hệ thống có thể giữ được `faq_availability = 0.9645` trong thời gian outage. Nói cách khác, stale cache là “vùng an toàn” cho FAQ, không phải là “đáp án chính xác ở mọi thời điểm”.

Trả lời:

- Vì sao câu hỏi cá nhân (“Đơn #48213 của tôi đang ở đâu?”) không được cache, kể cả khi cache theo từng user? Vì đây là dữ liệu nhạy cảm và thuộc phạm vi cá nhân; cache theo user nhưng không yêu cầu `prompt_version` + allowlist sẽ dễ lộ dữ liệu giữa các user hoặc trả về response của người khác. Thêm vào đó, trong C1, mục tiêu là câu hỏi FAQ được cache, còn các câu hỏi có `personal=True` không nằm trong allowlist nên phải được xử lý qua provider hoặc qua tầng bảo mật riêng.
- Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố? Cần alert trên `provider_error_rate`, `stale_served` và `cache_hit_ratio` theo thời gian: nếu stale cache đang được serve quá lâu hoặc số bad provider > 0 liên tục trong khi `cache_hit_ratio` tăng cao, đó là dấu hiệu sự cố provider thật sự. Một dashboard “green” vì cache không có nghĩa là hệ thống ổn định; on-call nên alert dựa trên “provider outage signal” chứ không chỉ trên user-facing availability. 

---

Hồ sơ này được viết dựa trên số liệu thực tế từ repo và thể hiện hướng tiếp cận blameless: hệ thống không “sai vì con người”, mà sai vì các cơ chế retry, breaker và cache chưa được phòng vệ đúng cách. Mục tiêu của bài lab là hiểu rõ này và đưa ra một client chịu lỗi theo kiểu production, không chỉ chạy từng test riêng lẻ.
