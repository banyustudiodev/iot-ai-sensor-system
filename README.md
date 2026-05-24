# IoT Environmental Sensor Monitoring with PHP, MySQL, and Python AI Model

Project ini adalah aplikasi web sederhana untuk simulasi data sensor lingkungan berbasis **PHP**, **MySQL**, dan **Python lokal**. Aplikasi ini dapat melakukan generate data sensor secara otomatis, menyimpan data ke database, memanggil model AI dari file `.pkl`, lalu menampilkan hasil prediksi pada dashboard web.

Project ini tidak menggunakan Flask. PHP memanggil script Python lokal melalui `shell_exec()` untuk membaca file model hasil Google Colab.

## 1. Fitur Aplikasi

Aplikasi ini memiliki fitur utama sebagai berikut:

1. Simulasi data sensor lingkungan.
2. Penyimpanan data sensor ke MySQL.
3. Dashboard monitoring data sensor terbaru.
4. Riwayat data sensor.
5. Prediksi status sensor menggunakan model AI Python.
6. Realtime generate dan prediksi otomatis setiap 30 detik.
7. Penyimpanan hasil prediksi ke database.

## 2. Teknologi yang Digunakan

Project ini menggunakan:

- PHP Native
- MySQL
- HTML
- CSS
- JavaScript
- Python
- scikit-learn
- pandas
- numpy
- joblib
- XAMPP

## 3. Arsitektur Sistem

Alur sistem:

```text
Dashboard PHP
      ↓
Generate Data Sensor
      ↓
Simpan ke MySQL
      ↓
PHP memanggil script Python
      ↓
Python membaca model.pkl, scaler.pkl, label_encoder.pkl
      ↓
Python mengembalikan hasil prediksi JSON
      ↓
PHP menyimpan hasil prediksi ke MySQL
      ↓
Dashboard menampilkan data terbaru dan prediksi
```

## 4. Struktur Folder Project

Simpan project di dalam folder `htdocs` XAMPP.

Contoh lokasi untuk macOS:

```text
/Applications/XAMPP/htdocs/iot-ai-sensor-system
```

Struktur folder:

```text
iot-ai-sensor-system/
│
├── config/
│   └── database.php
│
├── api/
│   ├── insert_sensor.php
│   ├── predict_sensor.php
│   ├── get_latest_sensor.php
│   ├── get_sensor_history.php
│   └── generate_and_predict.php
│
├── simulation/
│   └── generate_sensor.php
│
├── ai_model/
│   ├── model.pkl
│   ├── scaler.pkl
│   ├── label_encoder.pkl
│   ├── predict.py
│   └── requirements.txt
│
├── assets/
│   └── style.css
│
├── venv/
├── index.php
└── database.sql
```

## 5. Persiapan Database

Buka phpMyAdmin:

```text
http://localhost/phpmyadmin
```

Buat database dan tabel dengan menjalankan file `database.sql`.

Isi file `database.sql`:

```sql
CREATE DATABASE IF NOT EXISTS iot_sensor_db;

USE iot_sensor_db;

DROP TABLE IF EXISTS sensor_data;

CREATE TABLE sensor_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    device_id VARCHAR(50) NOT NULL,
    temperature DECIMAL(6,2) NOT NULL,
    humidity DECIMAL(6,2) NOT NULL,
    co DECIMAL(10,6) NOT NULL,
    lpg DECIMAL(10,6) NOT NULL,
    smoke DECIMAL(10,6) NOT NULL,
    light_intensity INT NOT NULL,
    motion_status TINYINT NOT NULL,
    prediction_label ENUM('NORMAL', 'WARNING', 'DANGER') NULL,
    prediction_score DECIMAL(6,4) NULL,
    prediction_reason TEXT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 6. Konfigurasi Koneksi Database

Buat file:

```text
config/database.php
```

Isi file:

```php
<?php

$host = "localhost";
$username = "root";
$password = "";
$database = "iot_sensor_db";

$conn = new mysqli($host, $username, $password, $database);

if ($conn->connect_error) {
    http_response_code(500);
    die(json_encode([
        "status" => "error",
        "message" => "Koneksi database gagal",
        "error" => $conn->connect_error
    ]));
}

