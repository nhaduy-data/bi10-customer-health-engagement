# Challenge 1 — Customer Financial Health Score
## Pipeline
**Mục tiêu:** Xây dựng Financial Health Score 0–100 minh bạch, phân khúc khách hàng, giải thích các yếu tố tích cực/tiêu cực và kiểm chứng tính hợp lý bằng dữ liệu.

### Bước 1 — Data Preparation & Quality Control
- Thực hiện EDA, làm sạch và chuẩn hóa dữ liệu `CUSTOMER`, `CONTRACT`.
- Đối soát hợp đồng trùng, dữ liệu thiếu và thông tin không nhất quán.
- Chuẩn hóa dữ liệu ở cấp **credit case** và **customer**, kèm cờ chất lượng.

### Bước 2 — Financial Statistical Analysis
- Phân tích hai trụ cột: **Debt Affordability** và **Payment Reliability**.
- Khảo sát phân phối, ngoại lệ và mối liên hệ giữa tỷ lệ trả góp/thu nhập với DUE.
- Kiểm tra liên hệ giữa DUE và Written off; phân tầng theo sản phẩm.
- Đánh giá tương quan, trùng lặp thông tin và giá trị bổ sung của đặc trưng.
- Phân biệt **mối liên hệ thống kê** với **quan hệ nhân quả/khả năng dự báo**.

### Bước 3 — Feature Engineering & Selection
- Tổng hợp dữ liệu thành **Customer Feature Mart**, một dòng cho mỗi khách hàng.
- Chọn bốn tín hiệu chính: **A1 — ODSR Proxy**, **A2 — Residual Income**, **P1 — Observed DUE Rate**, **P3 — Written-off History**.
- Giữ **A3 — Active Case Count** và **P2 — DUE Breadth** làm yếu tố hỗ trợ diễn giải.
- Xác định điều kiện hợp lệ, mức độ bao phủ và cảnh báo chất lượng theo từng trụ cột.

### Bước 4 — Explainable Financial Health Scoring
- Chuẩn hóa A1, A2 và P1 về thang 0–100 bằng các hàm điểm đơn điệu.
- Tính **Capacity = 80% × A1 + 20% × A2**.
- Tính **Payment** từ tỷ lệ DUE được làm mượt theo **Empirical Bayes**.
- Kết hợp **FHS cơ sở = 50% × Capacity + 50% × Payment**.
- Áp dụng **Written-off cap = 50** khi ghi nhận sự kiện Written off.
- Bù trụ cột thiếu bằng **Group Median Imputation** với cơ chế fallback; lưu `Evidence Flag`.
- Không tự gán FHS cho khách hàng thiếu cả hai trụ cột.

### Bước 5 — Validation & Parameter Calibration
- Dùng **Masked Holdout Validation** so sánh Reweighting, Global Median và Group Median.
- Hiệu chuẩn tham số làm mượt Payment theo **Beta-Binomial** (`k ≈ 6,35`).
- Kiểm tra độ nhạy trọng số, ngưỡng ODSR, tham số smoothing và Written-off cap.
- Chọn cấu hình có cơ sở tài chính, ổn định, dễ giải thích; lưu tham số vào JSON.
- Đối chiếu với **Historical Approval Rate** theo mức điểm/sản phẩm, không dùng tỷ lệ dư nợ.
- Công bố giới hạn: Approval Rate không đơn điệu; đây không phải mô hình dự báo vỡ nợ đã xác thực.

### Bước 6 — Explainability & Financial Segmentation
- Phân rã tác động của A1, A2, P1 và Written-off cap đối với điểm từng khách hàng.
- Xác định **Top Positive/Negative Factors** và tạo **Business Reason Codes**.
- Phân khúc: **Financial Stress, Watchlist, Moderate, Strong, Insufficient Evidence**.
- Giữ riêng cảnh báo Written off và mức độ đầy đủ bằng chứng cho hành động kinh doanh.

