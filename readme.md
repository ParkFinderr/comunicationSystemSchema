# 📡 ParkFinder Backend API, Websocket & IoT Protocols

Dokumentasi ini mencakup spesifikasi **RESTful API**, **WebSocket Events**, dan **MQTT Topics** untuk sistem ParkFinder.
Dokumen ini menjadi acuan utama bagi Tim Frontend (Mobile & Web) dan Tim IoT Engineer.

<!-- **Base URL:** `nanti menyusul` -->
**Auth Header:** `Authorization: Bearer <jwt>`

---

## 🔌 1. RESTful API Endpoints

### 🔐 A. Authentication & User
Mengelola sesi pengguna dan data profil.

| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/register` | Public | Mendaftarkan user baru & menyimpan data awal di Firestore. |
| `POST` | `/auth/login` | Public | Verifikasi token Firebase, cek role, & **simpan FCM Token** (untuk notif). |
| `POST` | `/auth/logout` | User | Hapus sesi & **hapus FCM Token** agar notifikasi berhenti. |
| `GET` | `/users/profile` | User | Mengambil data profil & list kendaraan user. |
| `POST` | `/users/vehicles`| User | Menambah plat nomor kendaraan baru. |

### 🅿️ B. Parking Areas (Public Info)
Digunakan untuk menampilkan data di "Halaman Utama" aplikasi/web.

| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/areas` | Public | List semua gedung & sisa slot kosong (*Available Counter*). |
| `GET` | `/areas/:id/slots`| Public | **[Snapshot]** Mengambil status warna-warni slot (Merah/Hijau) saat aplikasi *pertama dibuka*. Update selanjutnya via WebSocket. |

### 🎟️ C. Reservations (Core Logic)
Menangani alur booking, check-in, dan check-out.

| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/reservations` | Public | Booking slot. <br>**Note:** Body request `type` bisa `"APP"` (User) atau `"GUEST"` (Tamu Web). |
| `PATCH`| `/reservations/:id/arrive` | User | **[Tombol "Sudah Sampai"]** <br>✅ Validasi Lokasi. <br>📡 **Trigger IoT:** Kirim `buzzerOff` ke alat. |
| `PATCH`| `/reservations/:id/complete`| User | **[Tombol "Keluar"]** <br>✅ Selesaikan sesi. <br>📡 **Trigger IoT:** Kirim `reset` ke alat (agar sensor tidak bunyi saat mobil mundur). |
| `GET` | `/reservations/history` | User | List riwayat parkir user. |

---

## ⚡ 2. WebSocket Events (Real-Time)
Digunakan oleh Frontend untuk update UI tanpa refresh halaman.

**Connection URL:** `wss://api.parkfinder.id`

### 📥 Listen (Server ➔ Client)

| Event Name | Payload Example | Description |
| :--- | :--- | :--- |
| `slotUpdate` | `{ "areaId": "G1", "slotId": "A01", "status": "occupied" }` | Dikirim saat ada perubahan status slot (oleh Sensor atau Booking). **Frontend wajib update warna denah.** |
| `bookingTimer`| `{ "timeLeft": 280 }` | Hitung mundur sisa waktu check-in (khusus user yang sedang booking). |

---

## 🤖 3. MQTT Specifications (For IoT Team)
Protokol komunikasi antara Hardware (ESP32/STM32) dan Server Cloud.

**Broker:** `tcp://mqtt.parkfinder.id:1883` (GCE)

### 📡 Topics & Payloads

| Direction | Topic Format | Payload | Action / Logic |
| :--- | :--- | :--- | :--- |
| **IoT ➔ Server** | `parkfinder/sensor/{area}/{slot}` | `0` | **Slot Kosong.** Server update DB jadi hijau. |
| **IoT ➔ Server** | `parkfinder/sensor/{area}/{slot}` | `1` | **Ada Mobil.** Server cek reservasi. |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `buzzerOn` | **ALARM!** Mobil masuk tapi belum booking/check-in (Parkir Liar). |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `buzzerOff`| **SILENT.** User valid sudah menekan tombol "Sampai" di aplikasi. |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `reset` | **RESET STATE.** User valid menekan tombol "Keluar". Abaikan pembacaan sensor sesaat (saat mobil mundur). |

> **⚠️ Note for IoT **
> Pastikan perangkat *subscribe* ke topik `parkfinder/control/...` segera setelah nyala untuk menerima perintah Buzzer/Reset dari server.

---