# Bank Network Topology & Telemetry Simulation Engine

Đồ án mô phỏng **mô hình mạng doanh nghiệp của ngân hàng** (Trụ sở chính – Chi nhánh) kết hợp **hệ thống giám sát (observability)** tập trung: topology được dựng trên Cisco Packet Tracer, còn dữ liệu telemetry (CPU, RAM, băng thông WAN, độ trễ, trạng thái link/HSRP/OSPF) được sinh bởi một Python exporter, thu thập bằng Prometheus và hiển thị trên dashboard NOC của Grafana.

---

## 1. Tổng quan

Hệ thống gồm hai phần **tách rời (decoupled)**, cùng bám theo một bản mô tả topology (`topology.yaml`):

| Phần | Công cụ | Vai trò |
|------|---------|---------|
| Mô hình mạng | Cisco Packet Tracer (`Bank_Enterprise_Network.pkt`) | Thiết kế và kiểm tra cấu hình mạng tĩnh: topology HQ – Chi nhánh, VLAN, OSPF, HSRP, ACL |
| Giám sát | Python exporter + Prometheus + Grafana (Docker Compose) | Engine giả lập telemetry độc lập, thu thập và hiển thị trên dashboard |

> **Lưu ý:** Packet Tracer **không** được kết nối trực tiếp với exporter. Exporter không lấy dữ liệu từ Packet Tracer mà tự sinh telemetry theo mô hình giả lập, dựa trên các thiết bị và vai trò được mô tả trong topology. Vì vậy, số liệu trên Grafana là dữ liệu mô phỏng, không phải lưu lượng thật của file `.pkt`.

**Điểm chính**

- Topology gồm **1 HQ + 2 chi nhánh**: Chi nhánh HCM chạy mô hình **High Availability (HSRP, 2 router)**, Chi nhánh Bình Dương là **Standard Branch (1 router)**.
- Phân đoạn mạng theo **VLAN kiểu ngân hàng**: Core Banking, ATM, POS, Workstation, Guest.
- Telemetry được sinh theo **giờ làm việc hành chính** (cao điểm 8:00–11:30 và 13:00–17:00).
- Có **kịch bản giả lập sự cố** để demo: đứt link WAN chính của HCM (HSRP failover) và quá tải chi nhánh Bình Dương.
- Toàn bộ stack giám sát chạy bằng **một lệnh** `docker compose up`.

---

## 2. Kiến trúc hệ thống

```
┌────────────────────────── Cisco Packet Tracer ──────────────────────────┐
│                                                                          │
│                      ┌──────────────────┐                                │
│        ┌─────────────│  HQ-CORE-RTR01   │─────────────┐                  │
│        │  Serial/WAN └────────┬─────────┘  Serial/WAN │                  │
│        │                      │ Gi0/0/0 (trunk)       │                  │
│  ┌─────┴──────┐        ┌──────┴──────┐         ┌──────┴───────┐          │
│  │BR-HCM-RTR01│        │   HQ-SW01   │         │ BR-BD-RTR01  │          │
│  │BR-HCM-RTR02│        │ Core Banking│         │ (Single)     │          │
│  │ (HSRP HA)  │        │ DB + Staff  │         └──────┬───────┘          │
│  └─────┬──────┘        └─────────────┘                │                  │
│   Switch HCM                                      Switch BD             │
│  ATM/POS/Staff/Guest                            ATM/Staff/Guest         │
└──────────────────────────────────────────────────────────────────────────┘

        topology.yaml  (mô tả topology; exporter nạp khi khởi động)
               │
               ▼
   ┌─────────────────────┐  :8000   ┌──────────────┐  :9090   ┌─────────────┐
   │ Python Exporter     │─────────▶│  Prometheus  │─────────▶│   Grafana   │
   │ (telemetry giả lập) │ /metrics │ scrape 5s    │  PromQL  │ NOC :3005   │
   └─────────────────────┘          └──────────────┘          └─────────────┘
```

### 2.1. Topology mạng

![Topology trên Packet Tracer](Figure/6.PNG)

