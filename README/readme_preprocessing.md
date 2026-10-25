# Dữ liệu sạch BI10 Vòng 2

Thư mục này chứa 5 bảng dữ liệu đã làm sạch từ 5 file CSV gốc của đề, dùng chung cho cả nhóm. Mọi phân tích tiếp theo (Financial Health, Engagement, phân khúc, dashboard) đọc từ đây để số liệu giữa các phần khớp nhau.

Code làm sạch nằm ở `BI10_R02_clean.ipynb` trong thư mục `BI10_Round2`. Muốn chạy lại thì cài `pip install duckdb pandas`, mở notebook trong thư mục `BI10_Round2` rồi chạy hết, mất khoảng 3–5 phút. Lý do của các bước làm sạch dựa trên kết quả trong notebook EDA `BI10_R02_EDA.ipynb`.

Ngày chốt dữ liệu là 04/06/2025, ngày cuối của CONTRACT, LEADS và PAST_BEHAVIOR.

## Các file

| File | Số dòng | Số cột |
|---|---|---|
| CUSTOMER_clean.csv | 1.000.528 | 9 |
| CONTRACT_clean.csv | 4.280.377 | 28 |
| LEADS_clean.csv | 2.104.642 | 9 |
| PAST_BEHAVIOR_clean.csv | 4.412.617 | 5 |
| CALL_clean.csv | 4.000.000 | 15 |
| PRODUCT_MAPPING.csv | 21 | 2 |
| PROVINCE_MAPPING.csv | 68 | 2 |

`CUSTOMER_clean.csv` và hai file quy đổi có tên tỉnh viết có dấu, nên được lưu dạng UTF-8 có dấu hiệu bảng mã ở đầu file để Excel hiển thị đúng dấu. pandas, DuckDB và Power BI vẫn đọc bình thường.

## CUSTOMER_clean.csv

- Bỏ `FLAG_CAR_OWNER`, `NAME_SALARY_FREQUENCY`, `CNT_PERSON_DEPENDENT`. Cột đầu có giá trị "X" ở 100% dòng, cột thứ hai là XNA ở 99%, cột thứ ba trống 98,5%. Với cột người phụ thuộc, nhóm đã cân nhắc gán bằng số con cho khách có con, nhưng vẫn còn quá nhiều khách thiếu dữ liệu nên quyết định bỏ.
- Thay `ADDRESS` bằng `PROVINCE`, chỉ giữ tỉnh. Dữ liệu gốc có 68 cách ghi tên tỉnh, đã quy về đúng 63 tỉnh thành theo danh mục trước khi sáp nhập năm 2025, vì địa chỉ trong dữ liệu dùng tên tỉnh cũ. Tên tỉnh viết có dấu, ví dụ "Hồ Chí Minh", "Hà Nội". Có 149.243 khách không có tỉnh: 149.242 khách vốn để trống địa chỉ và 1 khách có địa chỉ lỗi "#NAME?". Bảng đối chiếu từng cách ghi gốc với tỉnh nằm trong `PROVINCE_MAPPING.csv`; dòng "#NAME?" trong đó là giá trị lỗi có sẵn trong dữ liệu gốc nên không có tỉnh. Khách thiếu địa chỉ tập trung ở nhóm khách có mã từ 120 triệu trở lên (33,4% thiếu, so với 6,3% ở nhóm còn lại), và các dòng đầu file thuộc nhóm này, nên mở file ra sẽ thấy vài chục dòng đầu trống tỉnh dù cả file có 851.285 khách có tỉnh.
- Thêm `AGE_AT_LAST_APPLY`, là tuổi của khách tại lần nộp hồ sơ gần nhất. Tuổi được tính tại lúc giao dịch chứ không tính tại ngày chốt, vì điều cần biết là khách bao nhiêu tuổi khi vay.
- 32 khách có tuổi lúc nộp hồ sơ dưới 18 hoặc trên 80. Ví dụ có khách ghi sinh năm 1901 nộp hồ sơ năm 2024, có khách sinh năm 2023 nộp hồ sơ khi chưa đầy 1 tuổi. Đây là lỗi ngày sinh, nên `DATE_BIRTH` và `AGE_AT_LAST_APPLY` của 32 khách này được để trống và cột `FLAG_INVALID_BIRTH` bằng 1.
- Giữ nguyên `CNT_CHILDREN`, kể cả các giá trị trên 5 (cao nhất là 99), theo quyết định của nhóm.

## CONTRACT_clean.csv

