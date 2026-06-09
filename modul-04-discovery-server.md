# Modul 4: FastDDS Discovery Server (No VPN)

## Tujuan Pembelajaran
- Mahasiswa memahami mekanisme Discovery Server pada FastDDS.
- Mahasiswa mampu menjalankan FastDDS Discovery Server di VPS menggunakan koneksi TCP.
- Mahasiswa mampu menghubungkan node ROS2 di PC ke VPS tanpa menggunakan VPN (murni TCP via Public IP).

## Teori Singkat (10 menit)
- **Kelemahan Simple Discovery:** ROS2 menggunakan metode "Simple Discovery" di mana setiap node berteriak (multicast) "Saya ada di sini!" ke seluruh jaringan. Ini boros bandwidth dan tidak berfungsi di internet (WAN).
- **Discovery Server:** FastDDS menyediakan mode "Discovery Server" berarsitektur Client-Server. Semua node melapor ke satu server sentral (di VPS). Server ini yang bertugas memberi tahu node A bagaimana cara berkomunikasi dengan node B.
- **Kelebihan:** Sangat ringan, traffic discovery berkurang drastis, dan cocok untuk menghubungkan robot di berbagai lokasi via internet tanpa VPN (karena kita akan memaksanya menggunakan protokol TCP).

**Diagram Arsitektur:**
```text
PC Robot 1 (ROS2)                VPS Ubuntu 24.04                PC Robot 2 (ROS2)
┌──────────────────────┐          ┌──────────────────────┐       ┌──────────────────────┐
│  ROS2 Node           │          │  FastDDS Discovery   │       │  ROS2 Node           │
│  (Client)            │   TCP    │  Server (ID: 0)      │  TCP  │  (Client)            │
│                      │◄────────►│                      │◄─────►│                      │
│ ROS_DISCOVERY_SERVER │          │  Port: 11811 (TCP)   │       │ ROS_DISCOVERY_SERVER │
└──────────────────────┘          └──────────────────────┘       └──────────────────────┘
            │                                                               │
            └────────────────────── Peer-to-Peer Data ──────────────────────┘
                                  (Jika jaringan memungkinkan)
```

## Alat dan Prasyarat
- VPS Ubuntu 24.04 dengan ROS2 Jazzy terinstal
- PC Lokal dengan ROS2 Jazzy terinstal

---

## Langkah-Langkah

### Bagian A: Menjalankan Discovery Server di VPS (15 menit)

**Step 1: Buka Port di VPS**
Secara default, server akan berjalan di port TCP/UDP yang kita tentukan. Mari kita gunakan port `11811`.
```bash
sudo ufw allow 11811/tcp
sudo ufw allow 11811/udp
```
*(Ingat buka di panel Security Group jika menggunakan AWS/Azure!)*

**Step 2: Start Discovery Server**
Buka terminal VPS. Source ROS2, kemudian jalankan command `fastdds discovery`:
```bash
source /opt/ros/jazzy/setup.bash
fastdds discovery -i 0 -l 0.0.0.0 -p 11811
```
Penjelasan argumen:
- `-i 0`: Menetapkan ID server ini adalah 0.
- `-l 0.0.0.0`: Mendengarkan koneksi dari semua antarmuka jaringan (IP publik maupun lokal).
- `-p 11811`: Menggunakan port 11811.

Server sekarang aktif dan mendengarkan koneksi masuk. Biarkan terminal ini terbuka.

---

### Bagian B: Koneksi PC Client ke VPS (15 menit)

**Step 3: Setup Environment Variable di PC**
Buka terminal di PC Lokal Anda. Kita perlu memberi tahu ROS2 agar *tidak* menggunakan Simple Discovery multicast, melainkan menggunakan server kita.

Ganti `<VPS_PUBLIC_IP>` dengan IP publik VPS Anda.
```bash
export ROS_DISCOVERY_SERVER="<VPS_PUBLIC_IP>:11811"
```

**Step 4: Jalankan ROS2 Node di PC**
Di terminal yang sama (yang sudah di-export), jalankan sebuah listener:
```bash
ros2 run demo_nodes_cpp listener
```
Node ini sekarang sudah terhubung ke Discovery Server di VPS.

---

### Bagian C: Menghubungkan Dua PC atau VPS (20 menit)

Sekarang mari kita buktikan. Kita akan menjalankan talker di mesin yang berbeda (bisa dari VPS itu sendiri, atau dari PC teman Anda).

**Step 5: Setup Client di Mesin Kedua**
Buka terminal baru di VPS (atau di PC teman Anda yang berbeda jaringan).
Sama seperti tadi, arahkan `ROS_DISCOVERY_SERVER` ke IP yang sama:
```bash
# Jika di terminal VPS, cukup arahkan ke localhost
export ROS_DISCOVERY_SERVER="127.0.0.1:11811"

# Jika di PC teman Anda:
# export ROS_DISCOVERY_SERVER="<VPS_PUBLIC_IP>:11811"
```

**Step 6: Jalankan Talker**
```bash
source /opt/ros/jazzy/setup.bash
ros2 run demo_nodes_cpp talker
```

Kembali ke terminal listener di Step 4. Anda akan melihat pesan `"Hello World"` berhasil diterima!
Koneksi ROS2 Anda sekarang terhubung via WAN tanpa VPN, murni menggunakan infrastruktur bawaan FastDDS.

---

## Latihan Mandiri
1. Buat arsitektur Multi-Server! Jalankan server ID 0 di VPS, dan server ID 1 di PC Lokal (port berbeda). Hubungkan keduanya.
2. Coba jalankan perintah `ros2 topic list` di terminal baru di PC Anda. Mengapa daftar topic kosong? (Petunjuk: ROS2 CLI juga bertindak sebagai node yang harus mengetahui keberadaan Discovery Server).
3. Matikan server di VPS dengan menekan `Ctrl+C`. Apa yang terjadi pada komunikasi *talker* dan *listener* yang sudah berjalan? (Fakta: mereka tetap berkomunikasi secara *peer-to-peer* selama IP mereka tidak berubah).

---

## Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| Talker jalan, tapi Listener tidak menerima pesan | Port belum terbuka atau terblokir firewall provider cloud. | Cek status firewall ufw. Gunakan `nc -zv <VPS_IP> 11811` dari PC untuk mengetes apakah port TCP/UDP terbuka. |
| Node lama terhubung / gagal terima data (timeout) | PC klien berada di balik Symmetric NAT (contoh: tethering provider seluler). | Discovery Server kadang gagal menembus NAT yang sangat ketat untuk data peer-to-peer. Jika ini terjadi, gunakan Modul 5 (DDS Router). |
| `ros2 topic list` tidak menampilkan apapun | Terminal tersebut belum di-export variabel server. | Jalankan `export ROS_DISCOVERY_SERVER="<VPS_PUBLIC_IP>:11811"` sebelum menjalankan perintah CLI ROS2. |

---
*Lanjut ke [Modul 5: eProsima DDS Router — WAN Bridge](modul-05-dds-router.md)*