| Khu vực | Thiết bị | Vai trò | Ghi chú |
|---------|----------|---------|---------|
| HQ Data Center | `HQ-CORE-RTR01` | Core Router | Management IP `10.10.99.1`; sub-interface VLAN 10 (CoreBanking `10.10.10.0/24`), VLAN 20 (HQ-Workstation `10.10.20.0/24`) |
| HQ Data Center | `HQ-SW01` | Switch | Kết nối Core-Banking DB (`10.10.10.100`) và PC nhân viên HQ |
| Chi nhánh HCM | `BR-HCM-RTR01` | HA-Master | HSRP priority **110** |
| Chi nhánh HCM | `BR-HCM-RTR02` | HA-Standby | HSRP priority **100** |
| Chi nhánh Bình Dương | `BR-BD-RTR01` | Single Router | Không có HSRP |

### 2.2. Địa chỉ WAN (point-to-point /30)

| Liên kết | Dải mạng | Băng thông |
|----------|----------|-----------|
| HQ ↔ BR-HCM-RTR01 | `192.168.100.0/30` (HQ `.1`, HCM `.2`) | 10 Mbps |
| HQ ↔ BR-BD-RTR01 | `192.168.100.4/30` (HQ `.5`, BD `.6`) | 10 Mbps |
| BR-HCM-RTR02 | WAN IP `192.168.100.10/30` | – |

### 2.3. Phân hoạch VLAN / subnet

| VLAN | Chức năng | HQ | Chi nhánh HCM | Chi nhánh BD |
|------|-----------|----|---------------|--------------|
| 10 | Core Banking | `10.10.10.0/24` | – | – |
| 20 | HQ Workstation | `10.10.20.0/24` | – | – |
| 30 | ATM | – | `10.1.10.0/24` (VIP `.254`) | `10.2.10.0/24` |
| 40 | POS | – | `10.1.20.0/24` (VIP `.254`) | `10.2.20.0/24` |
| 50 | Workstation | – | `10.1.30.0/24` (VIP `.254`) | `10.2.30.0/24` |
| 60 | Guest | – | `10.1.40.0/24` (VIP `.254`) | `10.2.40.0/24` |

> Tại chi nhánh HCM, RTR01 dùng IP `.1`, RTR02 dùng IP `.2`, địa chỉ ảo HSRP (VIP) là `.254`.

---

## 3. Cấu trúc thư mục

```
Bank-Network-Observability/
├── Bank_Enterprise_Network.pkt        # Mô hình mạng Cisco Packet Tracer
├── topology.yaml                      # Ground truth: topology, IP, VLAN, ngưỡng SLA
├── docker-compose.yml                 # Khởi chạy exporter + Prometheus + Grafana
├── grafana_dashboard.json             # Dashboard NOC (import vào Grafana)
├── exporter/
│   ├── exporter.py                    # Bộ sinh telemetry giả lập
│   ├── requirements.txt
│   └── Dockerfile
├── prometheus/
│   └── prometheus.yml                 # Cấu hình scrape (5s)
├── Figure/                            # Ảnh minh họa kết quả
├── Prometheus Time Series Collection and Processing Server.pdf
└── Bank Network NOC Observability Dashboard - Dashboards - Grafana.pdf
```

---

## 4. Công nghệ sử dụng

- **Cisco Packet Tracer** – dựng topology, cấu hình định tuyến/chuyển mạch
- **Python 3.10** – `prometheus-client 0.17.1`, `PyYAML 6.0.1`
- **Prometheus v2.45.0** – thu thập time-series
- **Grafana 10.0.0** – dashboard giám sát
- **Docker & Docker Compose** – đóng gói và triển khai

---

## 5. Hướng dẫn chạy

### 5.1. Yêu cầu

- Docker Desktop (hoặc Docker Engine + Compose v2)
- Cisco Packet Tracer (chỉ cần để mở file `.pkt`)
- Các cổng `8000`, `9090`, `3005` còn trống

### 5.2. Khởi chạy stack giám sát

```bash
cd Bank-Network-Observability
docker compose up -d --build
```

| Dịch vụ | Địa chỉ | Ghi chú |
|---------|---------|---------|
| Exporter | http://localhost:8000/metrics | Metrics dạng Prometheus |
| Prometheus | http://localhost:9090 | Kiểm tra *Status → Targets* |
| Grafana | http://localhost:3005 | Tài khoản mặc định `admin` / `admin` |

