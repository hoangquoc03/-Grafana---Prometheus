# Bài 2: Tối ưu hóa chu kỳ Scrape

## Hiện trạng

Prometheus đọc cấu hình từ `prometheus/prometheus.yml`, được mount chỉ đọc vào `/etc/prometheus/prometheus.yml` trong container theo `compose.yaml`. Trước khi thực hiện bài này, file và cấu hình đang chạy mà `/api/v1/status/config` trả về đều đã có `scrape_interval: 15s` và `evaluation_interval: 15s`; không có bằng chứng hiện trạng này từng chạy ở `1s`. Vì vậy, không thể khẳng định ở workspace này đã đo hay tái hiện được sự thay đổi từ `1s`.

Cấu hình sau bài tập giữ nguyên job scrape Prometheus và bổ sung job minh họa `backend-app` theo yêu cầu:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - prometheus:9090

  - job_name: backend-app
    static_configs:
      - targets:
          - backend:8080
```

Không có service `backend` trong `compose.yaml`; target `backend:8080` được giữ để minh họa cấu hình và dự kiến sẽ DOWN/không phân giải được tên đến khi có backend thực tế trong cùng network. Điều này không phải lỗi cú pháp hay lỗi interval. Không tạo target hoặc metric giả.

## Ý nghĩa hai chu kỳ

- `scrape_interval: 15s` là khoảng cách giữa các lần Prometheus thu thập metric từ mỗi target; dữ liệu mới được cập nhật theo nhịp này.
- `evaluation_interval: 15s` là khoảng cách Prometheus đánh giá recording rule và alerting rule. Dự án hiện chưa khai báo rule; tùy chọn này vẫn đặt nhịp đánh giá toàn cục khi có rule.

Tăng interval từ `1s` lên `15s` làm giảm số lần scrape so với cấu hình `1s`, đổi lại dữ liệu quan sát ít cập nhật thường xuyên hơn. Mức giảm CPU/RAM phải đo và so sánh thực tế; không suy ra chỉ từ cấu hình.

## Áp dụng

Compose hiện mount cấu hình đúng vị trí, nhưng Prometheus không có `--web.enable-lifecycle` trong command. Do đó HTTP reload không được bật; thay vì giả định reload thành công, dùng restart service `prometheus`:

```powershell
docker compose config
docker compose restart prometheus
docker compose ps
docker compose logs --tail=100 prometheus
```

Nếu host port mặc định đã được dùng như trong môi trường kiểm tra này, đặt cổng host thay thế (không làm đổi cổng nội bộ service) trong PowerShell trước lệnh Compose:

```powershell
$env:PROMETHEUS_PORT='19090'
$env:GRAFANA_PORT='13000'
```

Không dùng `docker compose down -v`; thao tác này không cần thiết và có thể xóa named volume dữ liệu.

## Xác minh

1. Kiểm tra cấu hình Compose và mount:

   ```powershell
   docker compose config
   ```

2. Xem log Prometheus sau restart. Tìm thông báo nạp `/etc/prometheus/prometheus.yml`, hoàn thành tải cấu hình và server sẵn sàng; xử lý lỗi parse YAML nếu có.
3. Mở `/api/v1/status/config` trên cổng host đang map tới 9090 và xác nhận `scrape_interval` cùng `evaluation_interval` đều `15s`.
4. Mở `/targets`. Target `prometheus:9090` cần UP; `backend:8080` chỉ UP nếu backend hiện diện, hoạt động và expose metric tương thích.
5. Chạy `up` trong Grafana Explore, hoặc kiểm tra truy vấn qua datasource API. Metric `up` bằng `1` thể hiện target tương ứng được scrape thành công; `0`/thiếu series cần đối chiếu target và log.
6. Đối chiếu timestamp các mẫu `up` qua một khoảng thời gian dài hơn 15 giây để quan sát nhịp scrape. Không tuyên bố CPU/RAM giảm nếu chưa đo được trước/sau trong điều kiện tương đương.

## Kết quả kiểm tra thực tế

- Trước chỉnh sửa: `/api/v1/status/config` xác nhận cả hai interval đã là `15s`; target `prometheus:9090` UP và truy vấn `up` trả giá trị `1`. Vì vậy, không có thay đổi interval từ `1s` thực tế trong workspace để báo cáo như đã xảy ra.
- Sau chỉnh sửa: `prometheus/prometheus.yml` vẫn đặt hai interval `15s`, giữ job Prometheus và thêm `backend-app`. Compose không có service `backend`.
- `docker compose config --quiet` và `docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml` thành công. Cấu hình được mount tại `/etc/prometheus/prometheus.yml`.
- Command Prometheus không bật `--web.enable-lifecycle`; đã áp dụng bằng `docker compose restart prometheus` (không xóa volume). `docker compose ps` xác nhận Prometheus và Grafana ở trạng thái `Up`. Log sau restart có thông báo nạp đúng file cấu hình và `Server is ready to receive web requests`, không có lỗi parse.
- `/api/v1/status/config` sau restart trả `scrape_interval: 15s` và `evaluation_interval: 15s`. `/api/v1/targets` báo `prometheus:9090` **UP**, `backend:8080` **DOWN** với lỗi DNS `lookup backend ... no such host`; Compose không khai báo service backend.
- Truy vấn `up` qua API Prometheus và Grafana datasource proxy đều có dữ liệu: `prometheus:9090` trả `1`, `backend:8080` trả `0`. Đây xác nhận truy vấn Grafana tới Prometheus hoạt động, nhưng backend chưa được scrape thành công.
- Quan sát `timestamp(up)` qua khoảng 65 giây cho thấy các mẫu của target Prometheus cách nhau xấp xỉ 15–16,94 giây trong lần đo này. Đây là quan sát ngắn hạn, không phải benchmark tài nguyên.
- Mức tiêu thụ CPU/RAM và so sánh tần suất mẫu trước/sau chưa được đo; không thể kết luận tài nguyên giảm.