$conn->set_charset("utf8mb4");

?>
```

Catatan:

Untuk XAMPP, konfigurasi default biasanya:

```text
username: root
password: kosong
```

## 7. Persiapan File Model AI

Kode Google Colab: https://colab.research.google.com/drive/1O4S76NzWAViOwUK4Ggd08v-CmWklpuCo?usp=sharing
Akses Model: https://drive.google.com/drive/folders/1BPFgM3Yo95k-00X2Nu8xVjUAZMccM8iH?usp=sharing

Masukkan tiga file model dari Google Colab ke folder:

```text
ai_model/
```

File yang wajib ada:

```text
model.pkl
scaler.pkl
label_encoder.pkl
```

Pastikan nama file sesuai. Jika file masih bernama `scaler (1).pkl`, ubah menjadi:

```text
scaler.pkl
```

## 8. Membuat Virtual Environment Python

Masuk ke folder project:

```bash
cd /Applications/XAMPP/htdocs/iot-ai-sensor-system
```
Catatan: sesuaikan dengan lokasi folder XAMPP kalian di Windows, /Applications/XAMPP/htdocs/iot-ai-sensor-system merupakan alamat folder di Macbook.

Buat virtual environment:

```bash
python3 -m venv venv
```

Aktifkan virtual environment:

```bash
source venv/bin/activate
```

Install library Python:

```bash
pip install pandas numpy scikit-learn joblib
```

Jika model dibuat menggunakan scikit-learn versi tertentu, gunakan versi yang sama. Contoh:

```bash
pip install scikit-learn==1.6.1 pandas numpy joblib
```

Simpan daftar library:

```bash
pip freeze > ai_model/requirements.txt
```

## 9. Script Python untuk Prediksi

Buat file:

```text
ai_model/predict.py
```

Isi file:

```python
import sys
import json
import os
import joblib
import pandas as pd

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

MODEL_PATH = os.path.join(BASE_DIR, "model.pkl")
SCALER_PATH = os.path.join(BASE_DIR, "scaler.pkl")
LABEL_ENCODER_PATH = os.path.join(BASE_DIR, "label_encoder.pkl")

FEATURE_COLUMNS = [
    "temperature",
    "humidity",
    "co",
    "lpg",
    "smoke",
    "light_intensity",
    "motion_status"
]

def main():
    try:
        if len(sys.argv) < 2:
            raise ValueError("Input JSON tidak ditemukan.")

        input_json = sys.argv[1]
        data = json.loads(input_json)

        input_data = pd.DataFrame([{
            "temperature": float(data["temperature"]),
            "humidity": float(data["humidity"]),
            "co": float(data["co"]),
            "lpg": float(data["lpg"]),
            "smoke": float(data["smoke"]),
            "light_intensity": float(data["light_intensity"]),
            "motion_status": int(data["motion_status"])
        }], columns=FEATURE_COLUMNS)

        model = joblib.load(MODEL_PATH)
        scaler = joblib.load(SCALER_PATH)
        label_encoder = joblib.load(LABEL_ENCODER_PATH)

        input_scaled = scaler.transform(input_data)

        prediction_code = model.predict(input_scaled)[0]
        prediction_proba = model.predict_proba(input_scaled)[0]

        prediction_label = label_encoder.inverse_transform([prediction_code])[0]
        prediction_score = float(max(prediction_proba))

        result = {
            "status": "success",
            "prediction_label": prediction_label,
            "prediction_score": round(prediction_score, 4),
            "prediction_code": int(prediction_code)
        }

        print(json.dumps(result))

    except Exception as e:
        result = {
            "status": "error",
            "message": str(e)
        }

        print(json.dumps(result))


if __name__ == "__main__":
    main()
```

## 10. Tes Script Python Manual

Jalankan:

```bash
cd /Applications/XAMPP/htdocs/iot-ai-sensor-system
source venv/bin/activate
```

Tes prediksi:

```bash
python ai_model/predict.py '{"temperature":36.5,"humidity":70,"co":0.005,"lpg":0.008,"smoke":0.45,"light_intensity":700,"motion_status":1}'
```

Contoh output:

```json
{"status": "success", "prediction_label": "WARNING", "prediction_score": 0.87, "prediction_code": 2}
```

Jika output JSON muncul, script Python sudah berjalan.

## 11. API Insert Data Sensor

Buat file:

```text
api/insert_sensor.php
```

Isi file:

```php
<?php

