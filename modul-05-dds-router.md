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

**Step 1: Install DDS Router (via Docker)**
Image resmi eProsima DDS Router tidak tersedia di DockerHub untuk di-pull langsung. Anda harus mengunduhnya dari website eProsima.

1. Buka halaman [eProsima Downloads](https://www.eprosima.com/index.php/downloads-all) di browser PC Anda.
2. Unduh file Docker image DDS Router (misalnya versi `ubuntu-ddsrouter v3.5.1.tar`).
3. Upload file `.tar` tersebut ke VPS Anda (bisa menggunakan `scp`, SFTP, atau aplikasi seperti FileZilla).
4. Di terminal VPS, install Docker dan load image tersebut:
```bash
sudo apt update
sudo apt install -y docker.io

# Load image ke Docker (sesuaikan nama file dengan versi yang Anda unduh)
sudo docker load -i "ubuntu-ddsrouter v3.5.1.tar"
```

> [!TIP]
> **Alternatif: Build dari Source Code (Tanpa Docker)**
> Jika VPS Anda tidak bisa/tidak boleh diinstal Docker, Anda bisa melakukan kompilasi native mengikuti [Developer Manual eProsima](https://eprosima-dds-router.readthedocs.io/en/latest/rst/developer_manual/installation/sources/linux.html). Langkahnya kira-kira sebagai berikut:
> ```bash
> sudo apt install -y cmake g++ pip wget git libasio-dev libtinyxml2-dev libssl-dev libyaml-cpp-dev
> pip3 install -U colcon-common-extensions vcstool
> 
> mkdir -p ~/DDS-Router/src && cd ~/DDS-Router
> wget https://raw.githubusercontent.com/eProsima/DDS-Router/main/ddsrouter.repos
> vcs import src < ddsrouter.repos
> colcon build
> 
> # Cara menjalankannya nanti:
> source ~/DDS-Router/install/setup.bash
> ddsrouter -c /path/to/ddsrouter.yaml
> ```

**Step 2: Menyiapkan Konfigurasi DDS Router**
Siapkan file konfigurasi bernama `ddsrouter.yaml` di VPS Anda. File ini sudah tersedia di dalam folder `configs/ddsrouter.yaml` pada repositori tutorial ini, sehingga Anda cukup menyalinnya. Isinya adalah sebagai berikut:
```yaml
# ddsrouter.yaml
version: v4.0

participants:
  - name: WanParticipant
    kind: wan
    listening-addresses:
      - ip: <VPS_PUBLIC_IP>
        port: 11811
        transport: tcp

  - name: LocalParticipant
    kind: local
    domain: 1
```
*Pastikan Anda mengganti `<VPS_PUBLIC_IP>` dengan IP Publik asli VPS Anda, karena DDS Router TCP Server perlu mengetahui IP mana yang diekspos ke internet.*

**Step 3: Buka Port di Firewall VPS**
Pastikan port 11811 TCP sudah terbuka:
```bash
sudo ufw allow 11811/tcp
```

**Step 4: Jalankan DDS Router**
Jalankan DDS Router di VPS menggunakan Docker:
```bash
sudo docker run -it --rm --net=host \
  -v $(pwd)/ddsrouter.yaml:/root/DDS_ROUTER_CONFIGURATION.yaml \
  ubuntu-ddsrouter:v3.5.1
```
*(Sesuaikan tag `v3.5.1` dengan versi yang Anda load sebelumnya. Pastikan file `ddsrouter.yaml` berada di direktori saat ini `$(pwd)` karena path harus absolut.)*

DDS router sekarang hidup dan mendengarkan koneksi WAN dari PC lokal.

---

### Bagian B: Konfigurasi PC Lokal (20 menit)

Sesuai dengan panduan arsitektur eProsima WAN TCP, komunikasi antara dua jaringan (VPS dan Lokal) dijembatani oleh **dua DDS Router**: satu sebagai Server (yang baru saja kita jalankan di VPS), dan satu lagi sebagai Client di PC lokal Anda.

**Step 5: Buat File ddsrouter_client.yaml di PC**
Buat file `ddsrouter_client.yaml` di PC lokal Anda. File ini akan membuat PC Anda bertindak sebagai TCP Client yang melakukan koneksi ke VPS:
```yaml
# ddsrouter_client.yaml
version: v4.0

participants:
  - name: WanParticipantClient
    kind: wan
    connection-addresses:
      - ip: <VPS_PUBLIC_IP>
        port: 11811
        transport: tcp

  - name: LocalParticipant
    kind: local
    domain: 0   # Mengikuti domain default ROS2 di PC Anda
```
*Pastikan Anda mengganti `<VPS_PUBLIC_IP>` dengan IP asli VPS Anda!*

**Step 6: Jalankan DDS Router Client di PC**
Sama seperti pada VPS, jalankan DDS Router di PC lokal Anda menggunakan Docker:
```bash
sudo docker run -it --rm --net=host \
  -v $(pwd)/ddsrouter_client.yaml:/root/DDS_ROUTER_CONFIGURATION.yaml \
  ubuntu-ddsrouter:v3.5.1
```
*(Asumsinya Anda juga sudah mengunduh dan me-load Docker image `.tar` eProsima di PC lokal. Jika belum, ulangi Step 1 di PC Anda).*

**Step 7: Jalankan Node ROS2**
Buka terminal baru di PC (biarkan DDS Router Client berjalan), lalu jalankan node ROS 2 standar tanpa perlu konfigurasi XML tambahan:
```bash
ros2 run demo_nodes_cpp talker
```
Karena node `talker` menggunakan `domain: 0` (default) dan DDS Router Client juga memiliki partisipan lokal di `domain: 0`, pesan akan otomatis ditangkap oleh DDS Router dan diteruskan via TCP ke DDS Router Server di VPS.

---

### Bagian C: Verifikasi Koneksi (15 menit)

Data `/chatter` sekarang sudah menyeberang ke VPS, di-routing oleh DDS Router dan diteruskan ke domain 1.

**Step 8: Cek di VPS**
Buka terminal baru di VPS (biarkan DDS router tetap jalan). Karena DDS router me-rutekan pesan ke `domain: 1` (lihat config yaml — peserta `LocalParticipant` berada di domain 1), maka kita harus melakukan echo di domain 1:
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