Dừng hệ thống:

```bash
docker compose down
```

### 5.3. Cấu hình Grafana

1. Đăng nhập Grafana tại http://localhost:3005.
2. Vào **Connections → Data sources → Add data source → Prometheus**, đặt URL là `http://prometheus:9090`, rồi **Save & test**.
3. Vào **Dashboards → New → Import**, tải lên file `grafana_dashboard.json`, chọn data source Prometheus vừa tạo.
4. Mở dashboard **Bank Network NOC Observability Dashboard** (tự làm mới mỗi 5 giây).

### 5.4. Mở mô hình mạng

Mở file `Bank_Enterprise_Network.pkt` bằng Cisco Packet Tracer. Có thể dùng chế độ **Simulation** để theo dõi gói tin (ví dụ ICMP từ `HQ-CORE-RTR01` tới `PC-Guest-HCM`).

![Mô hình mạng ở chế độ Simulation](Figure/7.PNG)

---

## 6. Telemetry (Prometheus metrics)

| Metric | Loại | Label | Ý nghĩa |
|--------|------|-------|---------|
| `bank_router_cpu_usage_percent` | Gauge | `router`, `role` | Mức dùng CPU của router (%) |
| `bank_router_ram_usage_percent` | Gauge | `router`, `role` | Mức dùng RAM của router (%) |
| `bank_wan_throughput_mbps` | Gauge | `router`, `interface` | Băng thông WAN (Mbps) |
| `bank_wan_latency_ms` | Gauge | `router` | Độ trễ ping tới Core Banking Server (ms) |
| `bank_link_status` | Gauge | `router`, `interface` | Trạng thái link WAN (1 = Up, 0 = Down) |
| `bank_hsrp_state` | Gauge | `router`, `group` | Trạng thái HSRP (1 = Active, 0 = Standby) |
| `bank_ospf_neighbor_state` | Gauge | `router`, `neighbor` | Trạng thái OSPF neighbor (1 = Full, 0 = Down) |

### Mô hình sinh dữ liệu

- Exporter nạp `topology.yaml` khi khởi động (thiếu file thì thoát) và cập nhật toàn bộ metric **mỗi 5 giây**. Tên router, interface và vai trò dùng khi sinh metric hiện được **khai báo cứng trong `exporter.py`**, chưa được sinh tự động từ yaml.
- **Giờ cao điểm** (8:00–11:30 và 13:00–17:00): hệ số lưu lượng `2.5`, CPU của Core Router tăng 20%. Ngoài giờ: hệ số `0.8`.
- Giá trị được sinh ngẫu nhiên trong dải hợp lý cho từng vai trò (Core, HA-Master, HA-Standby, Single-Router).

---

## 7. Kịch bản giả lập sự cố

Các cờ sự cố nằm trong `exporter/exporter.py`:

```python
FAULT_SIMULATION = {
    "br_hcm_primary_down": False,  # Ngắt link WAN chính của Branch HCM
    "br_bd_overload": False,       # Quá tải băng thông / latency Branch BD
}
```

Đổi giá trị cờ thành `True` rồi build lại exporter:

```bash
docker compose up -d --build exporter
```

| Kịch bản | Cờ | Hiện tượng trên dashboard |
|----------|----|---------------------------|
| **Đứt link WAN chính HCM** | `br_hcm_primary_down` | `BR-HCM-RTR01`: link = 0, HSRP chuyển Standby, OSPF neighbor Down, latency 999 ms (timeout). `BR-HCM-RTR02` lên Active, tiếp quản lưu lượng (HSRP failover) |
| **Quá tải chi nhánh Bình Dương** | `br_bd_overload` | CPU 88–97%, RAM 82–91%, băng thông 8.8–9.8 Mbps (vượt 80% đường truyền 10 Mbps), latency 180–320 ms |

### Ngưỡng SLA (khai báo trong `topology.yaml`)