header("Content-Type: application/json");
require_once "../config/database.php";

if ($_SERVER["REQUEST_METHOD"] !== "POST") {
    http_response_code(405);
    echo json_encode([
        "status" => "error",
        "message" => "Method harus POST"
    ]);
    exit;
}

$device_id = $_POST["device_id"] ?? null;
$temperature = $_POST["temperature"] ?? null;
$humidity = $_POST["humidity"] ?? null;
$co = $_POST["co"] ?? null;
$lpg = $_POST["lpg"] ?? null;
$smoke = $_POST["smoke"] ?? null;
$light_intensity = $_POST["light_intensity"] ?? null;
$motion_status = $_POST["motion_status"] ?? null;

if (
    $device_id === null ||
    $temperature === null ||
    $humidity === null ||
    $co === null ||
    $lpg === null ||
    $smoke === null ||
    $light_intensity === null ||
    $motion_status === null
) {
    http_response_code(400);
    echo json_encode([
        "status" => "error",
        "message" => "Data sensor belum lengkap"
    ]);
    exit;
}

$device_id = trim($device_id);
$temperature = (float) $temperature;
$humidity = (float) $humidity;
$co = (float) $co;
$lpg = (float) $lpg;
$smoke = (float) $smoke;
$light_intensity = (int) $light_intensity;
$motion_status = (int) $motion_status;