### Bước 7 — Final Outputs & Business Insights
- Xuất CSV cho **1.000.528 khách hàng**; **762.665 khách hàng có FHS (76,23%)**.
- Bàn giao điểm thành phần, điểm tổng, phân khúc, cờ bằng chứng và lý do chấm điểm.
- Lưu **selected scoring configuration**, bảng kiểm chứng và biểu đồ minh họa.
- Tổng hợp Business Insights và dữ liệu đầu vào cho **Challenge 3 — Next Best Action**.

> **Phạm vi diễn giải:** FHS là điểm đánh giá theo các tín hiệu tài chính quan sát được và quy tắc minh bạch; không phải xác suất vỡ nợ. ODSR và lịch sử DUE còn phụ thuộc giới hạn định nghĩa, kỳ dữ liệu và mức độ quan sát.

## Cở sở và ý tưởng
**Mục tiêu:** Chấm điểm tài chính khách hàng 0–100, phân khúc, giải thích yếu tố tích cực/bất lợi và kiểm chứng phương pháp. **Phiên bản:** `BI10_FHS_V2.1_CALIBRATED`. **Đơn vị:** `SKP_CLIENT` — một dòng/khách hàng.

> FHS là **chỉ số sức khỏe tài chính nội bộ có điều kiện (provisional proxy)**, không phải FICO, xác suất vỡ nợ (PD), DTI được xác minh hay quyết định cấp tín dụng tự động.

### 1. Dữ liệu và lý do chọn cách tiếp cận

Dữ liệu có **6.234.093 dòng hợp đồng**, nhưng chỉ **4.280.377 mã `SKP_CREDIT_CASE` duy nhất**, thuộc **1.000.528 khách hàng**. Vì vậy, toàn bộ bước tính toán hợp đồng phải xử lý trùng lặp ở cấp *case* trước khi tổng hợp lên khách hàng.

Kiểm toán thu nhập cấp case: **54,22%** có giá trị dương nhất quán; **45,52%** kết hợp 0/thiếu với một giá trị dương duy nhất; **0,26%** không có giá trị dương. Hệ thống đối chiếu trước/sau cleaning và giữ cảnh báo chất lượng dữ liệu, không tự xem thu nhập là đã xác minh.

Bộ dữ liệu **không đủ** chi phí sinh hoạt, nợ ngoài tổ chức, dư nợ hiện tại, tiền tiết kiệm và dòng tiền. Vì vậy nhóm xây **scorecard hai trụ cột**, không huấn luyện mô hình PD từ một nhãn kết cục chưa có thời điểm độc lập:

- **Capacity — khả năng đáp ứng nghĩa vụ:** A1 (gánh nặng trả góp) và A2 (thu nhập còn lại).
- **Payment — hành vi thực hiện nghĩa vụ:** P1 (tỷ lệ case có DUE) và P3 (sự kiện Written off).

A3 (số hợp đồng Active) và P2 (độ rộng cờ DUE) chỉ là **diagnostics**: A3 trùng lặp mạnh với A1; P2 có giá trị bổ sung không ổn định qua chỉ số kiểm thử.

### 2. Công thức tính và ý nghĩa tài chính

| Thành phần | Dữ liệu sử dụng | Công thức / tác động |
|---|---|---|
| **A1 — Debt-service proxy** | `ANNUITY` của case Active, `income_candidate` từ `AMT_INCOME_MAIN` | `ODSR = Σ Active ANNUITY / Income`; điểm A1 = `clip(100 × (0,50 − ODSR) / 0,40; 0; 100)` |
| **A2 — Residual-income proxy** | `income_candidate`, `active_annuity_known` | `RI = Income − Σ Active ANNUITY`; điểm A2 = `clip(100 × (RI − P10) / (P90 − P10); 0; 100)` |
| **P1 — Payment reliability** | `n_due_resolved_proxy = d`, `n_payment_observable_resolved_proxy = n` | `p̂ = (d + k × p₀)/(n + k)`; điểm P1 = `100/(1 + p̂/p₀)` |
| **P3 — Written-off event** | `n_written_off_cases`, trạng thái hợp đồng | Có Written off → giới hạn FHS tối đa **50**, thay vì phạt trùng hai lần |

