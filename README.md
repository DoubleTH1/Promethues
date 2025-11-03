# Cài đặt node-exporter, prometheus, grafana 
## Trên slave

| Thành phần        | Loại file             | Đường dẫn                                   | Ghi chú                         |
| ----------------- | --------------------- | ------------------------------------------- | ------------------------------- |
| **Node Exporter** | File thực thi         | `/usr/local/bin/node_exporter`              | Binary chính                    |
|                   | File cấu hình service | `/etc/systemd/system/node_exporter.service` | Systemd unit                    |
| **Prometheus**    | File thực thi         | `/usr/local/bin/prometheus`                 | Binary chính                    |
|                   | File cấu hình chính   | `/etc/prometheus/prometheus.yml`            | Danh sách scrape targets        |
|                   | File cấu hình service | `/etc/systemd/system/prometheus.service`    | Systemd unit                    |
| **Grafana**       | File thực thi         | `/usr/local/grafana`                        | Thư mục chứa binary và cấu hình |
|                   | File cấu hình service | `/etc/systemd/system/grafana.service`       | Systemd unit                    |

## Trên master

| Thành phần        | Loại file             | Đường dẫn                                    | Ghi chú      |
| ----------------- | --------------------- | -------------------------------------------- | ------------ |
| **Node Exporter** | File thực thi         | `/usr/local/bin/node_exporter/node_exporter` | Binary chính |
|                   | File cấu hình service | `/etc/systemd/system/node_exporter.service`  | Systemd unit |

Các file nén nằm ở thư mục '/root/'
## Kiến trúc tổng thể
Thực hiện cài đặt trên 2 node là master và slave
```text
                                       ┌─────────────────────────────────────────────────┐
                                       │              GRAFANA SERVER                     │
                                       │----------------------------------------         │
                                       │  - Giao diện Dashboard (Web UI)                 │
                                       │  - Kết nối đến Prometheus qua API               │
                                       │  - Port: 3000                                   │
                                       │  - File thực thi: /usr/local/grafana/           │
                                       │  - Config: /etc/systemd/system/grafana.service  │
                                       └──────────────┬──────────────────────────────────┘
                                                      │
                                      HTTP (API query │
                                      PromQL)         │
                                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           PROMETHEUS SERVER (Slave)                          │
│------------------------------------------------------------------------------│
│  - Thu thập metrics từ Node Exporter trên các node                           │
│  - Lưu dữ liệu vào Time-Series Database (TSDB)                               │
│  - Cung cấp API cho Grafana truy vấn dữ liệu                                 │
│                                                                              │
│  ⚙️ Cấu hình: /etc/prometheus/prometheus.yml                                 │
│  ⚙️ Dịch vụ:  /etc/systemd/system/prometheus.service                         │
│  ⚙️ File thực thi: /usr/local/bin/prometheus                                 │
│  ⚙️ Port: 9090 (default)                                                     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │   scrape_configs:                                                      │  │
│  │     - job_name: 'node_exporter_master'                                 │  │
│  │       static_configs: [targets: ['ip:port']]                           │  │
│  │     - job_name: 'node_exporter_slave'                                  │  │
│  │       static_configs: [targets: ['ip:port']]                           │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────┬───────────────────────────────────────────────────────────────┘
               │
               │ HTTP GET /metrics
               │ (Prometheus pulls data định kỳ, ví dụ mỗi 15s)
               ▼
        ┌───────────────────────────────────────────────────────┐
        │               MASTER NODE                             │
        │--------------------------------------------           │
        │  Node Exporter                                        │
        │  - Port: 9100 (default)                               │
        │  - Binary: /usr/local/bin/node_exporter               │
        │  - Service: /etc/systemd/system/node_exporter.service │
        │  → Cung cấp metrics: CPU, RAM, Disk, Net...           │
        └───────────────────────────────────────────────────────┘
               ▲
               │
               │ HTTP GET /metrics
               ▼
        ┌────────────────────────────────────────────────────────┐
        │               SLAVE NODE                               │
        │--------------------------------------------            │
        │  Node Exporter                                         │
        │  - Port: 9100 (default)                                │
        │  - Binary: /usr/local/bin/node_exporter                │
        │  - Service: /etc/systemd/system/node_exporter.service  │
        │  → Cung cấp metrics: CPU, RAM, Disk, Net...            │
        └────────────────────────────────────────────────────────┘
```

