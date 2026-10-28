# Bài 3: Xử lý cảnh báo giả (False Alarm)

## Vấn đề

Ngưỡng cảnh báo CPU `> 20%` kết hợp với `for: 10s` có thể kích hoạt cảnh báo khi tải tăng nhẹ hoặc spike ngắn. Alert nhạy quá mức làm phát sinh thông báo không hữu ích, gây mệt mỏi cảnh báo và che khuất sự cố đáng chú ý hơn.

Trong workspace này trước khi bài tập bắt đầu chưa có `alert_rules.yml` hay rule `HighCPUUsage` trong Prometheus. Vì vậy không thể xác nhận rule cũ `20%`/`10s` đã thực sự được nạp/chạy. Bài tập tạo rule với ngưỡng và thời gian duy trì như yêu cầu.

## Rule sau khi chỉnh sửa

Prometheus tải rule từ `prometheus/alert_rules.yml`, được mount chỉ đọc tại `/etc/prometheus/alert_rules.yml`. Đường dẫn `rule_files` trong `prometheus/prometheus.yml` trỏ đến đúng vị trí trong container.

```yaml
groups:
  - name: CPU_Alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage trên 85% liên tục trong 5 phút trên instance {{ $labels.instance }}."
```

Biểu thức tính phần trăm CPU bận từ tốc độ tăng của metric CPU idle, rồi so sánh với ngưỡng `> 85`; chỉ nhân `100` một lần. `for: 5m` yêu cầu biểu thức đúng liên tục năm phút trước khi alert chuyển từ Pending sang Firing. `labels` gắn metadata `severity: warning`. `annotations` cung cấp nội dung hiển thị cho người vận hành, gồm instance bị ảnh hưởng.

Ngưỡng `85%` cùng `for: 5m` giảm cảnh báo do spike ngắn so với ngưỡng thấp và thời gian chờ ngắn; không bảo đảm mọi cảnh báo giả biến mất, và có thể làm chậm phát hiện mức tải cao ngắn hạn.

## Kiểm tra cú pháp và áp dụng

Từ thư mục dự án:

```powershell
docker compose config
docker compose exec -T prometheus promtool check rules /etc/prometheus/alert_rules.yml
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
```

Nếu container chưa chạy nhưng image đã tải, có thể kiểm tra rule bằng container tạm và mount chỉ đọc:

```powershell
docker run --rm -v "${PWD}/prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro" prom/prometheus:v2.53.4 promtool check rules /etc/prometheus/alert_rules.yml
```

Trong cấu hình Compose hiện tại Prometheus không bật `--web.enable-lifecycle`, do đó không dựa vào HTTP reload; restart đúng service:

```powershell
docker compose restart prometheus
docker compose ps
docker compose logs --tail=100 prometheus
```

Không xóa named volume. Nếu host ports mặc định bị ứng dụng khác sử dụng, đặt các biến `PROMETHEUS_PORT` và `GRAFANA_PORT` giống các giá trị đã dùng khi khởi chạy stack.

## Xác minh trạng thái

- Mở `/rules` trên cổng host Prometheus và tìm nhóm `CPU_Alerts`, rule `HighCPUUsage`.
- Mở `/alerts` để xem trạng thái hiện tại:
  - **Inactive**: điều kiện alert chưa đúng.
  - **Pending**: điều kiện đúng nhưng chưa đủ `for: 5m`.
  - **Firing**: điều kiện vẫn đúng liên tục đủ năm phút.
- Xem log Prometheus để xác nhận nạp cấu hình thành công và không có lỗi parse rule.
- Trong Grafana Explore, chạy `up` để xác nhận metric scrape. Rule CPU cụ thể cần target cung cấp `node_cpu_seconds_total`; target Prometheus tự scrape không nhất thiết cung cấp metric CPU theo tên này.
- Không tạo tải CPU giả để ép cảnh báo firing. Nếu chưa có target cung cấp metric CPU, rule có thể không có chuỗi dữ liệu để đánh giá; đó là thiếu dữ liệu metric, không phải lỗi cú pháp rule.

## Kết quả kiểm tra thực tế

- `docker compose config --quiet`: thành công. Compose mount `prometheus/alert_rules.yml` vào `/etc/prometheus/alert_rules.yml`; `rule_files` trỏ đúng đường dẫn đó.
- `docker compose exec -T prometheus promtool check rules /etc/prometheus/alert_rules.yml`: thành công, tìm thấy 1 rule. `promtool check config /etc/prometheus/prometheus.yml`: thành công, xác nhận một rule file và cấu hình hợp lệ.
- Vì cấu hình Compose không bật lifecycle API, đã áp dụng bằng `docker compose up -d` để cập nhật mount và tạo lại service Prometheus; không xóa named volume. `docker compose ps` cho thấy Prometheus và Grafana `Up`. Log xác nhận đã nạp cấu hình và server sẵn sàng, không có lỗi parse rule.
- API `/api/v1/rules` (trang `/rules` HTTP 200) liệt kê nhóm `CPU_Alerts`, rule `HighCPUUsage`, query dùng `> 85`, `duration: 300` (5 phút), `severity: warning`, và evaluation health `ok`.
- API `/api/v1/alerts` (trang `/alerts` HTTP 200) hiện không có alert đang chờ hoặc firing. Rule được đánh giá **Inactive**; đây không phải bằng chứng cảnh báo giả đã được loại bỏ mà chỉ cho thấy điều kiện hiện không có kết quả đúng.
- Truy vấn `node_cpu_seconds_total` hiện trả vector rỗng; Compose không có node exporter hoặc target cung cấp metric CPU này. Do thiếu series, chưa xác minh được ngưỡng CPU, Pending/Firing, cũng như trạng thái CPU thực tế. Không tạo tải hoặc metric giả.
- CPU/RAM trước và sau chưa đo; không kết luận tài nguyên giảm.
