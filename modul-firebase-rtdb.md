# Modul Tambahan: Firebase Realtime Database (RTDB) untuk Cloud Robotics

**Mata Kuliah:** RE405 Cloud Robotics  
**Referensi Resmi:**
- [Firebase RTDB Docs](https://firebase.google.com/docs/database)
- [Firebase Security Rules](https://firebase.google.com/docs/database/security)
- [Firebase Auth Custom Claims](https://firebase.google.com/docs/auth/admin/custom-claims)

---

## Tujuan Pembelajaran
- Mahasiswa memahami konsep Firebase Realtime Database sebagai solusi cloud state management untuk robot.
- Mahasiswa mampu membedakan Firebase RTDB dengan MQTT untuk skenario cloud robotics.
- Mahasiswa memahami Security Rules dan Firebase Auth untuk multi-robot fleet.
- Mahasiswa mengetahui pola arsitektur hybrid (MQTT + RTDB + DDS) skala besar.

---

## Teori Singkat (15 menit)

### Apa itu Firebase Realtime Database (RTDB)?

Firebase Realtime Database adalah database NoSQL berbasis cloud milik Google yang menyimpan dan menyinkronkan data dalam format **JSON tree** secara real-time ke semua client yang terhubung. Berbeda dari REST API biasa, Firebase menggunakan **WebSocket persistent connection** — bukan HTTP polling berkala — sehingga setiap perubahan data langsung "dipush" ke semua client dalam milidetik.

**Karakteristik Utama:**
- **Real-time sync:** Perubahan data langsung diterima semua client yang aktif (WebSocket).
- **Offline-first:** SDK Firebase menyimpan cache lokal. Saat koneksi putus, app tetap bisa baca/tulis data lokal. Saat online kembali, data otomatis di-sync ke server.
- **JSON tree structure:** Seluruh data disimpan sebagai satu pohon JSON besar.
- **Built-in auth & security:** Terintegrasi dengan Firebase Authentication dan aturan keamanan berbasis path.

**Diagram Arsitektur:**
```text
Robot (Python SDK)          Firebase RTDB (Cloud)        Dashboard (Web/Mobile)
┌─────────────────┐          ┌──────────────────┐         ┌──────────────────┐
│  firebase-admin │ WebSocket│  JSON Database   │WebSocket│  Firebase JS SDK │
│  Python SDK     │◄────────►│  /robots/{id}/   │◄───────►│  Real-time UI    │
│  (telemetry,    │          │    telemetry/    │         │  Monitoring      │
│   commands)     │          │    commands/     │         │  Dashboard       │
└─────────────────┘          │    status/       │         └──────────────────┘
                             │    config/       │
                             └──────────────────┘
```

---

## Bagian 1: Setup Firebase RTDB

### 1.1 Membuat Project Firebase
1. Buka [console.firebase.google.com](https://console.firebase.google.com)
2. Klik **Add project** → beri nama (misal: `re405-cloud-robot`)
3. Di sidebar kiri, pilih **Build → Realtime Database**
4. Klik **Create Database**, pilih region (us-central1 atau asia-southeast1)
5. Pilih mode awal: **Test Mode** (untuk development)

### 1.2 Install Firebase Admin SDK di Robot (Python)
Di PC robot/VPS yang menjalankan Python:
```bash
pip3 install firebase-admin
```

### 1.3 Service Account Key
Untuk akses dari server/robot (bukan browser):
1. Firebase Console → Project Settings → **Service Accounts**
2. Klik **Generate new private key** → simpan file JSON
3. Jangan commit file ini ke Git!

```python
import firebase_admin
from firebase_admin import credentials, db

# Initialize Firebase Admin SDK
cred = credentials.Certificate('/path/to/serviceAccountKey.json')
firebase_admin.initialize_app(cred, {
    'databaseURL': 'https://re405-cloud-robot-default-rtdb.firebaseio.com/'
})
```

---

## Bagian 2: Struktur Data JSON Tree untuk Multi-Robot Fleet

### 2.1 Prinsip Data Modeling RTDB
RTDB menyimpan data sebagai JSON tree. Struktur yang baik untuk fleet robot:

```json
{
  "robots": {
    "robot_001": {
      "status": {
        "online": true,
        "last_seen": 1720000000,
        "battery": 87.5
      },
      "telemetry": {
        "position": {"x": 1.23, "y": 4.56, "theta": 0.78},
        "velocity": {"linear": 0.5, "angular": 0.0}
      },
      "commands": {
        "cmd_vel": {"linear": {"x": 0.0}, "angular": {"z": 0.0}},
        "mode": "autonomous"
      },
      "config": {
        "max_speed": 1.0
      }
    }
  },
  "firmware": {
    "v2.1.0": {
      "url": "https://storage.googleapis.com/bucket/firmware-v2.1.0.bin",
      "checksum": "sha256:abc123...",
      "release_notes": "Bug fix: navigation"
    }
  }
}
```

**Mengapa path terpisah (`telemetry/`, `commands/`, `status/`)?**
- Telemetry ditulis **sangat sering** → pisahkan agar listener dashboard tidak kena flood update perintah.
- Commands ditulis **jarang** tapi penting → perlu security rules yang berbeda.
- Pemisahan path memudahkan penulisan **Security Rules** per path.

### 2.2 Menulis Data dari Robot (Python)
```python
import time
from firebase_admin import db

robot_id = "robot_001"
ref = db.reference(f'/robots/{robot_id}')

# Update status online
ref.child('status').update({
    'online': True,
    'last_seen': int(time.time()),
    'battery': 87.5
})

# Update telemetry posisi
ref.child('telemetry/position').set({
    'x': 1.23,
    'y': 4.56,
    'theta': 0.78
})
```

### 2.3 Membaca Command dari Robot (Listener)
```python
def on_command_received(event):
    """Dipanggil setiap ada perubahan di /commands"""
    print(f"Command baru: {event.data}")
    # Kirim ke ROS2 topic

# Pasang listener real-time
ref.child('commands').listen(on_command_received)
```

---

## Bagian 3: Presence System dengan `onDisconnect()`

### 3.1 Masalah Heartbeat Manual
Ketika robot kehilangan koneksi secara mendadak (mati listrik, putus internet), Firebase tidak langsung tahu robot sudah offline. Mekanisme heartbeat manual tidak reliabel karena robot mungkin tidak sempat kirim data sebelum mati.

### 3.2 Solusi: `onDisconnect()`
Firebase menyediakan operasi **"on-disconnect"** — instruksi yang **dieksekusi oleh server Firebase** secara otomatis ketika koneksi WebSocket client putus. Tidak perlu kode heartbeat di robot!

```javascript
// Di dashboard web (JavaScript SDK)
const statusRef = firebase.database().ref(`/robots/${robotId}/status`);

// Daftarkan: "saat koneksi putus, server jalankan ini"
statusRef.onDisconnect().update({
    online: false,
    last_seen: firebase.database.ServerValue.TIMESTAMP
});

// Set status online sekarang
statusRef.update({ online: true });
```

**Keunggulan:**
- Server Firebase yang mengeksekusi → tidak perlu heartbeat dari robot.
- Bekerja bahkan saat robot crash mendadak.
- `ServerValue.TIMESTAMP` = timestamp server, bukan waktu robot (lebih akurat).

---

## Bagian 4: Firebase Security Rules

### 4.1 Konsep Dasar
Security Rules adalah **aturan akses berbasis JSON** yang dievaluasi di **server Firebase** (bukan client). Tidak bisa di-bypass meskipun client diretas.

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "robots": {
      "$robotId": {
        "telemetry": {
          ".read": "auth != null",
          ".write": "auth.uid == $robotId"
        },
        "commands": {
          ".read": "auth.uid == $robotId",
          ".write": "auth != null && auth.token.role == 'operator'"
        },
        "status": {
          ".read": "auth != null",
          ".write": "auth.uid == $robotId"
        }
      }
    }
  }
}
```

**Penjelasan variabel:**
- `auth.uid`: ID unik user yang login.
- `$robotId`: wildcard path (nama folder robot).
- `auth.token.role`: custom claim yang di-set admin.
- Rules dievaluasi di server → aman dari client yang dikompromikan.

---

## Bagian 5: Firebase Auth + Custom Claims untuk Multi-Tenant

### 5.1 Masalah Multi-Tenant
Perusahaan A dan B pakai satu Firebase project. Bagaimana memastikan robot perusahaan A tidak bisa baca data perusahaan B?

### 5.2 Custom Claims
**Custom Claims** adalah metadata tambahan dalam token JWT Firebase yang di-set oleh **Admin SDK** (server-side). Client tidak bisa memanipulasi claims ini.

```python
# Di server admin (bukan di robot langsung)
from firebase_admin import auth

# Set custom claim untuk robot Company A
auth.set_custom_user_claims('robot_001_uid', {
    'fleet': 'companyA',
    'role': 'robot'
})

# Set custom claim untuk operator Company A
auth.set_custom_user_claims('operator_001_uid', {
    'fleet': 'companyA',
    'role': 'operator'
})
```

### 5.3 Security Rules dengan Custom Claims
```json
{
  "rules": {
    "fleet_data": {
      "$fleetId": {
        ".read": "auth.token.fleet == $fleetId",
        ".write": "auth.token.fleet == $fleetId && auth.token.role == 'operator'"
      }
    }
  }
}
```
Dengan ini, operator Company A (`auth.token.fleet == 'companyA'`) hanya bisa akses `fleet_data/companyA`, tidak bisa ke `fleet_data/companyB`.

---

## Bagian 6: OTA Firmware Update via Firebase RTDB

### 6.1 Mengapa Firebase untuk OTA Metadata?
- **Atomic write**: manifest firmware diupdate sekaligus untuk seluruh fleet.
- **Offline-first**: robot yang offline saat update dirilis akan dapat notifikasi saat online kembali.
- **Security rules**: hanya robot tertentu yang bisa akses versi firmware tertentu.

### 6.2 Workflow OTA
```text
Admin Server
    │  auth.set_custom_user_claims(robot_id, {'group': 'beta'})
    │  db.reference('/firmware/v2.1.0').set(manifest)
    ▼
Firebase RTDB
    │  Push update ke semua listener
    ▼
Robot (online)         Robot (offline)
    │                       │
    ▼                       ▼ (saat online kembali)
Download firmware      Download firmware
dari URL di manifest   dari URL di manifest
```

### 6.3 Contoh Manifest
```json
{
  "firmware": {
    "v2.1.0": {
      "url": "https://storage.googleapis.com/re405/firmware-v2.1.0.bin",
      "checksum": "sha256:abc123def456...",
      "target_group": "all",
      "mandatory": true,
      "released_at": 1720000000
    }
  }
}
```

---

## Bagian 7: Perbandingan RTDB vs MQTT

| Aspek | Firebase RTDB | MQTT (Mosquitto) |
|-------|--------------|------------------|
| **Paradigma** | State-based (last value wins) | Event-stream (setiap pesan penting) |
| **Protokol** | WebSocket (WSS/HTTPS) | TCP 1883 / TLS 8883 |
| **Offline support** | ✅ Built-in (SDK cache + auto-sync) | ❌ Manual |
| **Security** | ✅ Server-side rules + Firebase Auth | ⚠️ Username/password + TLS |
| **Query/Filter** | ✅ orderByChild, limitToLast | ❌ Hanya by topic |
| **Throughput** | ⚠️ ~100 writes/sec/client | ✅ Ribuan msg/sec |
| **Transaction** | ✅ Atomic transaction | ❌ Tidak ada |
| **Cocok untuk** | Command, config, state, OTA | Telemetry high-freq, sensor stream |

### 7.1 Rate Limit RTDB dan Solusinya
Firebase RTDB membatasi **~100 writes per detik per client**. Sensor IMU 100Hz tidak bisa dikirim langsung ke RTDB.

**Solusi:**
1. **Aggregate di edge**: kirim rata-rata/summary 1–10 Hz ke RTDB.
2. **Raw data ke MQTT + InfluxDB**: untuk time-series analysis.
3. **RTDB untuk state terkini saja**: posisi/status terakhir, bukan histori.

### 7.2 Arsitektur Hybrid Best-Practice
```text
Telemetry High-Freq  → MQTT Broker → InfluxDB/TimescaleDB
Command & State      → Firebase RTDB ← Dashboard
OTA Metadata         → Firebase RTDB → All Robots
Presence/Heartbeat   → Firebase onDisconnect() → RTDB
Robot-to-Robot LAN   → DDS (Zenoh/FastDDS) P2P
```

---

## Referensi
| Topik | Link |
|-------|------|
| Firebase RTDB Getting Started | https://firebase.google.com/docs/database/web/start |
| Security Rules | https://firebase.google.com/docs/database/security |
| Offline Capabilities & onDisconnect | https://firebase.google.com/docs/database/web/offline-capabilities |
| Admin SDK Python | https://firebase.google.com/docs/database/admin/start |
| Custom Claims | https://firebase.google.com/docs/auth/admin/custom-claims |

---

*Modul ini melengkapi ros2-vps-tutorial untuk RE405 Cloud Robotics. Firebase RTDB berperan sebagai lapisan **state management** dan **command & control** dalam arsitektur cloud robotics skala besar.*
