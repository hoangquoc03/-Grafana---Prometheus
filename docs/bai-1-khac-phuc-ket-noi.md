# Bài 1: Khắc phục kết nối Grafana – Prometheus

## Tình huống và nguyên nhân

Một datasource được cấu hình với URL `http://192.168.1.15:9090` có thể mất kết nối sau khi Docker Compose tạo lại container. Địa chỉ IP container không phải định danh ổn định; container mới có thể nhận IP khác. Khi đó Grafana tiếp tục gửi truy vấn đến IP cũ.

Compose cung cấp DNS nội bộ cho các service cùng network. Hai service trong dự án này cùng tham gia network `monitoring`, vì vậy Grafana có thể dùng tên service `prometheus` làm hostname. URL `http://prometheus:9090` được phân giải nội bộ sang địa chỉ hiện tại của service và không phụ thuộc IP container qua lần khởi động lại.

Port `9090` được publish ra host để người dùng mở Prometheus tại `http://localhost:9090`; việc publish này không cần thiết cho giao tiếp nội bộ giữa Grafana và Prometheus. Grafana gọi Prometheus qua network Compose.

Các cổng host có thể cấu hình qua `PROMETHEUS_PORT` và `GRAFANA_PORT` (mặc định lần lượt là `9090` và `3000`). Nếu cổng mặc định đang được stack khác sử dụng, đổi chúng trong `.env`; cổng nội bộ Prometheus vẫn là `9090`.

## Các file cấu hình

- `compose.yaml`: khai báo hai service, network dùng chung, cổng host, mounts và named volume lưu dữ liệu.
- `prometheus/prometheus.yml`: scrape chính Prometheus mỗi 15 giây bằng target service `prometheus:9090`.
- `grafana/provisioning/datasources/datasource.yml`: tự provision datasource Prometheus mặc định vào thư mục provisioning Grafana `/etc/grafana/provisioning/datasources/`; URL là `http://prometheus:9090`, access là `proxy`.
- `.env.example`: mẫu biến môi trường tài khoản Grafana. Mặc định `admin/admin` chỉ dành cho học tập cục bộ; tạo `.env` riêng để đặt mật khẩu khác. `.env` đã được bỏ qua bởi Git.
- `README.md`: hướng dẫn chạy nhanh.

Compose mount file Prometheus vào `/etc/prometheus/prometheus.yml` ở chế độ chỉ đọc; file datasource được mount vào đúng thư mục provisioning Grafana, cũng chỉ đọc. Dữ liệu Grafana và Prometheus lưu riêng trong `grafana-data` và `prometheus-data`.

## Chạy và kiểm tra

Chạy tại thư mục chứa `compose.yaml`:

```powershell
docker compose config
docker compose up -d
docker compose ps
docker compose logs --tail=100 grafana
docker compose logs --tail=100 prometheus
```

Sau khi chỉnh provisioning, nạp lại bằng cách khởi động lại Grafana:

```powershell
docker compose restart grafana
docker compose logs --tail=100 grafana
```

### Kết quả kiểm tra trong workspace