# Cài đặt node exporter
Trên node master <br>
Tìm và tải về phiên bản mới nhất của node exporter tại https://prometheus.io/download/
```sh
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
```
Giải nén file vừa tải về
```sh
sudo tar -xvzv node_exporter-1.10.2.linux-amd64.tar.gz
```
Tạo một user cho việc quản lý exporter
```sh
sudo useradd -rs /bin/false node_exporter
```
Copy binary file trong thư mục đã giải nén tới địa chỉ /usr/local/bin
```sh
cd node_exporter-1.0.1.linux-amd64/
cp node_exporter /usr/local/bin
```
Thiết lập quyền cho binary file
```sh
sudo node_exporter:node_exporter /usr/local/bin/node_exporter
```
Tạo một service cho việc chạy Node Exporter
```sh
cd /lib/systemd/system
sudo vim node_exporter.service
```
Thêm vào file nội dung sau
```conf
[Unit]
Description=Node Exporter
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
Khởi chạy service vừa mới tạo
```sh
sudo systemctl enable node_exporter.service
sudo systemctl start node_exporter.service
```
Kiểm tra service node exporter
```sh
sudo systemctl status node_exporter.service
```
- Trạng thái của service là active là dịch vụ đã hoạt động port mặc định là 9100
- Trường hợp trạng thái của dịch vụ fail xem chi tiết log hoạt động của node exporter
```sh
sudo journalctl -u node_exporter -xe
```
Sau khi node exporter được cài đặt và chạy có thể xác minh các metrics đang được export bằng cách:
```sh
sudo curl http://localhost:9100/metrics
```
Output sẽ trông như sau:
```output
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 3.8996e-05
go_gc_duration_seconds{quantile="0.25"} 4.5926e-05
go_gc_duration_seconds{quantile="0.5"} 5.846e-05
# etc.
```
**Cài tương tự trên node slave**
## Cài đặt prometheus
Tìm và tải về phiên bản mới nhất của promethues tại https://prometheus.io/download/
```sh
sudo wget https://github.com/prometheus/prometheus/releases/download/v3.7.3/prometheus-3.7.3.linux-amd64.tar.gz
```
Giải nén file vừa tải về
```sh
sudo tar -xvzf prometheus-3.7.3.linux-amd64.tar.gz
cd prometheus-3.7.3.linux-amd64.tar.gz
```
Copy các binary file trong thư mục đã giải nén vào /usr/local/bin
```sh
sudo cp prometheus promtool /usr/local/bin
```
Tạo user và group để quản lý prometheus
```sh
sudo useradd -rs /bin/false prometheus
```
Tạo thư mục để lưu trữ dữ liệu và file cấu hình
```sh
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```
Thiết lập quyền cho các thư mục
```sh
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus 
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```
Tạo systemd service cho prometheus
```sh
cd /etc/systemd/system
sudo vim prometheus.service
```
```conf
[Unit]
Description=Prometheus
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/data \   #thư mục lưu trữ dữ liệu
  --web.listen-address=:9090 \
  --web.console.templates=/etc/prometheus/consoles \    #thư mục chứa giao diện console
  --web.console.libraries=/etc/prometheus/console_libraries  #thư mục chứa thư viện console

[Install]
WantedBy=multi-user.target
```
Cấu hình /etc/prometheus/prometheus.yml
```sh 
cd /etc/prometheus
sudo vim prometheus.yml 
```
Thêm vào file nội dung sau, thay ip:port bằng địa chỉ IP và port của node exporter trên các node master và slave
```yaml 
global:
  scrape_interval: 15s  
    evaluation_interval: 15s
scrape_configs:
  - job_name: 'node_exporter_master'
    static_configs:
      - targets: ['ip_master:9100']   #thay ip_master bằng địa chỉ IP của node master   
    - job_name: 'node_exporter_slave'
      static_configs:
        - targets: ['ip_slave:9100']    #thay ip_slave bằng địa chỉ IP của node slave       
