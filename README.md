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
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.darwin-amd64.tar.gz
```
Giải nén file vừa tải về
```sh
sudo tar -xvzv node_exporter-1.10.2.darwin-amd64.tar.gz
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
