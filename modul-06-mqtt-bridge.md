# Modul 6: Integrasi MQTT Broker untuk IoT

## Tujuan Pembelajaran
- Mahasiswa mampu menghubungkan ekosistem ROS2 dengan ekosistem IoT standar (MQTT).
- Mahasiswa mampu menginstal dan mengonfigurasi MQTT Broker (Mosquitto) di VPS.
- Mahasiswa mampu merutekan topik ROS2 agar bisa dibaca oleh aplikasi web/Node-RED via MQTT.

## Teori Singkat (10 menit)
- **MQTT (Message Queuing Telemetry Transport):** Protokol messaging standar de-facto untuk IoT. Sangat ringan, menggunakan pola Publish/Subscribe, dan bekerja sangat baik di jaringan internet publik (WAN) yang lambat.
- **Mengapa MQTT untuk ROS2?** ROS2 DDS terkadang terlalu berat untuk mikrokontroler sederhana atau aplikasi dashboard web. Dengan memasang jembatan (bridge), robot dapat menerbitkan data ke ROS2, bridge akan menerjemahkannya ke MQTT, dan mengirimkannya ke VPS di cloud.
- **Arsitektur:** Kita akan menginstal Mosquitto Broker di VPS. Di PC Robot, kita akan menginstal `mqtt_client` (ROS2 package) yang bertugas menghubungkan dunia ROS2 dengan dunia MQTT.

**Diagram Arsitektur:**
```text
PC Robot (ROS2)                         VPS Ubuntu 24.04
┌──────────────────────┐                ┌──────────────────────┐
│  Node: talker        │                │                      │
│        │ /chatter    │      TCP       │  Mosquitto Broker    │
│        ▼             │ ◄────────────► │  Port: 1883 (TCP)    │
│  Node: mqtt_client   │   MQTT Pub/Sub │                      │
└──────────────────────┘                └──────────────────────┘
                                                  ▲
                                                  │
                                        ┌─────────┴────────────┐
                                        │ Aplikasi Web / IoT   │
                                        │ (Node-RED, MQTT.js)  │
                                        └──────────────────────┘
```

## Alat dan Prasyarat
- VPS Ubuntu 24.04 dengan IP Publik
- PC Lokal dengan ROS2 Jazzy terinstal
- Aplikasi MQTT Client (misal: MQTT Explorer) di laptop Anda untuk pengujian

---

## Langkah-Langkah

### Bagian A: Instalasi Mosquitto Broker di VPS (15 menit)

**Step 1: Install Mosquitto**
Buka SSH ke VPS Anda dan install Mosquitto broker beserta client-nya:
```bash
sudo apt update
sudo apt install mosquitto mosquitto-clients -y
```

**Step 2: Konfigurasi Akses Publik (Tanpa Autentikasi untuk Testing)**
Secara default, Mosquitto di Ubuntu hanya mengizinkan koneksi dari `localhost` (untuk keamanan). Kita perlu mengizinkan koneksi dari luar.
Buat/edit file konfigurasi tambahan:
```bash
sudo nano /etc/mosquitto/conf.d/default.conf
```
Isi dengan dua baris berikut:
```text
listener 1883
allow_anonymous true
```
Simpan dan keluar (Ctrl+O, Enter, Ctrl+X).

**Step 3: Restart Mosquitto dan Buka Firewall**
Restart layanan agar konfigurasi baru terbaca:
```bash
sudo systemctl restart mosquitto
```
Buka port 1883 di firewall VPS:
```bash
sudo ufw allow 1883/tcp
```
*(Ingat juga untuk membuka port ini di Security Group cloud provider Anda).*

---

### Bagian B: Setup MQTT Bridge di PC Lokal (20 menit)

Di PC Robot, kita akan menginstal paket `mqtt_client`.

**Step 4: Install ROS2 MQTT Client**
Buka terminal di PC Lokal Anda:
```bash
sudo apt update
sudo apt install ros-jazzy-mqtt-client
```