```
Khởi chạy service prometheus
```sh 
sudo systemctl daemon-reload
sudo systemctl enable prometheus.service    
sudo systemctl start prometheus.service
```
Kiểm tra trạng thái của prometheus
```sh   
sudo systemctl status prometheus.service
```
- Trạng thái của service là active là dịch vụ đã hoạt động port mặc định là 9090
- Trường hợp trạng thái của dịch vụ fail xem chi tiết log hoạt động của prometheus
```sh
sudo journalctl -u prometheus -xe
```
Sau khi prometheus được cài đặt và chạy có thể truy cập giao diện web của prometheus để kiểm tra
Mở trình duyệt và truy cập địa chỉ http://ip_slave:9090 (thay ip_slave bằng địa chỉ IP của node slave)
## Cài đặt Grafana  
Tìm và tải về phiên bản mới nhất của Grafana tại https://grafana.com/grafana/download
```sh
wget https://dl.grafana.com/grafana-enterprise/release/12.2.1/grafana-enterprise_12.2.1_18655849634_linux_amd64.tar.gz
```  
Giải nén file vừa tải về
```sh
tar -zxvf grafana-enterprise_12.2.1_18655849634_linux_amd64.tar.gz
```
Tạo user và group để quản lý grafana
```sh
sudo useradd -rs /bin/false grafana
```
Di chuyển thư mục grafana vừa giải nén tới /usr/local/grafana
```sh
sudo mv grafana-12.2.1 /usr/local/grafana
```
Phân quyền cho thư mục grafana
```sh
sudo chown -R grafana:grafana /usr/local/grafana
```
Tạo systemd service cho grafana
```sh
cd /etc/systemd/system
sudo vim grafana.service
```
Thêm cấu hình sau vào file
```conf
[Unit]
Description=Grafana Server 
After=network-online.target
[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/grafana/bin/grafana-server \
  --config=/usr/local/grafana/conf/defaults.ini \
  --homepath=/usr/local/grafana
Restart=on-failure
[Install]
WantedBy=multi-user.target
```
Khởi chạy service grafana
```sh
sudo systemctl daemon-reload
sudo systemctl enable grafana.service
sudo systemctl start grafana.service
```
Kiểm tra trạng thái của grafana
```sh
sudo systemctl status grafana.service
```
- Trạng thái của service là active là dịch vụ đã hoạt động port mặc định là 3000
- Trường hợp trạng thái của dịch vụ fail xem chi tiết log hoạt động của grafana
```sh
sudo journalctl -u grafana -xe
```
Sau khi grafana được cài đặt và chạy có thể truy cập giao diện web của grafana để kiểm tra
Mở trình duyệt và truy cập địa chỉ http://ip_slave:3000 (thay ip_slave bằng địa chỉ IP của node slave) <br>
Đăng nhập với tài khoản mặc định
- Username: admin
- Password: admin
Sau khi đăng nhập lần đầu sẽ được yêu cầu đổi mật khẩu
## Kết nối Grafana với Prometheus
- Mở giao diện web của Grafana tại http://ip_slave:3000
- Đăng nhập với tài khoản admin
- Chọn biểu tượng bánh răng (⚙️) ở thanh bên trái để vào phần "Configuration"
- Chọn "Data Sources"
- Nhấn nút "Add data source"
- Chọn "Prometheus" từ danh sách các loại data source
- Cấu hình các thông số sau:
  - Name: Prometheus
  - URL: `http://ip_slave:9090` (thay ip_slave bằng địa chỉ IP của node slave nơi Prometheus đang chạy)
- Nhấn nút "Save & Test" để lưu cấu hình và kiểm tra kết nối
Nếu kết nối thành công, bạn sẽ thấy thông báo "Data source is working".
## Tạo Dashboard giám sát
- Mở giao diện web của Grafana tại http://ip_slave:3000
- Đăng nhập với tài khoản admin
- Chọn biểu tượng dấu cộng (+) ở thanh bên trái và chọn "Dashboard"
- Nhấn "Add new panel"
- Trong phần "Query", chọn data source là "Prometheus"
- Nhập truy vấn PromQL để lấy dữ liệu từ Node Exporter, ví dụ:
    - CPU Usage: `100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
    - Memory Usage: `node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes`
    - Disk Usage: `node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} - node_filesystem_free_bytes{fstype!~"tmpfs|overlay"}`
    - Network Traffic: `rate(node_network_receive_bytes_total[5m])` và `rate(node_network_transmit_bytes_total[5m])`
- Tùy chỉnh biểu đồ theo ý muốn và nhấn "Apply" để thêm vào dashboard
- Lặp lại các bước trên để thêm nhiều panel giám sát khác nhau vào dashboard
- Sau khi hoàn tất, nhấn "Save dashboard" để lưu lại dashboard của bạn
# Chi tiết về cách hoạt động của node exporter
Node Exporter là một công cụ thu thập và xuất các metrics hệ thống từ các node (máy chủ) trong môi trường của bạn. Dưới đây là chi tiết về cách hoạt động của Node Exporter:
1. **Cài đặt và Khởi động**:
   - Node Exporter được cài đặt trên mỗi node mà bạn muốn giám sát. Sau khi cài đặt, nó được khởi động như một dịch vụ hệ thống (systemd service) và chạy liên tục trên node đó.
2. **Thu thập Metrics**:
   - Node Exporter thu thập các metrics hệ thống từ node, bao gồm CPU, bộ nhớ, đĩa, mạng và nhiều khía cạnh khác của hệ thống. Nó sử dụng các thư viện và API hệ thống để truy xuất thông tin này.
3. **Xuất Metrics**:
    - Node Exporter xuất các metrics đã thu thập dưới dạng định dạng mà Prometheus có thể hiểu được. Các metrics này được cung cấp thông qua một endpoint HTTP (mặc định là `http://<node_ip>:9100/metrics`).