**Tham số cuối:** `P10(RI) = 6.220.000`, `P90(RI) = 17.095.000` (đơn vị dữ liệu); `p₀ = 39.619 / 2.037.903 ≈ 1,9441%`; `k = 6,35079`, ước lượng bằng *Beta–Binomial marginal likelihood*. Ngưỡng ODSR tốt 10%, căng thẳng 50% là **quy tắc thiết kế**, không phải chuẩn pháp lý hay ngưỡng tối ưu dự báo.

```text
Capacity = 0,80 × A1 + 0,20 × A2
FHS_base = 0,50 × Capacity + 0,50 × Payment
         = 0,40 × A1 + 0,10 × A2 + 0,50 × P1
FHS_final = min(FHS_base, 50) nếu có Written off; ngược lại = FHS_base
```

Trọng số **50/50** và **80/20** được giữ như một lựa chọn quản trị cân đối, minh bạch, đã kiểm tra độ nhạy qua **9 cấu hình**; **không** được mô tả là hệ số dự báo tối ưu. Với khách hàng không có Active case, ODSR nội bộ có thể bằng 0, nhưng không được kết luận họ không có mọi khoản nợ.

### 3. Xử lý thiếu dữ liệu và khả năng bao phủ

Nếu đủ cả hai trụ cột → chấm trực tiếp (`FULL`). Nếu chỉ có một trụ cột → **Group Median Imputation** theo Product–Cohort, fallback Product rồi Global Median; giữ nguyên trọng số 50/50 và gắn cờ `CAPACITY_IMPUTED` / `PAYMENT_IMPUTED`. Thiếu cả hai → `NO_EVIDENCE`, **không tự tạo FHS**.

| Evidence | Khách hàng | % danh mục |
|---|---:|---:|
| FULL | 471.605 | 47,14% |
| CAPACITY_IMPUTED | 171.969 | 17,19% |
| PAYMENT_IMPUTED | 119.091 | 11,90% |
| NO_EVIDENCE | 237.863 | 23,77% |
| **Có FHS** | **762.665** | **76,23%** |

So với V1 (**25,82%**), coverage tăng **50,41 điểm phần trăm**, nhờ mở rộng eligibility và imputation — **không phải do thu thập thêm dữ liệu thực**.

### 4. Positive / Negative Financial Health Drivers

Các driver là **đóng góp theo công thức scorecard**, không phải tác động nhân quả. Đối với khách hàng `FULL`, nếu chưa bị Written-off cap và các yếu tố khác giữ nguyên, tăng 10 điểm A1 / A2 / P1 làm FHS_base tăng tương ứng **4 / 1 / 5 điểm**.

| Yếu tố | Tích cực khi | Bất lợi khi |
|---|---|---|
| **A1 (40%)** | Nghĩa vụ trả góp nội bộ thấp so với thu nhập | Gánh nặng trả góp quan sát cao |
| **A2 (10%)** | Thu nhập còn lại sau trả góp cao theo phân phối nội bộ | Thu nhập còn lại thấp |
| **P1 (50%)** | Ít/không ghi nhận DUE trên số case quan sát đủ | Có DUE trên tỷ lệ case cao |
| **P3 (cap)** | Không quan sát Written off → không bị cap | Có Written off → FHS ≤ 50 |

Bằng chứng thống kê cấp case: `ANY_DUE` tăng từ **1,85%** ở nhóm tỷ lệ trả góp/thu nhập dưới 10% lên **4,05%** ở 50–100%. `Written off` xuất hiện ở **13,69%** case có ANY_DUE so với **0,24%** case không có DUE (**RR ≈ 57,41**). Cả hai là **liên hệ đồng thời**, có thể khác nhau theo sản phẩm/cohort và **không chứng minh rủi ro tương lai**.

