# Bài tập Grafana + Prometheus (Bài 1–4)

> Bộ cấu hình để thực hành và tự chụp minh chứng nộp Portal. Chưa chạy Docker thực tế trong môi trường tạo tài liệu này. Các metric chỉ xuất hiện sau khi các dịch vụ chạy và Prometheus thu thập thành công.

## Cấu trúc

- `docker-compose.yml`: Prometheus, Node Exporter và Grafana; Grafana cổng 3000.
- `prometheus.yml`: cấu hình thu thập dữ liệu Prometheus và Node Exporter.
- `spring-boot/application.example.yml`: ví dụ bật `/actuator/prometheus` và gắn nhãn `application`.
- `prometheus-multi-server.example.yml`: ví dụ ba Node Exporter ở ba máy thật (bài 4); chưa áp dụng vào hệ demo mặc định.
- `screenshots/`: lưu ảnh tự chụp sau khi chạy.

## Khởi động

```powershell
docker compose config
docker compose up -d
docker compose ps
```

Mở `http://localhost:9090/targets` và kiểm tra hai target `prometheus` và `node-exporter` trạng thái **UP**. Mở `http://localhost:3000`, tài khoản mặc định `admin/admin`; đổi mật khẩu khi được yêu cầu. Chỉ dùng thông tin đăng nhập mặc định cho demo nội bộ, không công khai cổng ra Internet.

**Lưu ý Windows + Docker Desktop:** Node Exporter chạy trong Docker Desktop đo môi trường Linux của Docker, **không phải số liệu thực của Windows host**. Muốn giám sát VPS Linux thật cần chạy Node Exporter trên VPS và khai báo target Prometheus tương ứng.

## Bài 1 — Kết nối Grafana / Prometheus

Vào **Connections → Add new connection** (hoặc **Connections → Data sources → Add new data source** tùy giao diện). Chọn **Prometheus**, nhập **Prometheus server URL:** `http://prometheus:9090`, chọn **Save & test**.

Phải dùng `http://prometheus:9090` vì `prometheus` là tên service trong Docker Compose và hai container chia sẻ mạng mặc định. `localhost` ở bên trong container Grafana chỉ chính container Grafana, không phải container Prometheus. Trình duyệt trên máy host mới dùng `http://localhost:9090` do đã map port.

## Bài 2 — Dashboard CPU / RAM

Tạo Dashboard mới; thêm 2 panels, data source `Prometheus`:

**CPU (%), Time series:**

```promql
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{job="node-exporter",mode="idle"}[5m])))
```

Trong **Standard options → Unit**, chọn **Percent (0-100)**; trục từ 0 đến 100 nếu cần.

**RAM Free, Stat:**

```promql
node_memory_MemFree_bytes{job="node-exporter"}
```

Trong **Standard options → Unit** chọn **Data → bytes (SI)** để số Byte tự được hiển thị thành MB/GB, không cần nhân chia trong PromQL. Với đơn vị nhị phân chọn **bytes (IEC)** để hiển thị MiB/GiB. Với nhiều máy, panel Stat có thể hiện nhiều giá trị; chọn một `instance` nếu muốn một số duy nhất.

**Phân biệt:** `MemFree` = RAM hoàn toàn trống; `MemAvailable` = ước tính RAM có thể dùng mà không swap mạnh, thường hữu ích hơn để ra quyết định nâng cấp server.

## Bài 3 — Import JVM Micrometer ID 4701

