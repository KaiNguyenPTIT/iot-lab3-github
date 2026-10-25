# 📡 BÀI THỰC HÀNH SỐ 2: THU THẬP, LƯU TRỮ VÀ TIỀN XỬ LÝ DỮ LIỆU IoT

> **Học phần:** IoT và Ứng dụng (INT14149)  
> **Sinh viên thực hiện:** Nguyễn Đình Khải 
> **Mã sinh viên:** B23DCCN420

---

## 📖 1. Giới thiệu tổng quan
Dự án xây dựng một hệ thống pipeline thu thập, lưu trữ theo chuỗi thời gian (time-series) và tiền xử lý dữ liệu IoT hoàn chỉnh:
- **Tầng biên (Edge/Sensor):** Vi điều khiển ESP32 đọc dữ liệu từ cảm biến nhiệt độ/độ ẩm (DHT22) và cảm biến khoảng cách (HC-SR04) mô phỏng trên nền tảng Wokwi.
- **Tầng truyền thông (Protocol):** Giao thức MQTT qua Public Broker (`broker.hivemq.com`), QoS = 0/1.
- **Tầng lưu trữ (Time-Series DB):** InfluxDB v2.x tối ưu hóa lưu trữ dữ liệu cảm biến theo thời gian thực với chính sách lưu trữ (Retention Policy).
- **Tầng tiền xử lý (Preprocessing):** Xử lý dữ liệu thiếu (Interpolation), loại bỏ nhiễu dị biệt (Outlier Detection qua IQR), Resampling, làm mịn tín hiệu (Rolling Mean), tính toán độ biến thiên (Delta) và chuẩn hóa dữ liệu (MinMax Scaling).
- **Tầng hiển thị (Dashboard):** Web Dashboard tương tác thời gian thực xây dựng bằng Flask và Chart.js.

---

## 📁 2. Cấu trúc thư mục mã nguồn

```text
├── firmware.ino        # Mã nguồn nhúng C++ cho ESP32 chạy trên Wokwi (DHT22 + HC-SR04)
├── sensor_node.py      # Script Python giả lập node cảm biến phát dữ liệu MQTT (chạy thử độc lập)
├── collector.py        # Subscriber MQTT, validate gói tin, đo độ trễ và ghi vào InfluxDB (sensor_raw)
├── preprocess.py       # Pipeline tiền xử lý: Nội suy, Lọc Outlier IQR, Resampling, Feature Engineering
├── dashboard.py        # Ứng dụng Web Dashboard trực quan hóa dữ liệu Real-time và Processed
├── config.py           # File cấu hình tập trung (MQTT Broker, InfluxDB Token, Org, Buckets)
└── README.md           # Tài liệu hướng dẫn cài đặt và vận hành hệ thống
```

---

## 🗄️ 3. Thiết kế Schema Cơ sở dữ liệu (InfluxDB v2)

Hệ thống thiết kế tách biệt 2 Bucket dữ liệu:

### 3.1. Bucket `sensor_raw` (Dữ liệu thô - Retention: 7 ngày)
- **Measurement:** `environment`
- **Tag Set (Indexed):**
  - `device_id`: Định danh thiết bị (ví dụ: `esp32_wokwi`).
  - `location`: Vị trí đặt cảm biến (`lab_room_302`).
- **Field Set:**
  - `temperature` (Float): Nhiệt độ đo đạc (°C).
  - `humidity` (Float): Độ ẩm không khí (%).
  - `distance_cm` (Float): Khoảng cách cảm biến siêu âm (cm).
  - `latency_ms` (Float): Độ trễ truyền nhận gói tin end-to-end (ms).

### 3.2. Bucket `sensor_processed` (Dữ liệu đã tiền xử lý)
- **Measurement:** `environment_processed`
- **Tag Set:** `device_id`
- **Field Set:**
  - `temp_clean` (Float): Nhiệt độ sau khi làm sạch ngoại lai (IQR clipping).
  - `temp_rolling_mean` (Float): Nhiệt độ làm mịn bằng trượt trung bình cửa sổ 3 mẫu.
  - `temp_delta` (Float): Biến thiên nhiệt độ so với chu kỳ trước.
  - `temp_scaled` (Float): Dữ liệu nhiệt độ chuẩn hóa miền [0, 1].
  - `humidity` (Float): Độ ẩm sau khi resampling theo chu kỳ 10 giây.

---

## 🚀 4. Hướng dẫn cài đặt và chạy hệ thống