Trong file khách hàng cuối, `A1_SCORE_IMPACT`, `A2_SCORE_IMPACT`, `P1_SCORE_IMPACT`, `P3_CAP_IMPACT`, `TOP_POSITIVE_FACTOR`, `TOP_NEGATIVE_FACTOR` và `POSITIVE_EVIDENCE_CODES` / `NEGATIVE_EVIDENCE_CODES` phục vụ giải thích đến từng người. Cột *score impact* là chênh lệch so với subscore trung tính 50; nhóm IMPUTED có đóng góp bù riêng, không được nhầm với tín hiệu quan sát.

### 5. Phân khúc, validation và giới hạn

| Band | FHS | Khách hàng có điểm | Ý nghĩa hành động |
|---|---:|---:|---|
| Financial Stress | <40 | 14.957 | Ưu tiên rà soát nghĩa vụ |
| Watchlist | 40–<60 | 107.231 | Theo dõi/chăm sóc chủ động |
| Moderate | 60–<80 | 612.848 | Kết hợp Engagement & hồ sơ sản phẩm để cá nhân hóa |
| Strong | ≥80 | 27.629 | Xem xét cơ hội, vẫn phải thẩm định |
| Insufficient Evidence | Không có FHS | 237.863 | Bổ sung dữ liệu |

**Validation:** Trên **94.321** khách hàng FULL giả lập mất một trụ cột, MAE **Group Median / Reweighting** lần lượt là **5,232 / 14,820** khi che Capacity và **2,911 / 14,301** khi che Payment. Kiểm thử độ nhạy gồm trọng số, ODSR, smoothing và Written-off cap. **13.319** khách hàng có Written off, trong đó **8.137** thực sự bị cap làm giảm điểm.

**Approval Rate alignment** được kiểm tra ở cấp case trên **3.414.256** hồ sơ có kết quả được phân loại và ghép được với FHS. Theo các band từ thấp đến cao, tỷ lệ duyệt lần lượt **83,06% / 88,19% / 77,17% / 82,24% / 84,08%** — **không đơn điệu**. FHS được tính từ lịch sử có thể bao gồm sự kiện *sau* ngày duyệt, vì vậy đây là kiểm chứng mô tả nghiệp vụ, **không phải backtest dự báo**. Không sử dụng AMT_CREDIT để suy ra dư nợ hiện tại.

**Giới hạn bắt buộc công bố:** kỳ `AMT_INCOME_MAIN` chưa xác minh là tháng; kỳ quan sát DUE chưa được xác định đầy đủ; không biết toàn bộ nợ bên ngoài; mẫu FULL/IMPUTED có thể khác nhau hệ thống (*MNAR*). Score không dùng để tự động phê duyệt/từ chối.

### 6. Bàn giao và tái lập

- `C1_CHALLENGE1_FINAL/customer_financial_health_FINAL.csv`: 1.000.528 khách hàng và các cột điểm, drivers, warning, segment, guardrail.
- `C1_CHALLENGE1_FINAL/customer_financial_health_BUSINESS.csv`: Bản gọn dùng dashboard và Challenge 3.
- `C1_CHALLENGE1_FINAL/selected_scoring_config.json`, `selected_weights.json`: Cấu hình chấm và nguyên tắc chọn.
- `C1_CHALLENGE1_FINAL/validation_tables/`: Coverage, masked validation, weight robustness, cap, approval.
- `BI10_C1_8_Evidence_Visualizations.ipynb`: Tạo 8 hình PNG cho thuyết trình từ chính các file trên, cùng 2 CSV EDA đi kèm.

**Quy trình:** chạy cleaning → case reconciliation & Feature Mart → Final Calibration → Final Export → Eight Visualizations. Luôn dùng các file từ **cùng một lần chạy FINAL**, không pha trộn tham số V2 trước hiệu chuẩn với dataset V2.1.