1. Chạy Spring Boot với dependency `spring-boot-starter-actuator` và `micrometer-registry-prometheus`.
2. Ghép nội dung mẫu trong `spring-boot/application.example.yml` vào `application.yml` của app, giữ các thiết lập sẵn có.
3. Kiểm tra trên máy đang chạy app: `http://localhost:8080/actuator/prometheus`.
4. Bổ sung job `spring-boot` trong `prometheus.yml` (bỏ comment ở phần mẫu). Trên Docker Desktop và app chạy ngoài Docker dùng `host.docker.internal:8080`; nếu app nằm trong cùng network Compose dùng `ten-service-app:8080`.
5. Khởi động lại Prometheus bằng `docker compose restart prometheus`, kiểm tra `/targets` báo `UP` cho job Spring Boot.
6. Trong Grafana: **Dashboards → New → Import dashboard → 4701 → Load → chọn Prometheus → Import**.
7. Ở variable **application**, chọn `spring-demo` (hoặc tên thực tế).

Ví dụ ba biểu đồ quan trọng để thuyết minh:

- **JVM Heap Memory:** bộ nhớ heap dùng/được cấp, giúp phát hiện tăng bộ nhớ và nguy cơ OOM.
- **GC Pause / Garbage Collection:** thời gian JVM tạm dừng cho dọn rác; tăng bất thường ảnh hưởng độ trễ.
- **HTTP Request Duration/Latency:** thời gian xử lý request, phát hiện API chậm; tùy dashboard/phiên bản Micrometer mà tên panel/metric thực tế có thể khác.

**Câu bẫy:** Dashboard 4701 phân biệt ứng dụng qua common tag `application`, nên cần:

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
```

Lỗi `No data` còn có thể đến từ target `DOWN`, app chưa phát metric, chưa khớp job/label hoặc dashboard cũ không tương thích metric hiện tại. Kiểm tra trong Prometheus: `jvm_memory_used_bytes{application="spring-demo"}`.

## Bài 4 — Variable instance

Bắt đầu từ một dashboard. Trong **Dashboard settings → Variables → Add variable** (hoặc editor **Edit → Add → Variable** tùy phiên bản):

- Name: `instance`
- Type: `Query`
- Data source: `Prometheus`
- Query type hiện đại: `Label values`, Metric = `node_uname_info`, Label = `instance`
- Hoặc ở `Classic query` dùng `label_values(node_uname_info, instance)` (đây là cú pháp cho biến Grafana, **không phải** một biểu thức PromQL độc lập).

Thay query hard-code:

```promql
node_memory_MemFree_bytes{instance="192.168.1.10:9100"}
```

Bằng:

```promql
node_memory_MemFree_bytes{instance="$instance"}
```

Nếu bật Multi-value hoặc Include All, dùng toán tử regex:

```promql
node_memory_MemFree_bytes{instance=~"$instance"}
```

Để hiển thị thực sự ba server, Prometheus phải thu thập đủ ba Node Exporter riêng biệt. Xem `prometheus-multi-server.example.yml`; không chỉ đặt ba tên trên dashboard là tạo ra ba server.

## Minh chứng cần tự chụp

- Bài 1: `docker compose ps`, màn hình Data source với `Save & test` thành công.
- Bài 2: dashboard thật có Time series CPU và Stat RAM.
- Bài 3: dashboard JVM ID 4701 có dữ liệu, biến `application` không rỗng, ảnh cấu hình Spring Boot.
- Bài 4: menu `instance` có các giá trị thật và biểu đồ thay đổi khi chọn server.

## Gửi GitHub

```powershell
git init
git add .
git commit -m "docs: complete Grafana and Prometheus exercises 1-4"
# Sau khi tự tạo repository GitHub, thay <URL_REPO> bằng URL thực:
git branch -M main
git remote add origin <URL_REPO>
git push -u origin main
```

Dán URL repository lên Portal. Không commit mật khẩu thật, token, `.env`, dữ liệu nhạy cảm hoặc tuyên bố đã chạy thành công nếu chưa có bằng chứng.

## Nguồn tham khảo

- https://docs.docker.com/compose/how-tos/networking/
- https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/
- https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/
- https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/configure-standard-options/
- https://grafana.com/grafana/dashboards/4701-jvm-micrometer/
- https://docs.spring.io/spring-boot/reference/actuator/metrics.html
- https://prometheus.io/docs/guides/node-exporter/
