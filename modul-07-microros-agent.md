# Modul 7: Micro-ROS Agent untuk ESP32

## Tujuan Pembelajaran
- Mahasiswa mengetahui cara kerja Micro-ROS (XRCE-DDS) untuk perangkat dengan sumber daya terbatas.
- Mahasiswa mampu menjalankan `micro-ROS-Agent` di VPS untuk menerima koneksi dari internet.
- Mahasiswa mampu menghubungkan ESP32 langsung ke VPS tanpa memerlukan perantara PC/Raspberry Pi.

## Teori Singkat (10 menit)
- **Keterbatasan ESP32:** ROS2 standar (FastDDS/CycloneDDS) terlalu berat untuk memori dan CPU mikrokontroler seperti ESP32. Karena itu, dibuatlah **Micro-ROS**, versi ringan dari ROS2.
- **XRCE-DDS Protocol:** Micro-ROS menggunakan protokol klien-server yang disebut XRCE-DDS. ESP32 bertindak sebagai klien, dan membutuhkan sebuah **Agent** yang bertindak sebagai jembatan menuju jaringan DDS standar.
- **Topologi Langsung ke Cloud:** Biasanya, Agent diinstal di PC/Raspberry Pi yang menempel pada ESP32 (via kabel Serial/USB). Di modul ini, kita akan meletakkan Agent langsung di VPS Cloud. ESP32 akan menggunakan koneksi WiFi untuk menembak IP Publik VPS secara langsung melalui protokol UDP.

**Diagram Arsitektur:**
```text
Robot (ESP32 via WiFi)                   VPS Ubuntu 24.04
┌──────────────────────┐                 ┌───────────────────────────┐
│  Micro-ROS Client    │                 │   Micro-ROS Agent         │
│  (XRCE-DDS)          │      UDP        │   (Docker Container)      │
│  WiFi SSID: "Kampus" │◄───────────────►│   Port: 8888 (UDP)        │
│  Pub: /sensor_suhu   │                 │                           │
└──────────────────────┘                 └───────────────────────────┘
```

## Alat dan Prasyarat
- VPS Ubuntu 24.04 dengan IP Publik dan Docker terinstal
- Board mikrokontroler ESP32 (atau simulasi jika Anda belum memilikinya)
- Arduino IDE atau PlatformIO dengan library `micro_ros_arduino`

---

## Langkah-Langkah

### Bagian A: Instalasi Micro-ROS Agent di VPS (15 menit)

Cara termudah dan paling aman untuk menjalankan Micro-ROS Agent di Ubuntu 24.04 adalah menggunakan Docker.

**Step 1: Install Docker di VPS (Jika Belum)**
```bash
sudo apt update
sudo apt install docker.io -y
```

**Step 2: Buka Port 8888 UDP di Firewall**
Micro-ROS via WiFi menggunakan protokol UDP. Buka port 8888 (port default):
```bash
sudo ufw allow 8888/udp
```
*(Jangan lupa Security Group AWS/Azure jika menggunakan cloud provider tersebut).*

**Step 3: Jalankan Micro-ROS Agent via Docker**
Jalankan Agent menggunakan *image* Docker resmi untuk versi Jazzy:
```bash
sudo docker run -it --rm --net=host microros/micro-ros-agent:jazzy udp4 --port 8888
```
Penjelasan:
- `--net=host`: Mengikat langsung ke interface jaringan VPS agar discovery ROS2 bekerja normal.
- `udp4 --port 8888`: Menerima koneksi IPv4 UDP di port 8888.

Biarkan terminal terbuka. Anda akan melihat log dari agent yang sedang menunggu klien.

---

### Bagian B: Konfigurasi ESP32 sebagai Klien (20 menit)

Di PC Anda, buka Arduino IDE untuk memprogram ESP32.

**Step 4: Siapkan Kode Program ESP32**
Gunakan *sketch* bawaan dari library `micro_ros_arduino` (misal: contoh `micro-ros_publisher`). Ubah bagian koneksi WiFi dan Agent IP sesuai dengan VPS Anda:

```cpp
#include <micro_ros_arduino.h>
#include <stdio.h>
#include <rcl/rcl.h>
#include <rcl/error_handling.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/int32.h>

// Konfigurasi WiFi
#define WIFI_SSID "NAMA_WIFI_ANDA"
#define WIFI_PASS "PASSWORD_WIFI_ANDA"

// Konfigurasi IP VPS (Ubah ke IP Publik VPS Anda!)
#define AGENT_IP "103.xxx.xxx.xxx"
#define AGENT_PORT 8888

// ... (kode inisialisasi node, publisher, dll) ...

void setup() {
  Serial.begin(115200);
  
  // Hubungkan ESP32 langsung ke IP Publik VPS via WiFi
  set_microros_wifi_transports(WIFI_SSID, WIFI_PASS, AGENT_IP, AGENT_PORT);
  
  // (Lanjutkan setup node Micro-ROS di sini)
}
```

**Step 5: Upload ke ESP32**
Compile dan upload program ke ESP32. Buka *Serial Monitor* di Arduino IDE untuk melihat proses koneksi.

---

### Bagian C: Verifikasi Koneksi di Cloud (10 menit)

**Step 6: Cek Log Agent di VPS**
Jika ESP32 berhasil terhubung, terminal VPS Anda (yang menjalankan Docker) akan memunculkan log seperti:
```text
[XRCE-DDS] Session established: client_id: 1234, session_id: 0x81
[XRCE-DDS] Created participant: ...
[XRCE-DDS] Created publisher: ...
```

**Step 7: Cek Topik ROS2 di VPS**
Buka terminal baru di VPS (biarkan Agent tetap berjalan). Karena menggunakan `--net=host`, ROS2 di VPS otomatis melihat node ESP32.
```bash
source /opt/ros/jazzy/setup.bash
ros2 node list
ros2 topic list
```
Jika semuanya benar, topik dari ESP32 Anda akan muncul. Anda bisa langsung melakukan `ros2 topic echo` untuk melihat datanya di server VPS!

---

## Latihan Mandiri
1. Modifikasi program ESP32 untuk mendengarkan (Subscribe) topik `/led_cmd` tipe `std_msgs/Int32`. Dari VPS, cobalah publikasikan nilai `1` untuk menyalakan LED di ESP32, dan nilai `0` untuk mematikannya.
2. Coba letakkan ESP32 di jaringan operator seluler (Tethering HP), sementara VPS berada di internet publik. Apakah koneksi UDP tetap berhasil tanpa hambatan?

---

## Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| Serial ESP32 berhenti setelah "Connecting to WiFi..." | Salah password WiFi atau router menolak koneksi. | Cek pengaturan hotspot / WiFi. |
| Agent di VPS tidak mendeteksi sesi masuk | Port UDP 8888 terblokir oleh firewall VPS/Cloud. | Ulangi langkah `sudo ufw allow 8888/udp` dan periksa console *Security Group* VPS Anda. |
| Topik muncul di VPS, tapi tidak muncul di PC Lokal | Agent berjalan di LAN VPS, perlu direlay ke PC. | Kombinasikan Modul ini dengan Modul 1 (Tailscale) atau Modul 5 (DDS Router) agar topik di VPS dikirim juga ke PC Lokal Anda. |

---
*Lanjut ke [Modul 8: Husarnet P2P VPN khusus Robotika](modul-08-husarnet.md)*
