# Báo cáo kiểm thử hiệu năng API DummyJSON bằng JMeter

## 1. Giới thiệu

Báo cáo này trình bày kết quả kiểm thử hiệu năng hai endpoint của DummyJSON (`https://dummyjson.com`) bằng Apache JMeter. Kịch bản đo thời gian phản hồi, tỷ lệ lỗi và thông lượng khi có nhiều người dùng ảo gửi request đồng thời.

## 2. Mục tiêu

- Đo thời gian phản hồi trung bình, trung vị và các phân vị 90%, 95%, 99%.
- Đo tỷ lệ lỗi và thông lượng (số request mỗi giây).
- Đánh giá từng request theo ngưỡng: mã phản hồi 200 và thời gian dưới 2000 ms.

## 3. Môi trường và cấu hình

| Hạng mục | Giá trị |
|---|---|
| API | DummyJSON (`https://dummyjson.com`) |
| Công cụ | Apache JMeter (phiên bản: 5.6.3) |
| Hệ điều hành | Windows 10 |
| Số người dùng ảo | 10 cho mỗi request |
| Ramp-up | 10 giây |
| Số vòng lặp | 5 (tổng 50 mẫu cho mỗi request) |
| Request | `GET /products?limit=10`, `POST /products/add` |
| Khẳng định | Mã phản hồi = 200, thời gian < 2000 ms |
| Ngày thực hiện | 16/10/2026 |

> Ghi chú: DummyJSON chỉ mô phỏng việc thêm dữ liệu. Các thay đổi không được lưu thật.

**Hình 1. Giao diện JMeter sau khi mở, cây Test Plan bên trái**
![Giao diện JMeter](images/jmeter-01-giao-dien.png)

## 4. Thiết kế kịch bản

**Hình 2. Cấu hình Thread Group: 10 người dùng, ramp-up 10 giây, 5 vòng lặp**
![Thread Group](images/jmeter-02-thread-group.png)

**Hình 3. Cấu hình HTTP Request cho GET /products**
![HTTP Request](images/jmeter-03-http-request.png)

**Hình 4. Khẳng định mã 200 (Response Assertion)**
![Assertion](images/jmeter-04-assertion.png)

**Hình 5. Nội dung body của request POST /products/add**
![POST body](images/jmeter-05-post-request.png)

Lệnh chạy kiểm thử:

```bash
jmeter -n -t DummyJSON_LoadTest.jmx -l ket_qua.jtl -e -o bao_cao_html
```

## 5. Kết quả chi tiết từng mẫu

**Hình 6. View Results Tree: các request thành công và thất bại**
![View Results Tree](images/jmeter-06-view-results-tree.png)

## 6. Kết quả tổng hợp

| Request | Số mẫu | Lỗi | Tỷ lệ lỗi | Trung bình (ms) | Trung vị (ms) | Min (ms) | Max (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Thông lượng (req/s) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| GET /products | 50 | 50 | 100% | 1113,0 | 312,5 | 288 | 8715 | 4655,8 | 7236,1 | 8715 | 4,25 |
| POST /products/add | 50 | 0 | 0% | 310,6 | 299 | 284 | 575 | 318,9 | 429,6 | 575 | 14,97 |
| **Tổng** | **100** | **50** | **50%** | **711,8** | **305** | **284** | **8715** | **573,5** | **4705,9** | **8706,0** | **8,30** |

**Hình 7. Summary Report**
![Summary Report](images/jmeter-07-summary-report.png)

**Hình 8. Aggregate Report với phân vị 90%, 95%, 99%**
![Aggregate Report](images/jmeter-08-aggregate-report.png)

## 7. Phân tích

### 7.1. POST /products/add: đạt

Tất cả 50 mẫu thành công (0% lỗi). Trung vị khoảng 300 ms, thời gian tối đa 575 ms, nằm dưới ngưỡng 2000 ms. Thời gian phản hồi ổn định trong điều kiện tải nhẹ.

### 7.2. GET /products: không đạt

Cả 50 mẫu đều lỗi (100%), dù trung vị chỉ khoảng 312 ms. Điều này cho thấy phần lớn request nhận được phản hồi nhanh nhưng không phải mã 200, nên không phải do chậm.

Đuôi thời gian rất dài (p95 khoảng 7,2 giây, tối đa 8,7 giây) cho thấy một số request bị trì hoãn trước khi phản hồi. **Nguyên nhân chưa được xác định.** Các giả thuyết cần kiểm chứng:
- API giới hạn tần suất truy cập và trả về 429.
- API trả về lỗi máy chủ 5xx khi tải tăng.
- Dịch vụ chặn hoặc hạn chế request từ máy thử nghiệm.

### 7.3. Tổng thể

Request ghi dữ liệu (POST) ổn định. Request đọc (GET) không đạt trong lần chạy này và cần kiểm tra thêm trước khi kết luận.

## 8. Xác định nguyên nhân lỗi GET

Chạy lại với số người dùng ảo thấp (ví dụ 1) và xem View Results Tree để thấy mã và nội dung phản hồi. Hoặc phân tích file `ket_qua.jtl`:

Windows (PowerShell):
```powershell
Import-Csv ket_qua.jtl | Where-Object { $_.success -eq 'false' } | Group-Object responseCode | Select-Object Count, Name
```

macOS/Linux:
```bash
awk -F, 'NR>1 && $8=="false" {print $4}' ket_qua.jtl | sort | uniq -c
```

## 9. Minh họa báo cáo HTML

**Hình 9. Dashboard báo cáo HTML (biểu đồ Response Times và Throughput)**
![Dashboard HTML](images/jmeter-09-dashboard-html.png)

**Hình 10. Kết quả chạy từ dòng lệnh**
![CLI output](images/jmeter-10-cli-output.png)

## 10. Khuyến nghị

- Xác định mã lỗi của GET /products trước khi kết luận. Nếu là 429, cần giảm số người dùng ảo hoặc thêm độ trễ giữa các request.
- Chạy lại kịch bản GET với cùng cấu hình sau khi xác định nguyên nhân để so sánh.
- Theo dõi p95 và p99 thay vì chỉ nhìn trung bình, vì trung bình che khuất các request chậm.

## 11. Kết luận

POST /products/add đạt yêu cầu về tỷ lệ lỗi và thời gian phản hồi trong điều kiện thử nghiệm. GET /products chưa đạt do 100% mẫu lỗi. Cần hoàn tất phân tích mã lỗi ở mục 8 trước khi đưa ra kết luận chính thức.