4. **Tích hợp với Prometheus**:
   - Prometheus được cấu hình để định kỳ truy cập endpoint của Node Exporter trên mỗi node. Prometheus gửi các yêu cầu HTTP GET đến endpoint này để thu thập dữ liệu metrics.
   - Prometheus lưu trữ các metrics thu thập được trong cơ sở dữ liệu time-series của nó, cho phép truy vấn và phân tích dữ liệu theo thời gian.
5. **Giám sát và Cảnh báo**:
   - Dữ liệu metrics thu thập từ Node Exporter có thể được sử dụng để tạo các dashboard giám sát trong Grafana hoặc các công cụ tương tự.
   - Có thể thiết lập các quy tắc cảnh báo trong Prometheus dựa trên các metrics này để nhận thông báo khi có sự cố xảy ra trên các node.
Tóm lại, Node Exporter hoạt động như một cầu nối giữa hệ thống của bạn và Prometheus, cung cấp dữ liệu quan trọng để giám sát hiệu suất và sức khỏe của các node trong môi trường của bạn.
# Chi tiết về cách hoạt động của Prometheus
Prometheus là một hệ thống giám sát và cảnh báo mã nguồn mở được thiết kế để thu thập, lưu trữ và truy vấn các metrics từ các ứng dụng và hệ thống.
1. **Thu thập Metrics**:
    - Prometheus thu thập các metrics từ các nguồn khác nhau, bao gồm Node Exporter, ứng dụng tùy chỉnh và các dịch vụ khác. Nó sử dụng mô hình "pull" để định kỳ gửi các yêu cầu HTTP GET đến các endpoint cung cấp metrics.
    - Các nguồn metrics phải tuân theo định dạng mà Prometheus hiểu được, thường là định dạng text-based.
2. **Lưu trữ Dữ liệu**:
    - Các metrics thu thập được được lưu trữ trong cơ sở dữ liệu time-series của Prometheus. Dữ liệu được lưu trữ theo thời gian, cho phép truy vấn và phân tích các xu hướng và mẫu trong dữ liệu.
    - Prometheus sử dụng một định dạng lưu trữ hiệu quả để tối ưu hóa việc lưu trữ và truy xuất dữ liệu.
3. **Truy vấn Dữ liệu**:
    - Prometheus cung cấp một ngôn ngữ truy vấn mạnh mẽ gọi là PromQL (Prometheus Query Language) để truy vấn dữ liệu metrics. Có thể sử dụng PromQL để tạo các biểu đồ, bảng và cảnh báo dựa trên dữ liệu thu thập được.
    - PromQL hỗ trợ các phép toán phức tạp, cho phép bạn phân tích dữ liệu theo nhiều cách khác nhau.       