## Insight
**Phạm vi:** 1.000.528 khách hàng; FHS V2.1 là chỉ số tài chính nội bộ 0–100, không phải PD hay công cụ phê duyệt tự động. Mỗi insight gắn với **bằng chứng → ý nghĩa → hành động**.

### Executive takeaway

**76,23% khách hàng có Financial Health Score**, nhưng chỉ **47,14%** có đủ hai trụ cột được quan sát; **29,09%** tổng danh mục cần bù một trụ cột; **23,77%** không đủ bằng chứng để chấm. Do đó, *độ khỏe tài chính* và *độ tin cậy của điểm* phải được sử dụng cùng nhau trong quyết định kinh doanh.

### 1. Độ phủ tăng, nhưng không được đồng nhất điểm ước lượng với điểm quan sát

**Bằng chứng.** V1 có **258.346** khách hàng có điểm (25,82%); V2.1 có **762.665** (76,23%), tăng **504.319** khách hàng. Trong đó có **471.605 FULL**, **171.969 CAPACITY_IMPUTED** và **119.091 PAYMENT_IMPUTED**. Masked validation trên **94.321** khách hàng cho thấy Group Median giảm MAE so với reweighting từ **14,82 xuống 5,23** (thiếu Capacity) và từ **14,30 xuống 2,91** (thiếu Payment).

**Ý nghĩa.** Imputation mở rộng phạm vi phân tích nhưng không làm xuất hiện bằng chứng tài chính mới. Điểm IMPUTED không tương đương độ tin cậy của FULL.

**Hành động.** Hiển thị FHS **cùng** `SCORE_EVIDENCE_FLAG`; ưu tiên xác minh dữ liệu trước hoạt động có rủi ro cao. Giữ **237.863** khách hàng `NO_EVIDENCE` cho chiến dịch bổ sung thông tin, không mặc định là khách hàng rủi ro.

### 2. Hai chiều tạo khác biệt lớn nhất: gánh nặng nghĩa vụ và lịch sử thanh toán

**Bằng chứng.** Điểm nền có trọng số hiệu dụng: **A1 40%**, **A2 10%**, **P1 50%**; Written-off là cap riêng. Thống kê cấp hợp đồng cho thấy `ANY_DUE` từ **1,85%** (<10% nghĩa vụ/thu nhập) lên **4,05%** (50–100%). `Written off` đồng thời xuất hiện ở **13,69%** case có ANY_DUE, so với **0,24%** case không có DUE.

**Ý nghĩa.** Nghĩa vụ cao và cờ DUE là **tín hiệu theo dõi hợp lý**, nhưng tương quan không đủ để kết luận nhân quả/dự báo. A2 giúp bổ sung bối cảnh thu nhập còn lại, không đủ bằng chứng làm yếu tố dự báo độc lập mạnh. Phân tích driver là phân rã **điểm**, không phải ước lượng tác động nhân quả.

**Hành động.** Trên dashboard dùng `TOP_POSITIVE_FACTOR`, `TOP_NEGATIVE_FACTOR`, các `*_SCORE_IMPACT` và `POSITIVE/NEGATIVE_EVIDENCE_CODES` để giải thích theo khách hàng. Khi thuyết trình, ưu tiên biểu đồ Driver Analysis trên nhóm **FULL** để tránh giả định bù dữ liệu làm méo phân tích.

### 3. “Moderate” chiếm phần lớn danh mục: cần lớp phân khúc thứ hai

**Bằng chứng.** Trên **762.665** khách hàng có điểm: **14.957 Financial Stress (1,96%)**, **107.231 Watchlist (14,06%)**, **612.848 Moderate (80,36%)** và **27.629 Strong (3,62%)**.

**Ý nghĩa.** Phân khúc Moderate quá lớn để áp dụng một chiến lược chăm sóc hoặc chào bán chung. Phân phối cũng chịu ảnh hưởng chuẩn hóa điểm, smoothing và median imputation; không khẳng định mọi người trong band có cùng hồ sơ tài chính.

