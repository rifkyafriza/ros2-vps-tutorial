# RE405 — Koneksi ROS2 dari PC Lokal ke VPS

Panduan ini mengajarkan cara menghubungkan node ROS2 yang berjalan di PC lokal kamu ke VPS yang sudah kamu miliki, sehingga robot kamu bisa berkomunikasi dengan layanan cloud secara real-time.

## Prasyarat
- VPS Ubuntu 24.04 dengan IP publik (sudah dimiliki mahasiswa)
- ROS2 Jazzy terinstall di PC lokal
- Akun Tailscale gratis (untuk Modul 1)
- Akun Google / Firebase gratis (untuk Modul 9)
- Pemahaman dasar ROS2 topic, node, publisher, subscriber

## Modul
| # | Judul | Metode | Tingkat |
|---|-------|--------|---------|
| 1 | [Koneksi ROS2 via Tailscale VPN](modul-01-tailscale.md) | VPN Layer-3 + FastDDS unicast | Mudah |
| 2 | [Remote Control via rosbridge WebSocket](modul-02-rosbridge.md) | WebSocket JSON bridge | Menengah |
| 3 | [Koneksi Native WAN dengan Zenoh Bridge](modul-03-zenoh.md) | Zenoh bridge/router | Menengah |
| 4 | [FastDDS Discovery Server (No VPN)](modul-04-discovery-server.md) | Native FastDDS Discovery Server | Lanjut |
| 5 | [eProsima DDS Router — WAN Bridge](modul-05-dds-router.md) | Routing TCP WAN | Lanjut |
| 6 | [Integrasi MQTT Broker untuk IoT](modul-06-mqtt-bridge.md) | MQTT Pub/Sub Bridge | Mudah |
| 7 | [Micro-ROS Agent untuk ESP32](modul-07-microros-agent.md) | UDP over Public IP | Menengah |
| 8 | [Husarnet P2P VPN (Robot-Specific)](modul-08-husarnet.md) | IPv6 Multicast P2P | Menengah |
| 9 | [Firebase Realtime Database untuk Cloud Robotics](modul-firebase-rtdb.md) | Cloud state sync + Security Rules | Menengah |

## Arsitektur Sistem

`[PC Lokal: ROS2 Node] ←→ [Jaringan / VPN / TCP] ←→ [VPS: ROS2/Bridge/Router] ←→ [Dashboard / Cloud / Firebase]`

---
*Dibuat untuk Mata Kuliah RE405 Cloud Robotics.*
