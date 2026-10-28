# Bài 5: Debug App bằng Metrics

## Hiện trạng workspace

Đã kiểm tra workspace nhưng không tìm thấy ứng dụng Spring Boot, `pom.xml`, `build.gradle`, Gradle Kotlin build file, Maven/Gradle wrapper, `Dockerfile` ứng dụng, hay `application.yml`/`application.yaml`. Vì vậy không có cấu hình exposure hoặc dependency thực tế để sửa, không thể chạy build hay khởi động ứng dụng. Không tạo ứng dụng mới để giả lập kết quả.

Trước thay đổi, job `backend-app` trong `prometheus/prometheus.yml` scrape `backend:8080` nhưng không đặt `metrics_path`, nên Prometheus dùng mặc định `/metrics`. Spring Boot Actuator thường cung cấp metrics Prometheus ở `/actuator/prometheus`; scrape sai path có thể gây HTTP 404 nếu một ứng dụng tương thích đang chạy. Tuy nhiên, lỗi runtime hiện quan sát được không phải 404: DNS không phân giải được hostname `backend` trong Docker network. Kiểm tra `http://localhost:8080/actuator/prometheus` trên host cũng không kết nối được vì không có service lắng nghe.

## Thay đổi đã thực hiện

Đã giữ nguyên job hiện có, target `backend:8080`, các job khác, `rule_files`, chu kỳ scrape và cấu hình Grafana; chỉ đặt path metrics:

```yaml
- job_name: backend-app
  metrics_path: /actuator/prometheus
  static_configs:
    - targets:
        - backend:8080
```

Sau thay đổi, lỗi HTTP 404 do scrape path mặc định chỉ được xử lý nếu service `backend` thực sự tồn tại và expose endpoint tại path này. Thay đổi path không tạo ra ứng dụng, metric hay target backend.

## Dependencies và cấu hình cần có trong ứng dụng

Hai thành phần cần có trong một ứng dụng Spring Boot phù hợp:

- `spring-boot-starter-actuator`: cung cấp Actuator endpoint framework.
- `micrometer-registry-prometheus`: cung cấp registry và định dạng scrape Prometheus.

Không thể xác nhận hai dependency đã có hay chưa vì workspace không chứa manifest dependency. Cấu hình exposure thường cần cho `application.yml` của chính ứng dụng:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus"
```

Giữ lại endpoint đang được expose; không dùng `include: "*"`. Cũng cần kiểm tra profile/biến môi trường ghi đè, `management.endpoint.prometheus`, `management.server.port`, context path và Spring Security. `application.yml` này không được tạo ở workspace giám sát, vì không có ứng dụng đích để đặt cấu hình một cách an toàn.

Nếu cần bổ sung ứng dụng để hoàn thành phần này, hãy đưa mã nguồn Spring Boot vào workspace hoặc cung cấp đường dẫn đến repo ứng dụng; cấu trúc tối thiểu cần có build manifest (Maven `pom.xml` hoặc Gradle build file), source entry point, `application.yml`, dependency Actuator và Prometheus registry, cùng cấu hình chạy/service kết nối được từ network Prometheus. Không có các dữ liệu này nên chưa thể đưa ra build command thực tế cho ứng dụng.

## Các bước build và xác minh khi có ứng dụng

Chọn wrapper đúng với dự án:

```powershell
.\mvnw.cmd test
.\mvnw.cmd package
```

hoặc:

```powershell
.\gradlew.bat test
.\gradlew.bat build
```

Chạy ứng dụng theo cách mà dự án hiện có hỗ trợ, kiểm tra log startup, rồi gọi:

```powershell
curl.exe -i http://localhost:8080/actuator/prometheus
```

Endpoint mong đợi HTTP 200 với text metrics, ví dụ metric Micrometer/JVM nếu registry và instrumentation tương ứng đã hoạt động. Chỉ khi response thực sự trả metrics mới kết luận endpoint hoạt động. Sau đó kiểm tra Prometheus `/targets`, chạy `up` và có thể truy vấn `jvm_memory_used_bytes`.

Prometheus trong workspace chạy trong Docker, nên `localhost` bên trong container là chính container Prometheus, không phải máy host. Nếu backend cũng trong Compose, dùng DNS service (ví dụ `backend:8080`). Nếu backend ở host, dùng địa chỉ host có thể truy cập từ container, chẳng hạn `host.docker.internal:8080` khi môi trường Docker hỗ trợ.

## Diễn giải lỗi

- **404**: thường do sai path, endpoint chưa được expose, thiếu registry/Actuator hoặc context path khác.
- **401/403**: endpoint bị bảo vệ bởi security; kiểm tra Spring Security và chính sách truy cập metrics, không vô hiệu hóa bảo vệ tùy tiện.
- **Connection refused**: ứng dụng chưa chạy, không lắng nghe đúng port/interface, hoặc port mapping sai.
- **Timeout**: kiểm tra service phản hồi, network và firewall.
- **`no such host`**: tên service chưa tồn tại/không cùng network hoặc hostname sai; đây là lỗi hiện quan sát được đối với `backend`, không phải bằng chứng HTTP 404.

## Kết quả kiểm tra thực tế

- Tìm kiếm workspace không thấy source Spring Boot, build manifest/wrapper, Dockerfile ứng dụng, hay file `application*.yml`/`application*.yaml`. Do đó build, kiểm tra dependency, chỉnh exposure/profile/security, và khởi động lại ứng dụng không thể thực hiện trong workspace hiện có.
- `curl.exe http://localhost:8080/actuator/prometheus`: kết nối thất bại; không có HTTP response hay metric thực tế để xác minh.
- Trước chỉnh sửa, `/api/v1/targets` cho thấy job `backend-app` dùng `/metrics` và thất bại với `lookup backend ... no such host`. Đã đổi path sang `/actuator/prometheus`, giữ target/service hiện có.
- `docker compose config --quiet` thành công; `docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml` hợp lệ và xác nhận rule file Bài 3, scrape jobs Bài 2/4 vẫn có trong cấu hình.
- Đã chạy `docker compose restart prometheus`; log sau restart xác nhận nạp `/etc/prometheus/prometheus.yml` và `Server is ready to receive web requests`, không thấy lỗi parse.
- Sau restart, `/api/v1/targets` xác nhận scrape URL `http://backend:8080/actuator/prometheus`, nhưng target vẫn **DOWN** vì Docker DNS báo `lookup backend ... no such host`. Đây là lỗi thiếu service/backend trong network, không phải HTTP 404. Truy vấn Prometheus `up{job="backend-app"}` trả `0`; target Prometheus tự scrape trả `1`.
- Chưa xác minh `/actuator/prometheus` trả HTTP 200, metric JVM xuất hiện, hoặc target backend chuyển UP; cần cung cấp mã nguồn/khởi chạy service Spring Boot có endpoint được expose trong cùng network (hoặc cấu hình địa chỉ host có thể truy cập từ container). Không kết luận đã khắc phục hoàn toàn lỗi runtime chỉ dựa trên việc đổi scrape path.
