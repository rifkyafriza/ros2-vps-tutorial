# Modul 2: Remote Control via rosbridge WebSocket

## Tujuan Pembelajaran
- Mahasiswa mampu menjelaskan arsitektur rosbridge dan penggunaannya.
- Mahasiswa mampu menjalankan rosbridge server di VPS.
- Mahasiswa mampu membuat client non-ROS2 (menggunakan Python) untuk berkomunikasi dengan robot via internet.

## Teori Singkat (10 menit)
- **Arsitektur rosbridge:** rosbridge menyediakan API berbasis JSON melalui antarmuka WebSocket. Ini memungkinkan aplikasi web, aplikasi mobile, atau script Python biasa berinteraksi dengan ROS2 tanpa perlu menginstal stack ROS2 yang lengkap di sisi client.
- **Skenario Cloud Robotics:** Dalam tutorial ini, kita mendeploy `rosbridge_server` di VPS. Robot di PC (yang terhubung ke VPS via Tailscale dari Modul 1) akan publish datanya, lalu client dari manapun di internet bisa membuka WebSocket ke VPS (port 9090) untuk memonitor atau mengontrol robot.

**Diagram Arsitektur:**
```text
PC Robot (ROS2)              VPS Ubuntu 24.04         Client Mana Saja
┌─────────────────┐          ┌──────────────────┐     ┌──────────────┐
│  ROS2 Topics    │◄────────►│  rosbridge_suite │◄───►│  Browser     │
│  /scan          │ Tailscale│  ws://VPS:9090   │     │  Python App  │
│  /cmd_vel       │ (Mod 1)  │  JSON WebSocket  │     │  roslibjs    │
│  /pose          │          │  Port 9090 (TCP) │     │  roslibpy    │
└─────────────────┘          └──────────────────┘     └──────────────┘
                                      │
                              Tidak perlu ROS2
                              di sisi client!
```

## Alat dan Prasyarat
- Telah menyelesaikan Modul 1 (Tailscale)
- ROS2 Jazzy terinstal di VPS

---

## Langkah-Langkah

### Bagian A: Install dan Jalankan rosbridge di VPS (15 menit)

**Step 1: Install rosbridge_suite di VPS**
Buka terminal SSH VPS Anda, lalu install paket rosbridge:
```bash
sudo apt update
sudo apt install ros-jazzy-rosbridge-suite
```

**Step 2: Buka Port 9090 di Firewall VPS**
WebSocket rosbridge secara default berjalan di port 9090 TCP. Anda harus membukanya:
```bash
sudo ufw allow 9090/tcp
```
*(Jika VPS Anda di AWS/Azure, buka juga port 9090 di panel kontrol Security Group atau Network Security Group masing-masing provider).*

**Step 3: Jalankan rosbridge server**
Pastikan ROS2 telah di-source, lalu jalankan:
```bash
source /opt/ros/jazzy/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```
*(Catatan: Anda tidak perlu membuat file `rosbridge_websocket_launch.xml` secara manual. File ini adalah file bawaan (built-in) yang otomatis didapatkan ketika menginstall paket `ros-jazzy-rosbridge-suite` di Step 1).*

Biarkan terminal ini terbuka. Server akan standby mendengarkan koneksi WebSocket.

---

### Bagian B: Koneksi Robot ke VPS (20 menit)

Karena kita sudah mengonfigurasi Tailscale + FastDDS Unicast di Modul 1, secara otomatis VPS akan "melihat" semua topic yang ada di jaringan Tailnet tersebut.

**Step 4: Jalankan Node Robot di PC Lokal**
Buka terminal di PC Lokal Anda, jalankan simulator `turtlesim`:
```bash
ros2 run turtlesim turtlesim_node
```

Buka terminal kedua di PC Lokal, jalankan teleop manual (keyboard) untuk memastikan sistem jalan:
```bash
ros2 run turtlesim turtle_teleop_key
```

**Step 5: Verifikasi Topic di VPS**
Buka terminal baru di VPS (sementara rosbridge tetap jalan di terminal lain), dan cek apakah topic dari PC terbaca:
```bash
ros2 topic list
```
Topic `/turtle1/cmd_vel` dan `/turtle1/pose` harusnya sudah terlihat di VPS. Ini berarti data tersebut otomatis tersedia juga untuk rosbridge.

