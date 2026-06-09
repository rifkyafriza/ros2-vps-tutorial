# Modul 1: Koneksi ROS2 via Tailscale VPN

## Tujuan Pembelajaran
- Mahasiswa memahami konsep jaringan layer-3 VPN untuk menghubungkan perangkat IoT/Robot.
- Mahasiswa mampu mengonfigurasi Tailscale pada PC dan VPS Ubuntu.
- Mahasiswa mampu menghubungkan dua mesin ROS2 yang berbeda jaringan menjadi satu jaringan logis.

## Teori Singkat (10 menit)
- **Masalah Multicast:** Secara default, ROS2 menggunakan UDP Multicast untuk menemukan node lain (discovery). UDP Multicast biasanya diblokir oleh router di jaringan publik/WAN (internet).
- **Tailscale:** Layanan VPN berbasis WireGuard yang membangun "Mesh Network". Dengan Tailscale, PC dan VPS seolah-olah berada di jaringan lokal (LAN) yang sama.
- **FastDDS Unicast:** Karena VPN kadang tidak meneruskan paket Multicast dengan baik, kita mengonfigurasi FastDDS (middleware bawaan ROS2 Jazzy) untuk menggunakan Unicast langsung ke IP Tailscale masing-masing node.

**Diagram Arsitektur:**
```text
PC Lokal (ROS2)                  VPS Ubuntu 24.04
┌─────────────────────┐          ┌─────────────────────┐
│  ┌───────────────┐  │ Tailnet  │  ┌───────────────┐  │
│  │  ROS2 Node    │  │◄────────►│  │  ROS2 Node    │  │
│  │  (talker)     │  │ WireGuard│  │  (listener)   │  │
│  └───────────────┘  │   VPN    │  └───────────────┘  │
│  Tailscale: 100.x.x │          │  Tailscale: 100.y.y │
│  FastDDS Unicast    │          │  FastDDS Unicast     │
└─────────────────────┘          └─────────────────────┘
         │                                │
         └─────── Tailscale Mesh ─────────┘
                  (internet)
```

