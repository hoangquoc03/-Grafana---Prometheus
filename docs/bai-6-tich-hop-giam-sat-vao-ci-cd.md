# Bài 6: Tích hợp giám sát vào CI/CD

## Mục tiêu

Pipeline phải kiểm tra cấu hình Prometheus bằng `promtool check config` trước khi triển khai service. Nếu validation trả mã khác 0, Jenkins dừng pipeline và không chạy stage Deploy.

Trước bài tập, workspace chưa có `Jenkinsfile` hoặc `.gitlab-ci.yml`; vì vậy không thể xác nhận pipeline cũ thực sự deploy trực tiếp. Đã chọn Jenkins và tạo một Declarative Pipeline duy nhất.

## Thay đổi pipeline

`Jenkinsfile` định nghĩa thứ tự:

1. Checkout SCM do Jenkins Declarative Pipeline thực hiện mặc định.
2. **Validate Config** — chạy đúng lệnh `promtool check config prometheus/prometheus.yml`.
3. **Deploy** — chạy `docker compose up -d --no-deps prometheus`.

Validation stage dùng `set -eu` trên agent Unix; command lỗi hoặc tool/daemon không sẵn sàng làm bước `sh` thất bại. Declarative Pipeline không tiếp tục sang stage kế tiếp sau lỗi, nên Deploy không được gọi. Trên agent Windows, batch script cũng thoát mã lỗi khi validation thất bại. Không có `|| true` hoặc success fallback.

Nếu agent có `promtool`, pipeline dùng bản đó. Nếu chưa có, kiểm tra Docker CLI và quyền kết nối daemon (`docker info`), sau đó chạy `promtool` từ image đã pin đúng với Prometheus trong Compose (`prom/prometheus:v2.53.4`). Workspace mount chỉ đọc tại `/workspace` và lệnh kiểm tra file thật `prometheus/prometheus.yml`. Thiếu cả promtool lẫn Docker daemon làm validation thất bại, không triển khai.

## Đường dẫn cấu hình

Prometheus hiện tham chiếu `alert_rules.yml` và `targets.json` tương đối từ thư mục chứa `prometheus.yml`. Điều này giữ nguyên các file/mount trong Compose (cùng nằm dưới `/etc/prometheus`) và cho phép `promtool check config prometheus/prometheus.yml` trên agent kiểm tra các file phụ trợ trong workspace checkout. Không tạo bản cấu hình mẫu riêng.

Job và thiết lập Bài 1–5 vẫn được giữ: datasource Grafana, interval `15s`, rule `HighCPUUsage`, job `backend-app` tại `/actuator/prometheus`, cùng `microservices` dùng file service discovery.

## Kiểm tra và triển khai

Chạy tại gốc workspace:

```powershell
promtool check config prometheus/prometheus.yml
```

Nếu agent không cài `promtool`, có Docker daemon và image đã tải:

```powershell
docker run --rm -v "${PWD}:/workspace:ro" -w /workspace --entrypoint promtool prom/prometheus:v2.53.4 check config prometheus/prometheus.yml
```

Pipeline deploy chỉ chạy sau validation thành công:

```powershell
docker compose up -d --no-deps prometheus
```

Stage Deploy được định nghĩa cho lần chạy Jenkins được ủy quyền; trong bài tập này không chạy Jenkins pipeline hoặc deploy/restart service chỉ để chứng minh test.

## Kiểm thử lỗi

Để thử fail gate an toàn, tạo bản cấu hình sai trong thư mục tạm, chạy cùng `promtool check config` với image tương thích và xác nhận mã thoát khác 0. Pipeline Jenkins giữ nguyên `set -eu`/propagation mã lỗi, nên lệnh deploy kế tiếp không được thực thi. Xóa fixture sau test; không sửa cấu hình thật để tạo lỗi.

## Kết quả kiểm tra thực tế

Kiểm tra đã thực hiện trong workspace:

- Không có `promtool`, Jenkins CLI hay Groovy CLI trong PATH; Docker CLI và daemon sẵn sàng. Dùng image `prom/prometheus:v2.53.4`, trùng phiên bản với Compose.
- Kiểm tra cấu hình thật bằng `docker run --rm -v "${PWD}:/workspace:ro" -w /workspace --entrypoint promtool prom/prometheus:v2.53.4 check config prometheus/prometheus.yml`: thành công; Prometheus xác nhận file cấu hình và một rule file hợp lệ.
- Lần chạy đầu tiên thiếu `--entrypoint promtool` và thất bại (`prometheus: error: unexpected promtool`). Đã sửa pipeline để gọi đúng entrypoint, sau đó chạy validation thật thành công.
- Fixture cấu hình sai tạm thời được kiểm tra bằng cùng image. `promtool` từ chối YAML với exit code `1`; sentinel mô phỏng Deploy không được tạo. Fixture đã được dọn. Static check trên Jenkinsfile xác nhận stage `Validate Config` đứng trước `Deploy`, lệnh validation hiện diện và không có `|| true`.
- `docker compose config --quiet` thành công; runtime Prometheus/Grafana đang `Up`. Không chạy Jenkins end-to-end vì không có Jenkins server/CLI hoặc Groovy validator; không chạy stage Deploy/restart thực tế chỉ để thử pipeline.
- Các thay đổi đường dẫn tham chiếu rule/target thành tương đối đã được `promtool check config` xác minh. Cấu hình trong container vẫn đặt cả file trong `/etc/prometheus`; các file/mount và hành vi trước đó được giữ nguyên.
- Jenkinsfile dùng Declarative Pipeline với lựa chọn `sh` hoặc `bat`; chưa được validate bằng Jenkins Pipeline Linter và chưa chạy trong một Jenkins agent thực tế.
