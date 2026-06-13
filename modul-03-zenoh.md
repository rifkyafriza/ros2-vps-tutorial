# Modul 3: Koneksi Native WAN dengan Zenoh Bridge

## Tujuan Pembelajaran
- Mahasiswa memahami batasan protokol DDS pada Wide Area Network (WAN).
- Mahasiswa mengetahui Zenoh sebagai protokol pub/sub modern yang dioptimalkan untuk Cloud dan IoT.
- Mahasiswa mampu menghubungkan node ROS2 tanpa VPN menggunakan `zenoh-bridge-ros2dds`.

## Teori Singkat (10 menit)
- **Keterbatasan DDS di WAN:** DDS dibuat secara optimal untuk jaringan LAN yang mendukung UDP multicast. Ketika dihadapkan pada WAN atau Internet, NAT traversal dan ketiadaan dukungan multicast membuat DDS sangat rumit dikonfigurasi (seperti pada Modul 1).
- **Apa itu Zenoh?** Zenoh adalah protokol pub/sub dari Eclipse Foundation yang dirancang efisien untuk edge-computing, IoT, dan cloud. Zenoh mendukung komunikasi native berbasis TCP/UDP, membuatnya kebal terhadap masalah multicast di internet.
- **Konsep Bridge:** Kita tidak akan mengganti middleware ROS2 (FastDDS) di PC. ROS2 di PC tetap jalan normal di LAN. Namun, kita memasang sebuah program bernama `zenoh-bridge-ros2dds` yang akan mencegat data ROS2 dan mengubahnya menjadi protokol Zenoh, lalu menembakkannya via TCP langsung ke VPS.

**Diagram Arsitektur:**
```text
PC Robot (ROS2)                  VPS Ubuntu 24.04
┌──────────────────────┐          ┌─────────────────────┐
│  ROS2 Node (normal)  │          │  Zenoh Router       │
│  ┌─────────────────┐ │  TCP     │  (binary saja,      │
│  │ zenoh-bridge    │─┼─────────►│   tidak perlu ROS2) │
│  │ ros2dds (peer)  │ │ 7447     │  Port 7447 (TCP)    │
│  └─────────────────┘ │          └─────────────────────┘
└──────────────────────┘                   ▲
  Tidak perlu konfigurasi                  │
  DDS atau VPN tambahan!           Bisa ada banyak
                                   robot terhubung
```

## Alat dan Prasyarat
- VPS dengan Ubuntu 24.04
- ROS2 Jazzy terinstal di PC lokal
- *Catatan Penting:* Anda TIDAK perlu menginstal ROS2 di VPS untuk modul ini!

---

## Langkah-Langkah

### Bagian A: Setup Zenoh Router di VPS (20 menit)

**Step 1: Download binary Zenoh Bridge di VPS**
Buka terminal SSH VPS Anda. Kita hanya akan men-download file *executable* pre-compiled.
```bash
# Metode 1 (Rekomendasi): Install dari repositori Debian Eclipse Zenoh
echo "deb [trusted=yes] https://download.eclipse.org/zenoh/debian-repo/ /" | sudo tee -a /etc/apt/sources.list > /dev/null
sudo apt update
sudo apt install zenoh-bridge-ros2dds
```

> [!TIP]
> Alternatif: Download binary manual dari [GitHub Releases](https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds/releases):
> ```bash
> wget https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds/releases/download/1.9.0/zenoh-plugin-ros2dds-1.9.0-x86_64-unknown-linux-gnu-standalone.zip
> sudo apt install unzip
> unzip zenoh-plugin-ros2dds-1.9.0-x86_64-unknown-linux-gnu-standalone.zip
> chmod +x zenoh-bridge-ros2dds
> ```

**Step 2: Buka port 7447 di VPS**
Zenoh secara default berkomunikasi menggunakan port `7447` TCP. Buka port tersebut:
```bash
sudo ufw allow 7447/tcp
```
*(Jangan lupa Security Group AWS/Azure jika menggunakan cloud provider tersebut).*

**Step 3: Jalankan Zenoh Router di VPS**
Jalankan program bridge dalam mode `router` dan minta ia mendengarkan (listen) di semua antarmuka jaringan:
```bash
./zenoh-bridge-ros2dds --mode router --listen tcp/0.0.0.0:7447
```
Biarkan terminal ini terbuka. VPS sudah siap menerima koneksi Zenoh.

---

### Bagian B: Setup Zenoh Bridge di PC Robot (20 menit)

**Step 4: Download binary Zenoh Bridge di PC Lokal**
Sama seperti di VPS, buka terminal di Ubuntu PC lokal Anda dan install bridge:
```bash
# Metode 1 (jika sudah ditambahkan repo Eclipse Zenoh):
sudo apt install zenoh-bridge-ros2dds

# Metode 2 (download manual):
wget https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds/releases/download/1.9.0/zenoh-plugin-ros2dds-1.9.0-x86_64-unknown-linux-gnu-standalone.zip
unzip zenoh-plugin-ros2dds-1.9.0-x86_64-unknown-linux-gnu-standalone.zip
chmod +x zenoh-bridge-ros2dds
```