- Khử trùng từ 6.234.093 dòng xuống 4.280.377 dòng, mỗi `SKP_CREDIT_CASE` còn đúng một dòng. Gần 2 triệu hợp đồng bị ghi hai lần: một dòng có học vấn, thu nhập, nghề nghiệp đầy đủ, dòng kia là XNA hoặc 0, còn mọi trường khác giống hệt nhau. Bản sạch giữ dòng có thông tin đầy đủ. Nếu không khử trùng, mọi phép đếm theo hợp đồng sẽ cao lên khoảng 1,5 lần.
- Thêm `IS_SIGNED`: 1 với các trạng thái Active, Finished, Paid off, Signed, Written off, 0 với các trạng thái còn lại, theo định nghĩa của đề. Không dùng `SIGN_CONTRACT_TIME` để xác định đã ký, vì hợp đồng nộp trước 2017 không có ngày ký.
- Thêm `AMT_CREDIT_ADJ`: với BNPL lấy `AMT_BILLING`, vì `AMT_CREDIT` của BNPL luôn bằng 0; với các sản phẩm khác giữ `AMT_CREDIT`.
- Thêm `PRODUCT_GROUP` theo bảng quy đổi mã sản phẩm ở cuối file này.
- Thêm `AGE_AT_APPLY`, là tuổi của khách tại ngày nộp chính hợp đồng đó. Một hợp đồng nộp năm 2010 của khách nay 45 tuổi là hợp đồng của một người khoảng 30 tuổi.
- Giữ nguyên thu nhập, kể cả các giá trị rất lớn (cao nhất 9 tỷ đồng), theo quyết định của nhóm.
- Lưu ý khi dùng: `TRANSAC` thực chất là một ngày, không phải mã kênh như mô tả của đề. `FIRST_DUE` đến `FOURTH_DUE` là cờ trễ hạn ở từng kỳ trả 1–4 và không cộng dồn, nên một hợp đồng có thể trễ kỳ 4 mà không trễ kỳ 1.

## LEADS_clean.csv

- Bỏ 39.249 dòng trùng hoàn toàn ở mọi cột, từ 2.143.891 xuống 2.104.642 dòng.
- `MAX_OFFER`: 10.076 lead có giá trị từ 1 đến 59, trong khi không có lead nào từ 60 đến 99.999. Các giá trị này được hiểu là đơn vị triệu đồng và nhân với 1.000.000, kể cả giá trị 1. Bằng chứng: các lead Deal ghi 15, 6, 20, 30 dẫn đến khoản vay đúng 15, 6, 20, 30 triệu. Riêng giá trị 1 có bằng chứng yếu hơn. Một lead có giá trị 0 được đổi thành trống.
- Giữ nguyên `DATE_ACCEPTED`, kể cả 578.772 lead có giá trị 3000-01-01. Đây là giá trị quy ước cho trường hợp không có lần bấm nhận lead. Khi tính thời gian từ lúc tạo đến lúc nhận lead, cần tự loại các dòng này.
- Thêm `PRODUCT_GROUP`.

## PAST_BEHAVIOR_clean.csv

- 7 dòng gộp hai kênh trong một ô, ví dụ "SA, TLS SCORING", được tách thành 14 dòng, mỗi kênh một dòng ("SA SCORING" và "TLS SCORING"). Số dòng tăng từ 4.412.610 lên 4.412.617.
- Thêm `PRODUCT_GROUP`.

## CALL_clean.csv

- Không bỏ dòng nào. Có 12.527 `CALL_ID` xuất hiện hai lần, nhưng là các cuộc gọi khác ngày và khác nội dung, nên mỗi dòng vẫn là một cuộc gọi.
- Thêm `FLAG_AFTER_SNAPSHOT` bằng 1 cho 2.977.789 cuộc gọi (74,4%) diễn ra sau ngày chốt 04/06/2025. Nhóm chưa quyết định có dùng phần này hay không, nên dữ liệu được giữ lại và đánh dấu để mỗi người tự lọc.
- `PRODUCT_TYPE` có giá trị XNA được đổi thành trống.
- Thêm `PRODUCT_GROUP`.
- Giữ cả `RESPONSE_RESULT_2` và `CODE_DISPOSITION`. Hai cột giống nhau ở 99,85% dòng, nên chỉ cần dùng một cột khi phân tích.

## Quy đổi mã sản phẩm

Mỗi bảng dùng một bộ mã sản phẩm riêng. Cột `PRODUCT_GROUP` quy về 5 nhóm của đề:

- CD: CD
- TW: TW
- BNPL: BNPL, BNP
- CC: CC, CCX, VCC, SCW, CCW, CC_SC, CC_CC, CC_VCC
- CL: CL, CLX, ACL, ACLX, ACLD

Đề không có bảng đối chiếu, nên đây là giả định của nhóm dựa trên tên mã. Các mã CW, IN, EPOS, SAI chưa rõ nghĩa nên giữ nguyên mã gốc thay vì đoán nhóm. Bảng đầy đủ nằm trong `PRODUCT_MAPPING.csv`.

## Những gì cố ý chưa xử lý

- Thu nhập rất lớn và số con trên 5: giữ nguyên theo quyết định của nhóm.
- Khách có rất nhiều hồ sơ (có khách 1.794 hồ sơ) hoặc rất nhiều sự kiện chấm điểm (có khách 1.778 sự kiện): đây là dữ liệu thật, nên không xóa dòng. Khi đếm số lần theo từng khách, nhóm tự quyết định có giới hạn hay không.
- Cuộc gọi sau ngày chốt: giữ lại, có cờ đánh dấu.
