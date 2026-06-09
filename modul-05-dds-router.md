# Modul 5: eProsima DDS Router — WAN Bridge

## Tujuan Pembelajaran
- Mahasiswa mengetahui fungsi DDS Router untuk menjembatani berbagai jenis jaringan DDS.
- Mahasiswa mampu menggunakan DDS Router untuk menghubungkan ROS2 lokal ke VPS melalui koneksi WAN (TCP).
- Mahasiswa memahami perbedaan DDS Router dengan Discovery Server.

## Teori Singkat (10 menit)
- **Apa itu DDS Router?** DDS Router adalah aplikasi dari eProsima yang bertindak sebagai "bridge" (jembatan) pintar untuk meneruskan paket DDS (Data Distribution Service) dari satu jaringan ke jaringan lain.
- **Kelebihan DDS Router:** Berbeda dengan Discovery Server yang hanya mengelola "siapa ada di mana" (discovery), DDS Router benar-benar merutekan (routing) seluruh data pesan. Ini sangat berguna jika ada masalah NAT (Network Address Translation) berlapis yang tidak bisa ditembus oleh Discovery Server.
- **Topologi:** Pada modul ini, kita menjalankan DDS Router di VPS sebagai TCP Server. PC Lokal akan mengirim data menggunakan profil TCP Client (dikonfigurasi via FastDDS XML) menuju VPS. 

**Diagram Arsitektur:**
```text
PC Robot (ROS2 LAN)              VPS Ubuntu 24.04
┌──────────────────────┐          ┌──────────────────────┐
│  ROS2 Node (FastDDS) │          │  DDS Router          │
│  Mode: WAN TCP       │   TCP    │  Peserta WAN:        │
│  Client              │◄────────►│  Port 11811 (TCP)    │
│                      │          │  Peserta Local:      │
└──────────────────────┘          │  ROS2 Domain 1       │
                                  └──────────────────────┘
```

## Alat dan Prasyarat
- VPS Ubuntu 24.04
- PC Lokal dengan ROS2 Jazzy terinstal
- File konfigurasi `ddsrouter.yaml` (tersedia di folder `configs/`)

---

## Langkah-Langkah

### Bagian A: Instalasi dan Setup DDS Router di VPS (15 menit)

**Step 1: Install DDS Router**
Di terminal VPS Anda, kita bisa menginstall DDS Router dari repositori paket eProsima atau merakitnya (compile). Untuk kemudahan, kita anggap paket `ddsrouter` sudah terinstall (contoh via docker atau apt). Jika menggunakan docker:
```bash
sudo apt update
sudo apt install docker.io
```

**Step 2: Menyiapkan Konfigurasi DDS Router**
Siapkan file konfigurasi bernama `ddsrouter.yaml` di VPS Anda. File ini sudah tersedia di dalam folder `configs/ddsrouter.yaml` pada repositori tutorial ini, sehingga Anda cukup menyalinnya. Isinya adalah sebagai berikut:
```yaml
# ddsrouter.yaml
version: v3.0

participants:
  - name: WanParticipant
    kind: local
    domain: 0
    transport: tcp
    listening_addresses:
      - ip: 0.0.0.0
        port: 11811

  - name: LocalParticipant
    kind: local
    domain: 1
```

**Step 3: Buka Port di Firewall VPS**
Pastikan port 11811 TCP sudah terbuka:
```bash
sudo ufw allow 11811/tcp
```

**Step 4: Jalankan DDS Router**
Jalankan router di VPS menggunakan docker (atau aplikasinya langsung jika terinstall native):
```bash
docker run -it --rm --network host \
  -v $(pwd)/ddsrouter.yaml:/ddsrouter.yaml \
  eprosima/dds-router:latest \
  -c /ddsrouter.yaml
```
DDS router sekarang hidup dan mendengarkan koneksi WAN dari PC lokal.

---

### Bagian B: Konfigurasi PC Lokal (20 menit)

Di PC lokal, kita perlu memaksa FastDDS menggunakan profil TCP Client untuk terhubung ke IP publik VPS di port 11811.

**Step 5: Buat File XML FastDDS di PC**
Buat file `ddsrouter_client.xml` di PC lokal:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
    <participant profile_name="TCP_WAN_Client" is_default_profile="true">
        <rtps>
            <builtin>
                <initialPeersList>
                    <locator>
                        <tcpv4>
                            <address><VPS_PUBLIC_IP></address>
                            <port>11811</port>
                        </tcpv4>
                    </locator>
                </initialPeersList>
            </builtin>
        </rtps>
    </participant>
</profiles>
```
*Pastikan Anda mengganti `<VPS_PUBLIC_IP>` dengan IP asli VPS Anda!*

**Step 6: Set Environment Variable di PC**
Sama seperti pada Modul 1, beritahu ROS2 untuk memakai file XML ini:
```bash
export FASTRTPS_DEFAULT_PROFILES_FILE=$(pwd)/ddsrouter_client.xml
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

**Step 7: Jalankan Node ROS2**
Jalankan node talker di terminal PC:
```bash
ros2 run demo_nodes_cpp talker
```

---

### Bagian C: Verifikasi Koneksi (15 menit)

Data `/chatter` sekarang sudah menyeberang ke VPS, di-routing oleh DDS Router dan diteruskan ke domain 1.

**Step 8: Cek di VPS**
Buka terminal baru di VPS (biarkan DDS router tetap jalan). Karena DDS router me-rutekan pesan ke `domain: 1` (lihat config yaml), maka kita harus melakukan echo di domain 1:
```bash
export ROS_DOMAIN_ID=1
source /opt/ros/jazzy/setup.bash
ros2 topic echo /chatter
```
Jika semuanya dikonfigurasi dengan benar, terminal VPS akan menampilkan `"Hello World"` yang dikirim dari PC Lokal Anda.

---

## Latihan Mandiri
1. Ubahlah `ddsrouter.yaml` di VPS agar menerima input dari ROS_DOMAIN_ID=0 (WAN) dan meneruskannya lagi ke ROS_DOMAIN_ID=0 (Lokal VPS).
2. Tambahkan penyaringan topik (Topic Filtering) di `ddsrouter.yaml` sehingga ia HANYA merutekan topic `/chatter` dan memblokir topic lain (seperti `/parameter_events`). Baca dokumentasi eProsima untuk cara menambahkan *allowlist* atau *denylist*.
3. Jika koneksi Anda terputus sewaktu-waktu, amati apa yang terjadi pada log (output) dari `ddsrouter` di terminal VPS.

---

## Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| DDS Router langsung exit saat dijalankan | Format YAML ada yang salah (spasi/tab) | Pastikan file `ddsrouter.yaml` menggunakan spasi (bukan tab) dan indentasi benar. |
| ROS2 di PC macet saat dijalankan | Variabel path `FASTRTPS_DEFAULT_PROFILES_FILE` salah. | Pastikan menggunakan *absolute path* seperti `/home/user/ddsrouter_client.xml`. |
| Topik terbaca, tapi `ros2 topic echo` tidak mengeluarkan data | Pesan tersangkut di Discovery. Tipe data belum selaras antara PC dan VPS. | Jalankan `ros2 topic info /chatter -v` untuk mengecek status QOS (Quality of Service) topik. |

---
*Kembali ke [Halaman Utama (README)](README.md)*
