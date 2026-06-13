# Modul 8: Husarnet P2P VPN (Robot-Specific)

## Tujuan Pembelajaran
- Mahasiswa mengetahui keunggulan Husarnet dibandingkan VPN konvensional untuk robotika.
- Mahasiswa mampu mengatur jaringan IPv6 peer-to-peer menggunakan Husarnet.
- Mahasiswa mampu mengonfigurasi CycloneDDS agar bekerja secara *plug-and-play* pada jaringan Husarnet.

## Teori Singkat (10 menit)
- **VPN untuk Robotika:** Sebagian besar VPN (seperti Tailscale atau ZeroTier) dirancang untuk aplikasi IT umum dan kadang memblokir lalu lintas *multicast* UDP yang sangat dibutuhkan oleh ROS2 DDS. Hal ini membuat kita harus melakukan konfigurasi XML profil yang rumit (seperti di Modul 1).
- **Apa itu Husarnet?** Husarnet adalah VPN Peer-to-Peer berbasis IPv6 yang dirancang khusus untuk robotika dan mikrokontroler. Husarnet mendukung lalu lintas multicast secara bawaan!
- **Kelebihan Husarnet:** Jika menggunakan CycloneDDS dan Husarnet, ROS2 akan berjalan persis seperti di LAN. Kita tidak perlu mendefinisikan IP tujuan satu per satu dalam file XML.

**Diagram Arsitektur:**
```text
PC Robot (ROS2)                  VPS Ubuntu 24.04
┌─────────────────────┐          ┌─────────────────────┐
│  ┌───────────────┐  │ IPv6 P2P │  ┌───────────────┐  │
│  │  ROS2 Node    │  │◄────────►│  │  ROS2 Node    │  │
│  │  (talker)     │  │ Multicast│  │  (listener)   │  │
│  └───────────────┘  │ Dukung   │  └───────────────┘  │
│  CycloneDDS         │ Bawaan!  │  CycloneDDS         │
│  Interface: hnet0   │          │  Interface: hnet0   │
└─────────────────────┘          └─────────────────────┘
```

## Alat dan Prasyarat
- VPS Ubuntu 24.04
- PC Lokal dengan ROS2 Jazzy terinstal
- Akun gratis di [Husarnet Dashboard](https://app.husarnet.com)

---

## Langkah-Langkah

### Bagian A: Membuat Jaringan Husarnet (10 menit)

**Step 1: Buat Jaringan di Dashboard**
1. Login ke [app.husarnet.com](https://app.husarnet.com).
2. Buat jaringan baru (Create Network), beri nama misal `re405-robotics`.
3. Klik tombol **Add Element** dan catat **Join Code** yang diberikan (berbentuk deretan huruf dan angka panjang).

---

### Bagian B: Instalasi Husarnet di PC dan VPS (15 menit)

Lakukan langkah ini baik di **PC Lokal** maupun di **VPS**.

**Step 2: Install Husarnet**
Jalankan perintah instalasi otomatis di terminal:
```bash
curl https://install.husarnet.com/install.sh | sudo bash
```

**Step 3: Gabung ke Jaringan (Join)**
Ganti `<JOIN_CODE>` dengan kode yang Anda catat dari dashboard.
Untuk PC Lokal:
```bash
sudo husarnet join <JOIN_CODE> my-robot-pc
```
Untuk VPS:
```bash
sudo husarnet join <JOIN_CODE> my-vps-server
```

**Step 4: Cek Status Koneksi**
Periksa apakah kedua perangkat sudah terhubung:
```bash
husarnet status
```
Anda akan melihat interface jaringan baru bernama `hnet0` dengan alamat IPv6. Coba ping nama *hostname* VPS Anda dari PC Lokal:
```bash
ping6 my-vps-server
```

---

### Bagian C: Konfigurasi CycloneDDS (15 menit)

Husarnet sangat direkomendasikan untuk digunakan bersama **CycloneDDS** karena dukungan IPv6 multicast-nya yang sangat baik.

**Step 5: Install CycloneDDS**
Di PC dan VPS, pastikan CycloneDDS terinstal:
```bash
sudo apt update
sudo apt install ros-jazzy-rmw-cyclonedds-cpp
```

**Step 6: Buat Konfigurasi CycloneDDS Husarnet**
Siapkan file `cyclonedds_husarnet.xml` di PC dan VPS Anda. File ini sudah tersedia di dalam folder `configs/cyclonedds_husarnet.xml` pada repositori tutorial ini, cukup salin ke folder kerja Anda. Isinya sangat sederhana:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<CycloneDDS xmlns="https://cdds.io/config" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="https://cdds.io/config https://raw.githubusercontent.com/eclipse-cyclonedds/cyclonedds/master/etc/cyclonedds.xsd">
  <Domain id="any">
    <General>
      <NetworkInterfaceAddress>hnet0</NetworkInterfaceAddress>
      <AllowMulticast>true</AllowMulticast>
    </General>
    <Discovery>
      <ParticipantIndex>auto</ParticipantIndex>
      <MaxAutoParticipantIndex>120</MaxAutoParticipantIndex>
    </Discovery>
  </Domain>
</CycloneDDS>
```
*Perhatikan bahwa kita hanya perlu menyebutkan interface `hnet0` tanpa harus mendata IP satu per satu!*

**Step 7: Set Environment Variable**
Di PC dan VPS, jalankan:
```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI=file://$(pwd)/cyclonedds_husarnet.xml
source /opt/ros/jazzy/setup.bash
```

---

### Bagian D: Verifikasi Komunikasi (10 menit)

Sekarang, ROS2 Anda beroperasi persis seolah-olah berada dalam satu kabel LAN yang sama.

**Step 8: Test ROS2**
Jalankan talker di VPS:
```bash
ros2 run demo_nodes_cpp talker
```

Dan jalankan listener di PC Lokal:
```bash
ros2 run demo_nodes_cpp listener
```

Data akan mengalir secara lancar. Tidak perlu penentuan IP `unicast` manual, ROS2 bisa melakukan auto-discovery berkat fitur multicast IPv6 dari Husarnet!

---

## Latihan Mandiri
1. Tambahkan PC Lokal kedua ke dalam jaringan Husarnet yang sama dengan *Join Code* yang Anda miliki. Coba jalankan `ros2 topic list`. Apakah PC kedua otomatis langsung bisa melihat data `talker` dari VPS?
2. Bandingkan latensi jaringan Husarnet (yang berbasis IPv6) dengan Tailscale (Modul 1). Gunakan perintah `ros2 topic hz /chatter` untuk melihat frekuensi data yang masuk.

---

## Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| Perintah `husarnet` not found | Husarnet belum masuk system path. | Lakukan log out lalu log in kembali, atau jalankan perintah dengan path lengkap: `/usr/bin/husarnet`. |
| Topik tidak terlihat di mesin lain | Environment CycloneDDS belum diexport. | Pastikan `RMW_IMPLEMENTATION` dan `CYCLONEDDS_URI` diexport di *setiap* terminal baru. |
| Ping6 berhasil tapi ROS2 macet | IPv6 dinonaktifkan di sistem operasi. | Cek pengaturan `sysctl` Linux Anda, pastikan IPv6 aktif untuk interface `hnet0`. |

---
*Kembali ke [Halaman Utama (README)](README.md)*