- `docker compose config`: thành công, gồm cả xác nhận mount path và biến cổng.
- Lần `docker compose up -d` đầu tiên không thành công vì service khác trên máy đã chiếm host port `9090`; kiểm tra container cho thấy host port `3000` và `9091` cũng đang dùng. Không dừng hoặc thay đổi container ngoài dự án. Stack bài tập được chạy bằng host port `19090` cho Prometheus và `13000` cho Grafana; cổng nội bộ vẫn là `9090` và `3000`.
- `docker compose up -d` với các biến cổng trên: thành công. `docker compose ps`: cả `grafana` và `prometheus` ở trạng thái `Up`, cùng network `grafana-prometheus_monitoring`.
- Log Grafana ghi nhận `inserting datasource from configuration` với tên `Prometheus`; endpoint health Grafana trả HTTP 200. Log Prometheus cho biết đã nạp `/etc/prometheus/prometheus.yml` và server sẵn sàng. Không thấy lỗi DNS, `connection refused` hay timeout liên quan datasource.
- Prometheus readiness tại `http://localhost:19090/-/ready`: HTTP 200, `Prometheus Server is Ready.` Grafana health tại `http://localhost:13000/api/health`: HTTP 200.
- Grafana API trả datasource `Prometheus` với `url: http://prometheus:9090`, `access: proxy`, `isDefault: true`. Lệnh kiểm tra datasource health trả `status: OK`, `Successfully queried the Prometheus API.`
- Truy vấn `up` qua Grafana datasource proxy thành công; kết quả có `instance="prometheus:9090"`, `job="prometheus"`, giá trị `1`. Prometheus `/api/v1/targets` báo target `prometheus` có `health: up`, `lastError` rỗng.
- Workspace ban đầu trống nên không có cấu hình hardcode IP hoặc dashboard cũ để kiểm tra log trước/sau hay khẳng định dashboard hiển thị. Nguyên nhân IP tĩnh là tình huống mô phỏng; kiểm tra runtime xác nhận cấu hình DNS mới hoạt động. Chưa xác minh bằng nút **Save & test** trên giao diện hoặc dashboard do không có dashboard được cung cấp; API health/query là kiểm tra kết nối thực tế thay thế.

Để chạy lại ở môi trường hiện tại trong PowerShell mà không đổi `.env`, dùng:

```powershell
$env:PROMETHEUS_PORT='19090'
$env:GRAFANA_PORT='13000'
docker compose up -d
```

Sau đó mở <http://localhost:13000> và <http://localhost:19090/targets>. Để chạy lệnh `docker compose up -d` thông thường ở những lần sau, lưu các cổng rảnh này vào `.env` hoặc giải phóng cổng mặc định do ứng dụng khác đang sử dụng.

## Xác minh trong trình duyệt

1. Mở Grafana tại <http://localhost:3000> và đăng nhập bằng tài khoản cấu hình. Nếu chưa tạo `.env`, tài khoản học tập mặc định là `admin` / `admin`.
2. Vào **Connections → Data sources → Prometheus**. Xác nhận URL `http://prometheus:9090`, access proxy; nhấn **Save & test**. Kết quả thành công ở đây mới xác nhận Grafana truy cập datasource.
3. Vào **Explore**, chọn Prometheus, chạy truy vấn `up`. Với cấu hình hiện tại, target Prometheus tự scrape sẽ xuất hiện khi scrape thành công.
4. Mở <http://localhost:9090/targets>, xác nhận target `prometheus` ở trạng thái **UP**.
5. Mở lại dashboard. Nếu datasource và target hoạt động nhưng dashboard vẫn `No data`, kiểm tra truy vấn PromQL, metric thực sự có tồn tại và khoảng thời gian dashboard.

## Phân biệt lỗi

- **`no such host`**: DNS không phân giải được tên service; kiểm tra chính tả tên `prometheus`, hai service có cùng network và container đang tham gia network đó.
- **`connection refused`**: hostname có thể đã phân giải nhưng Prometheus chưa sẵn sàng hoặc không lắng nghe ở cổng 9090; xem log Prometheus, trạng thái container và cấu hình cổng.
- **Timeout / `context deadline exceeded`**: kiểm tra service có phản hồi không, network và firewall; xem log hai container.
- **Lỗi cấu hình/provisioning**: kiểm tra `docker compose config`, mount file đúng đường dẫn và YAML hợp lệ; xem log Grafana để biết datasource provisioning có lỗi hay không.
- **Datasource hoạt động nhưng dashboard `No data`**: không đồng nghĩa lỗi kết nối; kiểm tra `/targets`, chạy `up`, rồi xác minh metric, PromQL và khoảng thời gian dashboard.