Các ngưỡng này hiện chỉ dùng làm tài liệu tham chiếu: exporter và dashboard chưa đọc chúng để cảnh báo. Các giá trị sự cố ở bảng trên được chọn để vượt ngưỡng.

| Chỉ số | Ngưỡng |
|--------|--------|
| Độ trễ tối đa | 100 ms |
| Băng thông sử dụng tối đa | 80% |
| CPU nghiêm trọng | 85% |
| RAM nghiêm trọng | 85% |

---

## 8. Dashboard NOC (Grafana)

Dashboard `bank-noc-dash` gồm các panel:

| Panel | Loại | Truy vấn PromQL |
|-------|------|-----------------|
| HSRP Master Status (HCM Branch) | Stat | `bank_hsrp_state{router="BR-HCM-RTR01"}` |
| HSRP Standby Status (HCM Branch) | Stat | `bank_hsrp_state{router="BR-HCM-RTR02"}` |
| WAN Link Status | Stat | `bank_link_status` |
| OSPF Neighbor State | Stat | `bank_ospf_neighbor_state` |
| WAN Throughput (Mbps) | Time series | `bank_wan_throughput_mbps` |
| Ping Latency SLA to Core Server (ms) | Time series | `bank_wan_latency_ms` |
| Router CPU Usage (%) | Gauge | `bank_router_cpu_usage_percent` |
| Router RAM Usage (%) | Gauge | `bank_router_ram_usage_percent` |

![Dashboard – WAN throughput, latency, CPU, RAM](Figure/1.PNG)

![Dashboard – trạng thái HSRP, WAN link, OSPF](Figure/3.PNG)

Prometheus xác nhận target exporter đang hoạt động (`1/1 up`):

![Prometheus Targets](Figure/5.PNG)

---

## 9. Kiểm thử nhanh

```bash
# Kiểm tra exporter đang xuất metric
curl http://localhost:8000/metrics | grep bank_

# Kiểm tra Prometheus đã scrape thành công
curl "http://localhost:9090/api/v1/query?query=up"
```

Một số truy vấn PromQL hữu ích trên Prometheus:

```promql
bank_wan_latency_ms > 100                      # Vi phạm SLA độ trễ
bank_wan_throughput_mbps > 8                   # Băng thông vượt 80% đường 10 Mbps
bank_router_cpu_usage_percent > 85             # CPU ở mức nghiêm trọng
bank_link_status == 0                          # Link WAN bị đứt
```

---

## 10. Hạn chế và hướng phát triển

- Telemetry hiện là **dữ liệu giả lập**, chưa lấy từ thiết bị thật trong Packet Tracer.
- Chưa có **alert rule** của Prometheus và chưa tích hợp **Alertmanager**; cảnh báo mới dừng ở mức hiển thị trên dashboard.
- Tên router/interface trong `exporter.py` còn khai báo cứng và chưa hoàn toàn trùng với `topology.yaml` (ví dụ interface WAN `Se0/3/0` trong exporter so với `Se0/1/0` trong yaml); cần thống nhất khi mở rộng.
- Bật/tắt kịch bản sự cố phải sửa cờ trong `exporter.py` rồi build lại image. Có thể cải tiến bằng biến môi trường hoặc endpoint HTTP để đổi cờ không cần build lại.
- Có thể mở rộng: sinh metric trực tiếp từ `topology.yaml`, thêm file `prometheus/alert.rules.yml` (latency > 100 ms, link down, HSRP đổi trạng thái) cùng Alertmanager gửi cảnh báo qua email/Telegram, SNMP exporter thật với GNS3/EVE-NG, thêm chi nhánh và kịch bản sự cố (đứt cả hai link, DDoS vào VLAN ATM/POS).

---

## 11. Tài liệu đi kèm

- `Prometheus Time Series Collection and Processing Server.pdf` – ảnh chụp trang Prometheus
- `Bank Network NOC Observability Dashboard - Dashboards - Grafana.pdf` – bản xuất dashboard Grafana
- Thư mục `Figure/` – ảnh minh họa topology, dashboard và Prometheus Targets

---

## Thành viên thực hiện

| Họ và tên | MSSV | Vai trò |
|-----------|------|---------|
| _(điền)_ | _(điền)_ | _(điền)_ |