## Alat dan Prasyarat
- Akun Tailscale gratis (daftar di [tailscale.com](https://tailscale.com))
- VPS Ubuntu 24.04 dengan IP publik (contoh: DigitalOcean, AWS, Azure)
- ROS2 Jazzy terinstal di PC lokal dan VPS

---

## Langkah-Langkah

### Bagian A: Setup Tailscale (30 menit)

**Step 1: Install Tailscale di VPS**
Buka terminal VPS Anda (via SSH) dan jalankan command berikut:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
Terminal akan menampilkan sebuah link. Buka link tersebut di browser untuk login dan mendaftarkan VPS ke akun Tailscale Anda.

Cek IP Tailscale VPS Anda (biasanya berawalan 100.x.x.x):
```bash
tailscale ip -4
```
*(Catat IP ini, kita akan menyebutnya `<VPS_TAILSCALE_IP>`)*

**Step 2: Install Tailscale di PC Lokal**
Buka terminal di PC lokal Anda (Ubuntu) dan jalankan:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
Login seperti langkah sebelumnya.
Cek IP Tailscale PC Anda:
```bash
tailscale ip -4
```
*(Catat IP ini, kita akan menyebutnya `<PC_TAILSCALE_IP>`)*

**Step 3: Verifikasi Koneksi Tailscale**
Dari terminal PC, coba ping IP VPS:
```bash
ping <VPS_TAILSCALE_IP>
```
Lakukan sebaliknya, dari VPS ping IP PC. Jika berhasil, kedua mesin sudah terhubung.

**Step 4: Konfigurasi Firewall VPS**
Pastikan firewall di VPS mengizinkan komunikasi lewat jaringan Tailscale.
```bash
sudo ufw allow in on tailscale0
```
> [!NOTE]
> Jika VPS Anda di cloud provider yang menggunakan Security Group (seperti AWS atau Azure), pastikan port UDP 7400-7500 dibuka untuk rentang IP Tailscale.

---

### Bagian B: Konfigurasi FastDDS Unicast (20 menit)

Karena ROS2 mengandalkan multicast, kita perlu memaksanya menggunakan unicast agar stabil di Tailscale.

**Step 5: Buat File Profil FastDDS**
Di PC Lokal dan VPS, siapkan file profil `fastdds_tailscale.xml`. File ini sudah tersedia di dalam folder `configs/fastdds_tailscale.xml` pada repositori tutorial ini, jadi Anda cukup menyalinnya ke `~/fastdds_tailscale.xml`.

Gunakan konfigurasi berikut (ganti `<VPS_TAILSCALE_IP>` dan `<PC_TAILSCALE_IP>` sesuai IP Anda):
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
  <participant profile_name="TailscaleProfile" is_default_profile="true">
    <rtps>
      <builtin>
        <initialPeersList>
          <locator>
            <udpv4>
              <address><VPS_TAILSCALE_IP></address>
            </udpv4>
          </locator>
        </initialPeersList>
        <metatrafficUnicastLocatorList>
          <locator>
            <udpv4>
              <address><PC_TAILSCALE_IP></address>
              <port>7412</port>
            </udpv4>
          </locator>
        </metatrafficUnicastLocatorList>
      </builtin>
    </rtps>
  </participant>
</profiles>
```
*(Perhatikan: file XML harus diubah IP-nya sesuai dengan node mana ia berjalan. Pada PC lokal, bagian unicast menggunakan IP PC Lokal. Jika ini dijalankan di VPS, unicast locator harus menggunakan IP VPS dan peers list mengarah ke IP PC).*

**Step 6: Set Environment Variables**
Tambahkan konfigurasi berikut ke file `~/.bashrc` Anda di **PC Lokal dan VPS**:
```bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export FASTRTPS_DEFAULT_PROFILES_FILE=~/fastdds_tailscale.xml
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
```

> [!WARNING]
> **ROS_DOMAIN_ID** harus unik per kelompok! Karena semua kelompok di kelas menggunakan jaringan Tailscale yang sama, gunakan rumus: `(Nomor_Kelompok * 2) % 230` untuk mencegah konflik.

Terapkan perubahan dengan menjalankan:
```bash
source ~/.bashrc
```

---

### Bagian C: Test Koneksi ROS2 (30 menit)

**Step 7: Test Cross-Machine Talker-Listener**
Di terminal VPS, jalankan sebuah *publisher* (talker):
```bash
ros2 run demo_nodes_cpp talker
```

Di terminal PC Lokal, periksa apakah topic muncul:
```bash
ros2 topic list
```
*(Anda harus melihat `/chatter` muncul dalam daftar)*

Kemudian dengarkan topic tersebut:
```bash
ros2 topic echo /chatter
```
Jika Anda melihat pesan `"Hello World: X"`, selamat! ROS2 Anda sudah terhubung lewat internet.

**Step 8: Lakukan Sebaliknya (PC Publish, VPS Subscribe)**
Jalankan talker di PC Lokal, dan lakukan `ros2 topic echo /chatter` di VPS.

---

## Latihan Mandiri
1. Jalankan `turtlesim_node` di PC Lokal. Cobalah membuat node di VPS (dengan python) yang menerbitkan (`publish`) perintah kecepatan ke topic `/turtle1/cmd_vel` untuk menggerakkan kura-kura di PC Anda!
2. Cobalah kirim topic `/scan` dari VPS ke PC, dan amati seberapa besar *delay* jaringan dengan perintah `ros2 topic hz /scan`.
3. Ubahlah `ROS_DOMAIN_ID` di PC Lokal menjadi angka yang berbeda dengan VPS. Apakah topic `/chatter` masih bisa dilihat?

---

## Troubleshooting

| Masalah                                          | Kemungkinan Penyebab                                        | Solusi                                                                    |
| --------------------------------------------------| -------------------------------------------------------------| ---------------------------------------------------------------------------|
| `ros2 topic list` kosong                         | `ROS_DOMAIN_ID` tidak sama atau file XML salah konfigurasi. | Samakan nilai `ROS_DOMAIN_ID`. Cek ulang IP di `fastdds_tailscale.xml`.   |
| Ping Tailscale berhasil, tapi topic tidak muncul | Variabel `FASTRTPS_DEFAULT_PROFILES_FILE` belum diterapkan. | Pastikan path file benar dan jalankan `source ~/.bashrc`.                 |
| Connection refused atau sangat lambat            | Firewall VPS memblokir port UDP FastDDS.                    | Jalankan `sudo ufw allow in on tailscale0` di VPS.                        |
| Tailscale status "Relayed" (bukan "Direct")      | Koneksi terhalang NAT berlapis (Symmetric NAT).             | Biarkan menggunakan relay (agak lambat), atau setup STUN/port forwarding. |

---
*Lanjut ke [Modul 2: Remote Control via rosbridge WebSocket](modul-02-rosbridge.md)*