**Step 5: Jalankan Zenoh Bridge di PC**
Ganti `<VPS_PUBLIC_IP>` dengan IP Publik dari VPS Anda. Jalankan bridge dalam mode `peer` (atau client) agar terhubung ke router di VPS:
```bash
./zenoh-bridge-ros2dds --mode peer --connect tcp/<VPS_PUBLIC_IP>:7447
```

**Step 6: Jalankan Node ROS2 Secara Normal**
Buka terminal baru di PC Lokal. Anda *TIDAK* perlu mengatur `FASTRTPS_DEFAULT_PROFILES_FILE` seperti di Modul 1. Cukup source ROS2 dan jalankan talker biasa:
```bash
ros2 run demo_nodes_cpp talker
```

---

### Bagian C: Verifikasi Koneksi (20 menit)

Data `/chatter` sekarang sudah menyeberang via internet menuju router Zenoh di VPS. Untuk membuktikannya, kita bisa subscribe data tersebut langsung dari VPS, menggunakan `zenoh-python`.

**Step 7: Install Zenoh Python Client di VPS**
Buka terminal baru di VPS (biarkan router Zenoh di terminal sebelumnya tetap berjalan).
```bash
sudo apt update
sudo apt install python3-pip
pip3 install eclipse-zenoh
```

**Step 8: Buat Script Python untuk Subscribe (VPS)**
Di VPS, buat file `zenoh_sub.py`:
```python
import zenoh
import time

print("Menghubungkan ke Zenoh router lokal...")
# Connect ke router Zenoh yang jalan di localhost VPS
with zenoh.open(zenoh.Config()) as session:

    def listener(sample):
        print(f">> Menerima payload di topic: {sample.key_expr}")
        print(f"   Isi (bytes): {sample.payload.to_bytes()}")

    # Subscribe ke namespace ROS2 via Zenoh
    print("Mendengarkan topic rt/chatter ...")
    sub = session.declare_subscriber("rt/chatter", listener)

    time.sleep(60)
```

Jalankan skrip:
```bash
python3 zenoh_sub.py
```
Anda akan melihat data payload bytes mentah yang mewakili pesan "Hello World" dari `demo_nodes_cpp talker` yang berjalan di PC Anda! 

---

## Konfigurasi Menggunakan File (Opsional)
Mengetik argumen CLI yang panjang sangat merepotkan. Kita bisa menggunakan file konfigurasi JSON5 (contohnya ada di direktori `configs/zenoh-router.json5` dan `configs/zenoh-client.json5`).
Jalankan program cukup dengan:
```bash
./zenoh-bridge-ros2dds --config configs/zenoh-client.json5
```

---

## Perbandingan 3 Metode (Sejauh ini)

| Aspek | Tailscale (Modul 1) | rosbridge (Modul 2) | Zenoh (Modul 3) |
|-------|-----------|-----------|-------|
| Install di VPS | ROS2 + Tailscale | ROS2 + rosbridge | Hanya binary |
| Protokol | UDP (DDS) via VPN | JSON WebSocket | Binary TCP |
| Butuh VPN | Ya | Tidak | Tidak |
| Client non-ROS2 | Sulit (harus pakai VPN) | Mudah (Web/Python) | Mudah (zenoh python) |
| Kebutuhan CPU/RAM VPS | Besar (ROS2 penuh) | Sedang | Sangat Kecil |

---

## Latihan Mandiri
1. Cobalah jalankan *dua* PC Lokal secara bersamaan, keduanya menjalankan zenoh-bridge client yang mengarah ke IP VPS yang sama. Apakah kedua PC tersebut bisa saling *ping* topic `/chatter` satu sama lain tanpa terhubung dalam LAN/VPN?
2. Jalankan `ros2 run turtlesim turtlesim_node` di PC Lokal. Gunakan Zenoh bridge. Cobalah mengirim data ke topic `rt/turtle1/cmd_vel` langsung menggunakan Zenoh-Python dari VPS. (Ingat bahwa ROS2 topics ditambahkan prefix `rt/` secara default oleh Zenoh bridge).

---

## Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| zenoh-bridge di PC gagal terhubung (`connection refused`) | Port 7447 belum terbuka di VPS. | Pastikan `sudo ufw allow 7447/tcp` sudah dijalankan di VPS. |
| ROS2 berjalan, tapi tidak ter-bridge | `ROS_DOMAIN_ID` belum di-source di terminal bridge. | Jika ROS2 PC Anda menggunakan ID khusus, pastikan terminal yang menjalankan `zenoh-bridge-ros2dds` juga memiliki `export ROS_DOMAIN_ID=...` yang sama. |
| Python script tidak menerima apapun | Mismatch namespace topic. | Zenoh menambahkan prefix `rt/` ke topic biasa, misal `/chatter` menjadi `rt/chatter`. Pastikan string topic Anda benar. |

---
*Lanjut ke [Modul 4: FastDDS Discovery Server (No VPN)](modul-04-discovery-server.md)*