$stmt = $conn->prepare("
    INSERT INTO sensor_data 
    (
        device_id,
        temperature,
        humidity,
        co,
        lpg,
        smoke,
        light_intensity,
        motion_status
    )
    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
");

$stmt->bind_param(
    "sdddddii",
    $device_id,
    $temperature,
    $humidity,
    $co,
    $lpg,
    $smoke,
    $light_intensity,
    $motion_status
);

if ($stmt->execute()) {
    echo json_encode([
        "status" => "success",
        "message" => "Data sensor berhasil disimpan",
        "insert_id" => $stmt->insert_id
    ]);
} else {
    http_response_code(500);
    echo json_encode([
        "status" => "error",
        "message" => "Gagal menyimpan data sensor",
        "error" => $stmt->error
    ]);
}

$stmt->close();
$conn->close();

?>
```

## 12. API Prediksi dengan Model Python

Buat file:

```text
api/predict_sensor.php
```

Isi file:

```php
<?php

header("Content-Type: application/json");
require_once "../config/database.php";

$sql = "SELECT * FROM sensor_data ORDER BY id DESC LIMIT 1";
$result = $conn->query($sql);

if (!$result || $result->num_rows == 0) {
    echo json_encode([
        "status" => "error",
        "message" => "Belum ada data sensor"
    ]);
    exit;
}

$sensor = $result->fetch_assoc();

$inputData = [
    "temperature" => (float) $sensor["temperature"],
    "humidity" => (float) $sensor["humidity"],
    "co" => (float) $sensor["co"],
    "lpg" => (float) $sensor["lpg"],
    "smoke" => (float) $sensor["smoke"],
    "light_intensity" => (float) $sensor["light_intensity"],
    "motion_status" => (int) $sensor["motion_status"]
];

$jsonInput = json_encode($inputData);

$projectRoot = realpath(__DIR__ . "/..");

$pythonPath = $projectRoot . "/venv/bin/python";
$scriptPath = $projectRoot . "/ai_model/predict.py";

if (!file_exists($pythonPath)) {
    echo json_encode([
        "status" => "error",
        "message" => "Python virtual environment tidak ditemukan di: " . $pythonPath
    ]);
    exit;
}

if (!file_exists($scriptPath)) {
    echo json_encode([
        "status" => "error",
        "message" => "File predict.py tidak ditemukan di: " . $scriptPath
    ]);
    exit;
}

$command = escapeshellcmd($pythonPath) . " " .
           escapeshellarg($scriptPath) . " " .
           escapeshellarg($jsonInput);

$output = shell_exec($command);

if ($output === null || trim($output) === "") {
    echo json_encode([
        "status" => "error",
        "message" => "Python script tidak menghasilkan output."
    ]);
    exit;
}

$prediction = json_decode($output, true);

if (!$prediction) {
    echo json_encode([
        "status" => "error",
        "message" => "Output Python tidak valid.",
        "raw_output" => $output
    ]);
    exit;
}

if ($prediction["status"] !== "success") {
    echo json_encode($prediction);
    exit;
}

$predictionLabel = $prediction["prediction_label"];
$predictionScore = (float) $prediction["prediction_score"];
$sensorId = (int) $sensor["id"];

$predictionReason = "Model AI memprediksi status " . $predictionLabel .
    " berdasarkan fitur temperature, humidity, CO, LPG, smoke, light intensity, dan motion status.";

$stmt = $conn->prepare("
    UPDATE sensor_data
    SET prediction_label = ?, prediction_score = ?, prediction_reason = ?
    WHERE id = ?
");

$stmt->bind_param("sdsi", $predictionLabel, $predictionScore, $predictionReason, $sensorId);

if ($stmt->execute()) {
    echo json_encode([
        "status" => "success",
        "message" => "Prediksi berhasil diproses menggunakan model AI.",
        "prediction_label" => $predictionLabel,
        "prediction_score" => $predictionScore,
        "prediction_reason" => $predictionReason
    ]);
} else {
    echo json_encode([
        "status" => "error",
        "message" => "Gagal menyimpan hasil prediksi ke database.",
        "error" => $stmt->error
    ]);
}

$stmt->close();
$conn->close();

?>
```

## 13. API Generate dan Prediksi Realtime

Buat file:

```text
api/generate_and_predict.php
```

Isi file:

```php
<?php

header("Content-Type: application/json");
require_once "../config/database.php";

$device_id = "IOT-DEVICE-001";

$temperature = rand(250, 420) / 10;
$humidity = rand(300, 900) / 10;
$co = rand(100, 900) / 100000;
$lpg = rand(100, 1200) / 100000;
$smoke = rand(100, 900) / 1000;
$light_intensity = rand(0, 1000);
$motion_status = rand(0, 1);

$stmt = $conn->prepare("
    INSERT INTO sensor_data
    (
        device_id,
        temperature,
        humidity,
        co,
        lpg,
        smoke,
        light_intensity,
        motion_status
    )
    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
");

$stmt->bind_param(
    "sdddddii",
    $device_id,
    $temperature,
    $humidity,
    $co,
    $lpg,
    $smoke,
    $light_intensity,
    $motion_status
);

if (!$stmt->execute()) {
    echo json_encode([
        "status" => "error",
        "message" => "Gagal menyimpan data sensor",
        "error" => $stmt->error
    ]);
    exit;
}

$sensor_id = $stmt->insert_id;
$stmt->close();

$inputData = [
    "temperature" => (float) $temperature,
    "humidity" => (float) $humidity,
    "co" => (float) $co,
    "lpg" => (float) $lpg,
    "smoke" => (float) $smoke,
    "light_intensity" => (float) $light_intensity,
    "motion_status" => (int) $motion_status
];

$jsonInput = json_encode($inputData);

$projectRoot = realpath(__DIR__ . "/..");

$pythonPath = $projectRoot . "/venv/bin/python";
$scriptPath = $projectRoot . "/ai_model/predict.py";

$command = escapeshellcmd($pythonPath) . " " .
           escapeshellarg($scriptPath) . " " .
           escapeshellarg($jsonInput);

$output = shell_exec($command);

if ($output === null || trim($output) === "") {
    echo json_encode([
        "status" => "error",
        "message" => "Python script tidak menghasilkan output"
    ]);
    exit;
}

$prediction = json_decode($output, true);

if (!$prediction || $prediction["status"] !== "success") {
    echo json_encode([
        "status" => "error",
        "message" => "Prediksi gagal",
        "raw_output" => $output
    ]);
    exit;
}

$prediction_label = $prediction["prediction_label"];
$prediction_score = (float) $prediction["prediction_score"];

$prediction_reason = "Model AI memprediksi status " . $prediction_label .
    " berdasarkan fitur temperature, humidity, CO, LPG, smoke, light intensity, dan motion status.";

$updateStmt = $conn->prepare("
    UPDATE sensor_data
    SET prediction_label = ?, prediction_score = ?, prediction_reason = ?
    WHERE id = ?
");

$updateStmt->bind_param(
    "sdsi",
    $prediction_label,
    $prediction_score,
    $prediction_reason,
    $sensor_id
);

if (!$updateStmt->execute()) {
    echo json_encode([
        "status" => "error",
        "message" => "Gagal menyimpan hasil prediksi",
        "error" => $updateStmt->error
    ]);
    exit;
}

$updateStmt->close();

$finalStmt = $conn->prepare("
    SELECT * FROM sensor_data
    WHERE id = ?
    LIMIT 1
");

$finalStmt->bind_param("i", $sensor_id);
$finalStmt->execute();

$result = $finalStmt->get_result();
$finalData = $result->fetch_assoc();

$finalStmt->close();
$conn->close();

echo json_encode([
    "status" => "success",
    "message" => "Data sensor berhasil dibuat dan diprediksi otomatis",
    "data" => $finalData
]);

?>
```

## 14. API Riwayat Data Sensor

Buat file:

```text
api/get_sensor_history.php
```

Isi file:

```php
<?php

header("Content-Type: application/json");
require_once "../config/database.php";

$limit = $_GET["limit"] ?? 20;
$limit = (int) $limit;

if ($limit <= 0 || $limit > 100) {
    $limit = 20;
}

$stmt = $conn->prepare("
    SELECT * FROM sensor_data
    ORDER BY id DESC
    LIMIT ?
");

$stmt->bind_param("i", $limit);
$stmt->execute();

$result = $stmt->get_result();

$data = [];

while ($row = $result->fetch_assoc()) {
    $data[] = $row;
}

echo json_encode([
    "status" => "success",
    "data" => $data
]);

$stmt->close();
$conn->close();

?>
```

## 15. Simulasi Sensor Manual

Buat file:

```text
simulation/generate_sensor.php
```

Isi file:

```php
<?php

header("Content-Type: application/json");

$device_id = "IOT-DEVICE-001";

$temperature = rand(250, 420) / 10;
$humidity = rand(300, 900) / 10;
$co = rand(100, 900) / 100000;
$lpg = rand(100, 1200) / 100000;
$smoke = rand(100, 900) / 1000;
$light_intensity = rand(0, 1000);
$motion_status = rand(0, 1);

$url = "http://localhost/iot-ai-sensor-system/api/insert_sensor.php";

$data = [
    "device_id" => $device_id,
    "temperature" => $temperature,
    "humidity" => $humidity,
    "co" => $co,
    "lpg" => $lpg,
    "smoke" => $smoke,
    "light_intensity" => $light_intensity,
    "motion_status" => $motion_status
];

$options = [
    "http" => [
        "header" => "Content-Type: application/x-www-form-urlencoded\r\n",
        "method" => "POST",
        "content" => http_build_query($data),
        "timeout" => 10
    ]
];

$context = stream_context_create($options);
$response = file_get_contents($url, false, $context);

if ($response === false) {
    http_response_code(500);
    echo json_encode([
        "status" => "error",
        "message" => "Gagal mengirim data sensor ke API"
    ]);
    exit;
}

echo $response;

?>
```

## 16. Dashboard Web

Dashboard utama berada pada file:

```text
index.php
```

Fungsi utama dashboard:

1. Menampilkan data sensor terbaru.
2. Menampilkan riwayat data sensor.
3. Menjalankan generate sensor manual.
4. Menjalankan prediksi manual.
5. Menjalankan generate dan prediksi otomatis.
6. Menjalankan realtime setiap 30 detik.

## 17. Cara Menjalankan Aplikasi

Aktifkan XAMPP:

```text
Apache: Start
MySQL: Start
```

Pastikan database sudah dibuat melalui phpMyAdmin.

Buka aplikasi:

```text
http://localhost/iot-ai-sensor-system/
```

## 18. Urutan Penggunaan

Untuk penggunaan manual:

```text
1. Klik Generate Data Sensor.
2. Klik Prediksi dengan AI.
3. Hasil prediksi muncul di dashboard.
```

Untuk penggunaan otomatis:

```text
1. Klik Start Realtime 30 Detik.
2. Sistem akan generate data sensor baru.
3. Sistem langsung menjalankan prediksi AI.
4. Dashboard diperbarui otomatis.
5. Proses diulang setiap 30 detik.
```

## 19. Troubleshooting

### 19.1 Python virtual environment tidak ditemukan

Pesan error:

```text
Python virtual environment tidak ditemukan
```

Solusi:

Pastikan folder `venv` berada di root project:

```text
iot-ai-sensor-system/venv/
```

Jika belum ada, buat ulang:

```bash
python3 -m venv venv
source venv/bin/activate
pip install pandas numpy scikit-learn joblib
```

### 19.2 File predict.py tidak ditemukan

Pesan error:

```text
File predict.py tidak ditemukan
```

Solusi:

Pastikan file berada di:

```text
ai_model/predict.py
```

### 19.3 File model tidak ditemukan

Pastikan tiga file ini ada di folder `ai_model`:

```text
model.pkl
scaler.pkl
label_encoder.pkl
```

### 19.4 Output Python tidak valid

Pesan error:

```text
Output Python tidak valid
```

Solusi:

Tes script Python secara manual:

```bash
source venv/bin/activate
python ai_model/predict.py '{"temperature":36.5,"humidity":70,"co":0.005,"lpg":0.008,"smoke":0.45,"light_intensity":700,"motion_status":1}'
```

Jika error muncul, perbaiki dependency Python atau cek kompatibilitas versi scikit-learn.

### 19.5 shell_exec tidak berjalan

Jika PHP tidak bisa menjalankan Python, cek apakah `shell_exec()` aktif.

Buat file `check_shell.php`:

```php
<?php
echo shell_exec("whoami");
?>
```

Buka melalui browser:

```text
http://localhost/iot-ai-sensor-system/check_shell.php
```

Jika tidak ada output, kemungkinan `shell_exec()` dinonaktifkan di konfigurasi PHP.

### 19.6 Permission error pada macOS

Jika Python tidak bisa dipanggil, jalankan:

```bash
chmod +x venv/bin/python
chmod +x ai_model/predict.py
```

## 20. Endpoint API

Daftar endpoint:

| Endpoint | Method | Fungsi |
|---|---|---|
| `/api/insert_sensor.php` | POST | Menyimpan data sensor |
| `/api/predict_sensor.php` | GET | Memprediksi data sensor terbaru |
| `/api/get_sensor_history.php` | GET | Mengambil riwayat data sensor |
| `/api/generate_and_predict.php` | GET | Generate data sensor dan prediksi otomatis |
| `/simulation/generate_sensor.php` | GET | Generate data sensor manual |

## 21. Catatan Penting

Project ini menggunakan model AI yang sudah dibuat sebelumnya. Dokumentasi ini tidak membahas proses training model di Google Colab.

Aplikasi ini berfokus pada:

1. Pembuatan web PHP MySQL.
2. Penyimpanan data sensor.
3. Integrasi PHP dengan Python lokal.
4. Pembacaan file model `.pkl`.
5. Prediksi otomatis pada dashboard web.

## 22. Ringkasan Alur Kerja

```text
User membuka dashboard
      ↓
User klik Start Realtime
      ↓
JavaScript memanggil api/generate_and_predict.php setiap 30 detik
      ↓
PHP membuat data sensor simulasi
      ↓
PHP menyimpan data ke MySQL
      ↓
PHP memanggil ai_model/predict.py
      ↓
Python membaca model.pkl, scaler.pkl, label_encoder.pkl
      ↓
Python menghasilkan prediksi
      ↓
PHP menyimpan hasil prediksi
      ↓
Dashboard diperbarui otomatis
```

## 23. Status Project

Project ini merupakan prototipe pembelajaran untuk integrasi:

```text
PHP Web Application
MySQL Database
Python AI Inference
IoT Sensor Simulation
Realtime Dashboard
```