---

### Bagian C: Client Python dari Luar Jaringan (25 menit)

Sekarang, asumsikan Anda sedang menggunakan PC *lain* yang tidak memiliki ROS2, atau Anda men-deploy dashboard di Heroku/Vercel. Kita akan mengontrol robot hanya dengan script Python standar.

**Step 6: Install roslibpy**
Di PC Client mana saja, install library Python untuk terhubung ke rosbridge:
```bash
pip3 install roslibpy
```

**Step 7: Script Subscribe (Menerima Data Posisi)**
Buat file baru bernama `monitor_robot.py`:
```python
import roslibpy

# Ganti <VPS_PUBLIC_IP> dengan IP Publik VPS Anda!
client = roslibpy.Ros(host='<VPS_PUBLIC_IP>', port=9090)
client.run()

def pose_callback(message):
    print(f"Robot berada di X: {message['x']:.2f}, Y: {message['y']:.2f}")

# Format tipe message untuk ROS2 menggunakan /msg/ di tengahnya
listener = roslibpy.Topic(client, '/turtle1/pose', 'turtlesim/msg/Pose')
listener.subscribe(pose_callback)

try:
    print("Mendengarkan posisi robot... (Tekan Ctrl+C untuk berhenti)")
    while client.is_connected:
        pass
except KeyboardInterrupt:
    pass

client.terminate()
```
Jalankan `python3 monitor_robot.py`. Anda akan melihat posisi kura-kura tercetak secara real-time.

**Step 8: Script Publish (Mengontrol Robot)**
Buat file baru bernama `control_robot.py`:
```python
import roslibpy
import time

client = roslibpy.Ros(host='<VPS_PUBLIC_IP>', port=9090)
client.run()

publisher = roslibpy.Topic(client, '/turtle1/cmd_vel', 'geometry_msgs/msg/Twist')

# Perintah bergerak maju melingkar
gerak_maju = {
    'linear': {'x': 1.0, 'y': 0.0, 'z': 0.0},
    'angular': {'x': 0.0, 'y': 0.0, 'z': 1.0}
}

print("Mengirim perintah gerak ke robot...")
publisher.publish(roslibpy.Message(gerak_maju))
time.sleep(1)

client.terminate()
```
Jalankan file tersebut. Anda akan melihat kura-kura di layar PC Lokal Anda bergerak. Anda telah berhasil mengontrol ROS2 melalui WebSockets!

---

## Latihan Mandiri
1. Buatlah script Python yang menerima input dari keyboard pengguna (`w`, `a`, `s`, `d`) di terminal non-ROS2 lalu menerjemahkannya menjadi publish ke `/turtle1/cmd_vel` via `roslibpy` untuk mengontrol robot.
2. Cobalah buat file HTML sederhana (gunakan library JavaScript `roslibjs`) yang menampilkan posisi X, Y robot di halaman web browser, lalu letakkan file HTML tersebut di VPS agar bisa diakses secara publik.
3. Pelajari opsi peluncuran `rosbridge_websocket_launch.xml`. Bagaimana cara menambahkan autentikasi sederhana agar rosbridge tidak bisa diakses sembarang orang di internet?

---

## Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| Script Python macet / Timeout / Connection Refused | Port 9090 belum terbuka di firewall VPS. | Cek status firewall dengan `sudo ufw status`. Pastikan port 9090 TCP diizinkan. |
| `KeyError` atau message tidak berefek | Tipe message di roslibpy (baris 3) salah eja format ROS2. | Pastikan menggunakan nama lengkap dengan `/msg/`: `geometry_msgs/msg/Twist` (ROS2), BUKAN `geometry_msgs/Twist` (ROS1). |
| Rosbridge di VPS error atau command not found | Environment variabel ROS2 tidak di-source. | Lakukan `source /opt/ros/jazzy/setup.bash` sebelum mengetik `ros2 launch`. |
| Topik terbaca di VPS, tapi rosbridge error serialization | Data topic terlalu kompleks/besar untuk JSON. | Gunakan kompresi CBOR (lebih advance) atau downgrade ukuran data payload. |

---
*Lanjut ke [Modul 3: Koneksi Native WAN dengan Zenoh Bridge](modul-03-zenoh.md)*
