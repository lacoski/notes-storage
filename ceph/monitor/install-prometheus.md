# Hướng dẫn cài đặt Prometheus trên CentOS7


**Tắt Firewall và Selinux**

Thực hiện tắt Firewall và Selinux trên các node

```sh
sed -i 's/\(^SELINUX=\).*/\SELINUX=disabled/' /etc/sysconfig/selinux
sed -i 's/\(^SELINUX=\).*/\SELINUX=disabled/' /etc/selinux/config
setenforce 0
systemctl stop firewalld
systemctl disable firewalld
```

### 1. Cài đặt và cấu hình Prometheus server

**Bước 1: Chuẩn bị**

```sh
yum update -y
yum install vim git wget epel-release -y
```

**Bước 2: Tạo service user**

```sh
useradd --no-create-home --shell /bin/false prometheus
```

**Bước 3: Cấu hình thư mục cấu hình và thư mục lưu trữ cho prometheus**

```sh
mkdir /etc/prometheus
mkdir /var/lib/prometheus
chown prometheus:prometheus /etc/prometheus
chown prometheus:prometheus /var/lib/prometheus
```

**Bước 4: Tải source code prometheus**  

```sh
cd /opt
wget https://github.com/prometheus/prometheus/releases/download/v2.12.0/prometheus-2.12.0.linux-amd64.tar.gz
tar xvf prometheus-2.12.0.linux-amd64.tar.gz

cp prometheus-2.12.0.linux-amd64/prometheus /usr/local/bin/
cp prometheus-2.12.0.linux-amd64/promtool /usr/local/bin/

chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool

cp -r prometheus-2.12.0.linux-amd64/consoles /etc/prometheus
cp -r prometheus-2.12.0.linux-amd64/console_libraries /etc/prometheus

chown -R prometheus:prometheus /etc/prometheus/consoles
chown -R prometheus:prometheus /etc/prometheus/console_libraries

rm -rf prometheus-2.12.0.linux-amd64*
cd -
```

**Bước 5: Cấu hình file config**

Tạo một file config mới với nội dung như sau:

```sh
cat <<EOF > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
EOF
```

**Bước 6: Chạy thử Prometheus bằng câu lệnh sau:**

```sh
sudo -u prometheus /usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
```

Nếu có lỗi thì xem lại file cấu hình. `ctrl + c` để thoát.

**Bước 7: Đặt tiến trình trong systemd để dễ dàng quản lý**

```sh
cat <<EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF
```

Khởi động Prometheus:

```sh
systemctl daemon-reload
systemctl start prometheus
systemctl status prometheus
systemctl enable prometheus
```

Truy cập vào đường dẫn để sử dụng giao diện web của Prometheus `http://192.168.70.71:9090/graph`


## 2. Thêm node cần giám sát

Thực hiện cài đặt và cấu hình trên CentOS 7

**Bước 1: Tạo user cho prometheus**

```sh
useradd --no-create-home --shell /bin/false node_exporter
```

**Bước 2: Tải source code**

```sh
cd /opt
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz

tar xvf node_exporter-0.18.1.linux-amd64.tar.gz

cp node_exporter-0.18.1.linux-amd64/node_exporter /usr/local/bin
chown node_exporter:node_exporter /usr/local/bin/node_exporter

rm -rf node_exporter-0.18.1.linux-amd64*
cd -
```

**Bước 3: Chạy exporter dưới systemd**

```sh
cat <<EOF >  /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

Khởi động service:

```sh
systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
```

Truy cập vào đường dẫn để thấy các metric thu thập được `http://192.168.68.91:9100/metrics`

**Bước 4: Add thêm jobs vào node Prometheus server**

Thêm cấu hình của node exporter mới cài đặt vào file `/etc/prometheus/prometheus.yml` như sau:
```sh
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.68.91:9100']
```

Khởi động lại service:

```sh
systemctl restart prometheus
```

Giờ ta có thể thực hiện query tới các node exporter từ prometheus server.

## 3. Install grafana

Thêm repo `/etc/yum.repos.d/grafana.repo`

```
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

Install grafana

`yum install grafana -y`

```
yum install fontconfig freetype* urw-fonts -y
```

Start grafana

```
systemctl start grafana-server
systemctl enable grafana-server.service
```

Install plugin

```
grafana-cli plugins install vonage-status-panel
grafana-cli plugins install grafana-piechart-panel
```