**Step 5: Buat File Konfigurasi Mapping Topik**
Siapkan file bernama `mqtt_bridge_params.yaml`. File ini sudah tersedia di dalam folder `configs/mqtt_bridge_params.yaml` pada repositori tutorial ini, Anda cukup menyalinnya.

Isi file tersebut menentukan topik apa yang dikirim ke VPS:
```yaml
mqtt_client:
  ros__parameters:
    broker:
      host: "<VPS_PUBLIC_IP>"
      port: 1883
    bridge:
      # Dari ROS2 ke MQTT (publish dari robot ke VPS)
      ros2_to_mqtt:
        - ros_topic: "/chatter"
          mqtt_topic: "robot/chatter"
      # Dari MQTT ke ROS2 (subscribe dari VPS untuk dikirim ke robot)
      mqtt_to_ros2:
        - mqtt_topic: "robot/cmd_vel"
          ros_topic: "/cmd_vel"
```
*Jangan lupa ubah `<VPS_PUBLIC_IP>` menjadi IP VPS Anda sesungguhnya.*

**Step 6: Jalankan Node MQTT Bridge**
Jalankan node bridge dengan memuat konfigurasi parameter YAML tadi:
```bash
source /opt/ros/jazzy/setup.bash
ros2 run mqtt_client mqtt_client --ros-args --params-file mqtt_bridge_params.yaml
```

---

### Bagian C: Verifikasi Komunikasi (15 menit)

**Step 7: Kirim Data dari ROS2 ke MQTT**
Buka terminal baru di PC Lokal, jalankan node talker standar:
```bash
ros2 run demo_nodes_cpp talker
```

Sekarang, buka terminal di VPS (atau gunakan aplikasi MQTT Explorer di laptop Anda). Subscribe ke topik MQTT `robot/chatter`:
```bash
mosquitto_sub -h localhost -t "robot/chatter" -v
```
Anda akan melihat string JSON mentah yang berisi data pesan `"Hello World: X"`.

**Step 8: Kirim Perintah dari MQTT ke ROS2**
Biarkan MQTT Bridge tetap berjalan di PC Lokal. Buka terminal baru di PC Lokal, dan pantau topik ROS2 `/cmd_vel`:
```bash
ros2 topic echo /cmd_vel
```

Di terminal VPS, publikasikan pesan MQTT berformat JSON yang merepresentasikan pesan Twist:
```bash
mosquitto_pub -h localhost -t "robot/cmd_vel" -m '{"linear": {"x": 1.0, "y": 0.0, "z": 0.0}, "angular": {"x": 0.0, "y": 0.0, "z": 0.5}}'
```

Anda akan melihat terminal PC Lokal bereaksi dan mencetak pesan tersebut! Robot sekarang bisa dikontrol murni melalui perintah MQTT.

---

## Latihan Mandiri
1. Sambungkan dashboard **Node-RED** ke Mosquitto Broker VPS Anda, dan buat tombol UI yang menerbitkan perintah maju/mundur ke `robot/cmd_vel`.
2. Ubah konfigurasi `mqtt_bridge_params.yaml` agar meneruskan topik posisi robot dari `/turtle1/pose` ke MQTT `robot/pose`.
3. Tambahkan autentikasi *username* dan *password* pada Mosquitto di VPS demi keamanan. Jangan lupa ubah juga file `mqtt_bridge_params.yaml` agar menyertakan kredensial login.

---

## Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| `Connection Refused` pada mqtt_client | Port 1883 belum terbuka di VPS. | Cek `sudo ufw status`. Pastikan port 1883 TCP terbuka dan Security Group diizinkan. |
| Koneksi sukses, tapi data tidak masuk | Nama topik di YAML tidak cocok. | Pastikan `ros_topic` sama persis dengan yang ada di `ros2 topic list`. |
| Mqtt client error "Cannot parse JSON" | Tipe message ROS2 terlalu kompleks. | Tidak semua tipe pesan ROS2 didukung otomatis. Pesan dasar seperti `std_msgs` dan `geometry_msgs` terdukung penuh. |

---
*Kembali ke [Halaman Utama (README)](README.md)*
