# Bài 4: Tự động hóa giám sát với `file_sd_configs`

## Vấn đề và cách hoạt động

`static_configs` lưu danh sách target ngay trong `prometheus.yml`. Khi danh sách backend thay đổi, phải sửa cấu hình Prometheus và reload/restart cấu hình chính. `file_sd_configs` tách danh sách target ra file JSON riêng để cập nhật danh sách target mà không cần reload cấu hình Prometheus.

Job `microservices` đọc `/etc/prometheus/targets.json`, là file được mount từ `prometheus/targets.json` trong workspace. `refresh_interval: 30s` nghĩa là Prometheus định kỳ kiểm tra file theo chu kỳ này; thay đổi không được đảm bảo phát hiện tức thì. Nếu thay đổi cấu hình chính trong `prometheus.yml`, vẫn cần reload hoặc restart theo cấu hình Prometheus hiện tại.

## Cấu hình Prometheus

Trước bài tập, workspace chưa có job `microservices` hoặc các target `10.0.0.51:8080` và `10.0.0.52:8080`. Các job `prometheus` và `backend-app`, `global` với chu kỳ 15 giây và `rule_files` từ Bài 3 được giữ nguyên. Job mới:

```yaml
- job_name: microservices
  file_sd_configs:
    - files:
        - /etc/prometheus/targets.json
      refresh_interval: 30s
```

Compose mount file vào đúng đường dẫn container `/etc/prometheus/targets.json`.

## Danh sách target

`prometheus/targets.json` có nội dung:

```json
[
  {
    "targets": [
      "10.0.0.51:8080",
      "10.0.0.52:8080"
    ],
    "labels": {
      "env": "production"
    }
  }
]
```

Hai IP trên là địa chỉ mẫu theo đề bài; chúng chỉ hoạt động nếu thực sự có thể truy cập từ network của Prometheus. Nhãn `env: production` được gán cho cả hai target trong nhóm này.

Để thêm target, thêm địa chỉ `"host:port"` vào mảng `targets`; để xóa target, bỏ địa chỉ đó khỏi mảng. Giữ JSON hợp lệ, lưu file và chờ tối đa một chu kỳ kiểm tra file (30 giây) trước khi xác nhận thay đổi trên `/targets`. Không cần restart chỉ vì nội dung JSON thay đổi khi file được mount đúng.

## Kiểm tra và áp dụng

```powershell
python -m json.tool prometheus/targets.json
docker compose config
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
docker compose exec -T prometheus promtool check rules /etc/prometheus/alert_rules.yml
docker compose restart prometheus
docker compose ps
docker compose logs --tail=100 prometheus
```

Prometheus hiện không bật lifecycle API, nên thay đổi `prometheus.yml` áp dụng bằng restart service. Không xóa named volume. Nếu cổng host mặc định đã bị chiếm, đặt `PROMETHEUS_PORT` và `GRAFANA_PORT` giống các giá trị đã dùng khi khởi chạy stack.

Mở `/targets` trên cổng host đã map tới Prometheus để kiểm tra job `microservices`, target và nhãn `env="production"`. Đọc log để phân biệt lỗi đọc/parse file với lỗi kết nối target: target DOWN không nhất thiết đồng nghĩa file service discovery hỏng.

## Kết quả kiểm tra thực tế

- Khảo sát ban đầu: không có job `microservices` và cũng không có hai IP mẫu trong `prometheus.yml`; do đó không có `static_configs` cũ để thay. Đã bổ sung job mới bằng `file_sd_configs`, không sửa job Prometheus/backend-app hoặc rule Bài 3.
- Python không có trong shell (`python` không được nhận diện). Thay thế bằng PowerShell `ConvertFrom-Json`: parse thành công, xác nhận đúng hai target và nhãn `env=production`. Prometheus `promtool check config /etc/prometheus/prometheus.yml` cũng đọc và kiểm tra thành công cấu hình, gồm file service discovery; `promtool check rules` xác nhận rule Bài 3 vẫn hợp lệ.
- `docker compose config` xác nhận mount `prometheus/targets.json` tại `/etc/prometheus/targets.json`. Đã chạy `docker compose up -d` để áp dụng job và mount; Prometheus/Grafana vẫn `Up`. Log xác nhận đọc cấu hình `/etc/prometheus/prometheus.yml`, server sẵn sàng; không thấy lỗi parse JSON/config.
- `/api/v1/status/config` cho thấy `microservices` đang dùng `/etc/prometheus/targets.json` với refresh `30s`. `/api/v1/targets` xác nhận hai IP được phát hiện, mỗi target có `env=production`.
- Hai IP mẫu hiện **DOWN** với `context deadline exceeded` sau timeout scrape 10 giây. Đây là lỗi truy cập/không phản hồi của target từ network Prometheus, không phải lỗi phân tích file discovery. Chưa xác minh rằng IP này thực sự tồn tại hoặc metric endpoint hoạt động.
- Kiểm tra cập nhật động: tạm thêm target kiểm tra có thể truy cập `prometheus:9090` vào file; sau chờ 35 giây (hơn chu kỳ refresh), `/api/v1/targets` cho thấy target mới dưới job `microservices`, mang nhãn `env=production` và trạng thái UP. Container không restart trong phép thử. Sau khi xóa target thử nghiệm khỏi JSON, chờ thêm 35 giây và xác nhận nó biến mất khỏi job; file cuối chỉ còn hai IP yêu cầu.
- Prometheus đang map ra host port `19090` (Grafana `13000`) vì cổng mặc định đang được ứng dụng khác sử dụng; xác minh runtime dùng `http://localhost:19090/targets`. Cấu hình trong container và cổng nội bộ không đổi.
