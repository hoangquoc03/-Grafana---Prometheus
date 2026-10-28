# Grafana – Prometheus

Stack học tập tối thiểu dùng Docker Compose. Grafana kết nối Prometheus bằng DNS nội bộ của Compose (`http://prometheus:9090`), không phụ thuộc IP container.

## Chạy nhanh

Yêu cầu Docker Desktop hoặc Docker Engine có Docker Compose v2.

```powershell
docker compose config
docker compose up -d
docker compose ps
```

Mở:

- Grafana: <http://localhost:3000> — mặc định môi trường học tập: `admin` / `admin`.
- Prometheus: <http://localhost:9090> — Targets: <http://localhost:9090/targets>.

Trong Grafana, vào **Connections → Data sources → Prometheus**, xác nhận URL `http://prometheus:9090`, rồi nhấn **Save & test**. Mở **Explore**, chọn Prometheus và chạy `up`. Prometheus tự scrape chính nó; không cấu hình target giả.

## Mật khẩu và cổng

Giá trị mặc định `admin/admin` chỉ dành cho máy học tập cục bộ. Để thay đổi, sao chép `.env.example` thành `.env`, đặt `GRAFANA_ADMIN_PASSWORD` thành mật khẩu riêng và chạy lại `docker compose up -d`. Không commit `.env`; không sử dụng giá trị mặc định trên máy chủ hoặc môi trường có thể truy cập từ mạng.

Cổng host mặc định là Grafana `3000` và Prometheus `9090`. Nếu các cổng đã được ứng dụng khác sử dụng, đổi `GRAFANA_PORT` và `PROMETHEUS_PORT` trong `.env` (ví dụ `13000` và `19090`), rồi chạy `docker compose up -d`. URL datasource nội bộ vẫn là `http://prometheus:9090`. Khi dùng các cổng ví dụ này, mở Grafana ở <http://localhost:13000> và Prometheus ở <http://localhost:19090>.

## Vận hành

```powershell
docker compose logs --tail=100 grafana
docker compose logs --tail=100 prometheus
docker compose restart grafana
```

Dữ liệu được lưu trong hai named volume `grafana-data` và `prometheus-data`, tồn tại qua việc tạo lại container. Không dùng `docker compose down -v` nếu muốn giữ dữ liệu.

## Bài tập

- [Bài 1: Khắc phục kết nối Grafana – Prometheus](docs/bai-1-khac-phuc-ket-noi.md).
- [Bài 2: Tối ưu hóa chu kỳ scrape](docs/bai-2-toi-uu-scrape-interval.md). Chu kỳ scrape và evaluation được đặt `15s`; job `backend-app` chỉ hoạt động khi có service `backend` lắng nghe ở cổng `8080`.