### Bước 1: Chuẩn bị môi trường Python
Yêu cầu Python 3.10 trở lên. Cài đặt các thư viện cần thiết:
```bash
pip install paho-mqtt influxdb-client pandas numpy scikit-learn flask
```

### Bước 2: Khởi động InfluxDB
- Chạy binary InfluxDB cục bộ:
  ```cmd
  influxd.exe
  ```
- Truy cập trình duyệt tại `http://localhost:8086`, tạo Org `iot_lab`, tạo 2 Bucket: `sensor_raw` và `sensor_processed`.
- Tạo All-Access API Token và cập nhật vào file `config.py` (hoặc trực tiếp trong các file script).

### Bước 3: Chạy thiết bị cảm biến (Chọn 1 trong 2 cách)
- **Cách 1 (Wokwi):** Mở dự án ESP32 trên Wokwi, dán code từ `firmware.ino` và bấm **Play Simulation**.
- **Cách 2 (Python Simulator):** Chạy node cảm biến giả lập bằng lệnh:
  ```bash
  python sensor_node.py
  ```

### Bước 4: Thu thập và lưu trữ dữ liệu vào InfluxDB
Mở một terminal mới và chạy service Collector:
```bash
python collector.py
```
*Chương trình sẽ lắng nghe tin nhắn MQTT, kiểm tra tính hợp lệ và ghi các Point vào InfluxDB.*

### Bước 5: Chạy tiến trình tiền xử lý dữ liệu
Sau khi hệ thống thu thập được một lượng dữ liệu (khoảng 1–2 phút), chạy script xử lý:
```bash
python preprocess.py
```
*Kết quả tiền xử lý sẽ được tính toán và lưu tự động vào bucket `sensor_processed`.*

### Bước 6: Khởi chạy Web Dashboard giám sát
Khởi chạy giao diện giám sát:
```bash
python dashboard.py
```
Mở trình duyệt truy cập: **`http://localhost:5000`** để theo dõi biểu đồ dữ liệu thời gian thực, độ trễ và bảng số liệu tiền xử lý.

---

## 📊 5. Thuật toán tiền xử lý áp dụng
1. **Linear Interpolation:** Tự động bù đắp dữ liệu bị khuyết thiếu dựa trên mốc thời gian liền kề.
2. **IQR Outlier Filtering:** 
   - Tính $IQR = Q_3 - Q_1$.
   - Giới hạn hợp lệ trong khoảng $[Q_1 - 1.5 \times IQR, Q_3 + 1.5 \times IQR]$.
   - Loại bỏ và cắt gọt các điểm nhiễu cảm biến bất thường (như xung nhiệt độ cao đột biến).
3. **Resampling:** Gộp trung bình theo khung thời gian cố định 10 giây để đồng nhất nhịp lấy mẫu.
4. **Feature Engineering & Normalization:**
   - **Rolling Mean:** Cửa sổ trượt 3 bước để giảm dao động tức thời.
   - **Delta ($\Delta$):** Đo đạo hàm bậc nhất của chuỗi nhiệt độ.
   - **MinMaxScaler:** Co dãn miền giá trị về thang $[0, 1]$.

---

## 🧪 6. Bổ sung Bài thực hành 3: Trích chọn đặc trưng và học từ dữ liệu IoT

Bài 3 dùng lại dữ liệu ba kênh `temperature`, `humidity`, `distance_cm` của Bài 1–2 (không viết lại firmware,
không cài lại broker/database), tạo cửa sổ 60 giây và huấn luyện mô hình dự báo **nhiệt độ trung bình của phút
tiếp theo**.

### 6.1. Nguồn dữ liệu

`du_lieu_B2.csv` là **bộ mẫu mô phỏng 720 dòng/60 phút trên lưới 5 giây**, sinh bằng
`gen_du_lieu_B2.py` theo đúng schema kế thừa của Bài 2; cột `data_source` ghi `python_sample`.
Bộ mẫu được dùng vì nhóm chưa thu đủ 20–60 phút dữ liệu Wokwi liên tục, và Wokwi cũng là môi trường
mô phỏng. Dữ liệu **không phải** bản ghi tải từ ThingsBoard và **không phải** số đo thực của cảm biến.

Khi đã có dữ liệu Wokwi: xem **`HUONG_DAN_THU_DU_LIEU_WOKWI.md`** để thu 20–60 phút, export bucket
`sensor_raw` bằng `export_influx_to_csv.py`, rồi đổi `DATA_PATH` và `DATA_SOURCE = "wokwi_mqtt_influxdb"`
trong ô 1 của notebook và chạy lại. Mọi scaler/selector/model vẫn chỉ fit trên train.