**Hành động.** Kết hợp **Financial Segment × Evidence Flag × Engagement Score × Product Context** trong Challenge 3. Không thay đổi ngưỡng chỉ để làm các nhóm có kích thước bằng nhau.

### 4. Written-off là cảnh báo ưu tiên, không chỉ là một điểm trừ

**Bằng chứng.** Có **13.319** khách hàng ghi nhận ít nhất một case `Written off`; tất cả được gắn cờ trong danh mục chấm điểm. Cap 50 làm giảm điểm thật ở **8.137** khách hàng; nhóm còn lại đã có điểm không quá 50.

**Ý nghĩa.** FHS tổng hợp có thể che khuất sự kiện nghiêm trọng nếu chỉ sử dụng một ngưỡng điểm. Cap là **quy tắc quản trị thận trọng**, không phải tỷ lệ default hoặc chuẩn phân loại nợ.

**Hành động.** `SEVERE_CREDIT_EVENT` là guardrail ưu tiên: rà soát trạng thái nghĩa vụ, hỗ trợ xử lý hồ sơ, xác minh trước tiếp thị tín dụng mới; **không dùng riêng FHS để tự động từ chối**.

### 5. Approval Rate lịch sử không chứng minh độ chính xác dự báo

**Bằng chứng.** Trên **3.414.256** case được phân loại quyết định và ghép với FHS, Approval Rate theo các band điểm từ thấp đến cao lần lượt là **83,06% / 88,19% / 77,17% / 82,24% / 84,08%** — **không đơn điệu**. Mối liên hệ còn thay đổi theo sản phẩm/cohort.

**Ý nghĩa.** FHS hiện tại có thể dùng thông tin DUE/Written off **sau** quyết định phê duyệt trong quá khứ. Đây là đối chiếu nghiệp vụ mô tả, không phải kiểm định dự báo tại thời điểm nộp đơn. Không thể tính tỷ lệ dư nợ hiện tại từ `AMT_CREDIT`.

**Hành động.** Công bố trung thực sự không đơn điệu, không tối ưu trọng số để ép Approval Rate tăng theo FHS. Nếu triển khai production, cần dữ liệu có mốc thời gian và outcome độc lập sau ngày chấm.

---

### Quy tắc sử dụng FHS trong Next Best Action

| Thứ tự ưu tiên | Điều kiện | Hành động gợi ý |
|---|---|---|
| 1 | `SEVERE_CREDIT_EVENT` | Rà soát tín dụng; kiểm soát hành động rủi ro cao |
| 2 | `NO_EVIDENCE` | Thu thập thu nhập, nghĩa vụ, lịch sử thanh toán |
| 3 | `*_IMPUTED` | Xác minh trụ cột đã bù trước đề xuất quan trọng |
| 4 | `FULL` + Stress/Watchlist | Tư vấn quản lý nghĩa vụ, chăm sóc chủ động |
| 5 | `FULL` + Moderate/Strong | Kết hợp Engagement và điều kiện sản phẩm; vẫn qua quy trình thẩm định |

**Thông điệp cho ban giám khảo:** *Một thước đo tài chính có thể kiểm toán đáng tin cậy hơn một điểm số đầy đủ giả tạo.* Hệ thống đã mở rộng khả năng phân khúc, chỉ ra drivers của điểm và chủ động kiểm soát bất định; nhưng không biến các proxy quan sát thành khẳng định về nợ xấu hay xác suất trả nợ tương lai.

**Nguồn:** `C1_CHALLENGE1_FINAL` (customer-level scores, selected configuration và validation tables); statistical EDA cấp case; tài liệu phương pháp `README_Challenge1_FHS_CONCISE.md`. Hình 01–08 được tái lập bằng `BI10_C1_8_Evidence_Visualizations.ipynb`.