4. **Cảnh báo**:
    - Prometheus có khả năng thiết lập các quy tắc cảnh báo dựa trên các metrics thu thập được. Khi một điều kiện cảnh báo được kích hoạt, Prometheus có thể gửi thông báo qua các kênh khác nhau như email, Slack hoặc các hệ thống quản lý sự cố.
5. **Tích hợp với Công cụ Giám sát**:
    - Prometheus tích hợp tốt với các công cụ giám sát và trực quan hóa như Grafana. Có thể sử dụng Grafana để tạo các dashboard tùy chỉnh dựa trên dữ liệu từ Prometheus.
Tóm lại, Prometheus hoạt động như một hệ thống giám sát toàn diện, cung cấp các công cụ để thu thập, lưu trữ, truy vấn và cảnh báo dựa trên dữ liệu metrics từ các hệ thống và ứng dụng của bạn.
# Chi tiết về cách hoạt động của Grafana
Grafana là một nền tảng mã nguồn mở để trực quan hóa và giám sát dữ liệu từ nhiều nguồn khác nhau, bao gồm Prometheus. 
1. **Kết nối với Nguồn Dữ liệu**:
   - Grafana hỗ trợ kết nối với nhiều loại nguồn dữ liệu khác nhau, bao gồm Prometheus, InfluxDB, Elasticsearch và nhiều hơn nữa. Có thể cấu hình các nguồn dữ liệu này thông qua giao diện web của Grafana.
   - Khi kết nối với Prometheus, Grafana sử dụng API của Prometheus để truy vấn dữ liệu metrics.
2. **Tạo Dashboard**:
   - Grafana cho phép người dùng tạo các dashboard tùy chỉnh để trực quan hóa dữ liệu. Người dùng có thể thêm nhiều panel vào dashboard, mỗi panel có thể hiển thị dữ liệu dưới dạng biểu đồ, bảng, bản đồ nhiệt và nhiều loại hình trực quan khác.
   - Mỗi panel có thể được cấu hình để sử dụng các truy vấn PromQL để lấy dữ liệu từ Prometheus.
3. **Truy vấn Dữ liệu**:
   - Grafana sử dụng ngôn ngữ truy vấn của nguồn dữ liệu để lấy dữ liệu. Khi sử dụng Prometheus, Grafana gửi các truy vấn PromQL đến Prometheus để lấy dữ liệu metrics cần thiết cho các panel trên dashboard.
   - Người dùng có thể tùy chỉnh các truy vấn này để hiển thị dữ liệu theo cách họ muốn.
4. **Cập nhật Dữ liệu Thời gian Thực**:
   - Grafana hỗ trợ cập nhật dữ liệu thời gian thực trên các dashboard. Người dùng có thể cấu hình tần suất làm mới dữ liệu cho từng panel hoặc toàn bộ dashboard, cho phép họ theo dõi các thay đổi trong dữ liệu một cách liên tục.
5. **Cảnh báo**:
   - Grafana có khả năng thiết lập các quy tắc cảnh báo dựa trên dữ liệu hiển thị trên các panel. Khi một điều kiện cảnh báo được kích hoạt, Grafana có thể gửi thông báo qua các kênh khác nhau như email, Slack hoặc các hệ thống quản lý sự cố.
6. **Quản lý Người dùng và Quyền Truy cập**:
   - Grafana cung cấp các tính năng quản lý người dùng và quyền truy cập, cho phép người quản trị kiểm soát ai có thể xem hoặc chỉnh sửa các dashboard và nguồn dữ liệu.
   - Người dùng có thể được phân quyền ở mức độ tổ chức, nhóm hoặc cá nhân.
7. **Mở rộng và Tùy chỉnh**:
   - Grafana hỗ trợ các plugin để mở rộng chức năng của nó. Người dùng có thể cài đặt các plugin để thêm các loại panel mới, nguồn dữ liệu hoặc tính năng bổ sung.
   - Grafana cũng cung cấp một API để người dùng có thể tự động hóa việc quản lý dashboard và nguồn dữ liệu.
Tóm lại, Grafana hoạt động như một nền tảng trực quan hóa mạnh mẽ, cho phép người dùng tạo các dashboard tùy chỉnh để giám sát và phân tích dữ liệu từ nhiều nguồn khác nhau, bao gồm Prometheus.