### 6.2. Tệp bổ sung

```text
├── gen_du_lieu_B2.py        # Sinh bộ mẫu mô phỏng du_lieu_B2.csv (720 dòng/60 phút, lưới 5 giây)
├── export_influx_to_csv.py  # Export bucket sensor_raw → CSV đúng schema Bài 2/3 (data_source=wokwi_mqtt_influxdb)
├── HUONG_DAN_THU_DU_LIEU_WOKWI.md  # Hướng dẫn thu 60 phút dữ liệu thật từ Wokwi → MQTT → InfluxDB
├── du_lieu_B2.csv           # Tệp đầu vào thống nhất của Bài 3 (723 dòng gốc: 3 dòng trùng + 2 giá trị ngoài miền)
├── Bai3_Sinh_vien.ipynb     # Notebook đã chạy: kiểm tra dữ liệu → cửa sổ → 21 đặc trưng → SelectKBest → Ridge
├── ket_qua_B3/              # dac_trung_train.csv, dac_trung_test.csv, du_bao_test.csv, bang_loi.csv,
│                            # xep_hang_dac_trung.csv, tom_tat.json, hinh0_kien_truc.png, hinh1..3.png
└── Bao_cao_B3.docx          # Báo cáo Word 3-4 trang
```

### 6.3. Cách chạy

```bash
pip install pandas numpy scikit-learn matplotlib notebook nbconvert
python gen_du_lieu_B2.py                      # chỉ cần khi chưa có du_lieu_B2.csv
jupyter notebook Bai3_Sinh_vien.ipynb         # Run All
# hoặc chạy không giao diện:
jupyter nbconvert --to notebook --execute --inplace Bai3_Sinh_vien.ipynb
```

Notebook tự tạo `ket_qua_B3/` và xuất đủ bảng, hình, `tom_tat.json`.

**Thu dữ liệu Wokwi thật (tuỳ chọn, thay bộ mẫu):**

```bash
python collector.py                                  # terminal 1, ghi lại SESSION_ID in ra
# mở Wokwi, bấm Play, kéo slider theo kịch bản trong HUONG_DAN_THU_DU_LIEU_WOKWI.md
# sau 20-60 phút: Ctrl+C collector
python export_influx_to_csv.py --list-sessions
python export_influx_to_csv.py --session S20261025_1432 --out du_lieu_wokwi.csv
# sửa ô 1 notebook: DATA_PATH = Path("du_lieu_wokwi.csv"); DATA_SOURCE = "wokwi_mqtt_influxdb"
```

Script export đổi `_time` UTC → `+07:00`, điền `data_source = wokwi_mqtt_influxdb`, để trống `rssi`
(Wokwi không mô phỏng RSSI) và **không ghi đè** `du_lieu_B2.csv` nếu chưa có `--force`.

### 6.4. Kết quả chính (bộ mẫu mô phỏng 720 dòng)

| Cấu hình | MAE (°C) | RMSE (°C) |
|---|---|---|
| **Ridge_6** (SelectKBest k=6 → Ridge) | **1,527** | **1,784** |
| Persistence (`temperature_mean`) | 1,623 | 2,073 |
| Ridge_21 (đủ 21 đặc trưng) | 1,681 | 2,009 |
| Mean_train | 2,979 | 3,407 |

- 6 đặc trưng được chọn: `temperature_mean`, `temperature_max`, `temperature_delta`, `temperature_slope`,
  `humidity_delta`, `humidity_slope`.
- Chia tập: 504 mẫu train / 216 mẫu test → **81 cặp train, 33 cặp test**; cửa sổ 60 giây (12 mẫu), bước trượt 30 giây.
- Dự báo một cửa sổ: `temperature_forecast_next60s = 31,90 °C` (thực tế 32,44 °C).

### 6.5. Mở rộng dashboard

`dashboard.py` có thêm thẻ **“Dự báo TB 60 s tới”** và endpoint `GET /api/forecast`, đọc `ket_qua_B3/tom_tat.json`.
Giá trị này là **trung bình tương lai**, hiển thị tách khỏi telemetry `temperature` hiện tại (không gửi dưới tên
telemetry gốc).

> Ghi chú cấu hình: token InfluxDB trong `config.py`, `preprocess.py`, `dashboard.py` đã đồng bộ với token mà
> `collector.py` đang dùng. Không commit token thật khi nộp bài — nên đọc từ biến môi trường.


