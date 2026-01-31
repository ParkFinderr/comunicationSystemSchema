# 📡 ParkFinder Backend API, Websocket & IoT Protocols

Dokumentasi ini mencakup spesifikasi **RESTful API**, **WebSocket Events**, dan **MQTT Topics** untuk sistem ParkFinder.
Dokumen ini menjadi acuan utama bagi Tim Frontend (Mobile & Web) dan Tim IoT Engineer.

> **Base URL:** `BASE URL MENYUSUL`
> **Auth Header:** `Authorization: Bearer <jwt_token>`

---

## 🔌 1. RESTful API Endpoints

### 🔐 Tabel Modul Autentikasi
Menangani proses registrasi, login, dan logout sesi pengguna aplikasi.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/register` | Public | Registrasi akun baru (Sync dengan Firebase Auth). |
| `POST` | `/auth/login` | Public | Login pengguna & simpan **FCM Token** (untuk notifikasi). |
| `POST` | `/auth/logout` | User | Hapus sesi dan FCM Token dari database. |

<br>

### 🎫 Tabel Modul Akses & Tiket (Gate System)
**[CORE LOGIC]** Menangani validasi tiket fisik untuk izin akses "On-Site".

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `POST` | `/gate/generate-ticket` | **Admin/Mesin** | **[Printer]** Backend men-generate ID Tiket unik untuk dicetak mesin karcis. |
| `POST` | `/access/verify` | Public | **[Scan QR]** User memindai tiket fisik.<br>1. Mengikat tiket ke User ID (jika Login).<br>2. Membuat *Guest Session* (jika Tamu).<br>3. Membuka akses fitur reservasi. |
| `GET` | `/access/active-ticket` | User/Tamu | Mengecek apakah perangkat memiliki sesi tiket yang sedang aktif. |

<br>

### 👤 Tabel Modul Manajemen Pengguna
Mengelola data profil dan kendaraan.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `GET` | `/users/profile` | User | Mengambil detail data pengguna yang sedang login. |
| `PUT` | `/users/profile` | User | Mengubah data profil pengguna. |
| `POST` | `/users/vehicles`| User | Menambahkan data kendaraan baru. |
| `DELETE`| `/users/vehicles/:id`| User | Menghapus kendaraan. |
| `GET` | `/admin/users` | **Admin** | Monitoring seluruh pengguna terdaftar. |
| `DELETE`| `/admin/users/:id` | **Admin** | Ban/Hapus pengguna. |

<br>

### 🅿️ Tabel Modul Manajemen Lahan Parkir
Menampilkan informasi gedung dan slot.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `GET` | `/areas` | Public | Menampilkan daftar gedung dan sisa kuota. |
| `GET` | `/areas/:id/slots`| Public | **[Snapshot]** Data awal status slot saat aplikasi dibuka. |
| `POST` | `/areas/slots` | **Admin** | Menambahkan slot baru. |
| `PATCH`| `/areas/slots/:id/status`| **Admin** | Mengubah status slot menjadi `'maintenance'` (Rusak). |

<br>

### 📝 Tabel Modul Reservasi & Transaksi
Menangani alur booking, swap, cancel, dan konfirmasi IoT.
**Syarat:** User harus memiliki Tiket Valid (`ticketId`).

| Method | Endpoint | Hak Akses | Deskripsi & Trigger |
| :--- | :--- | :--- | :--- |
| `POST` | `/reservations` | **Ticket Holder** | **[Booking]** Input: `slotId`, `ticketId`. Validasi tiket sebelum kunci slot. |
| `GET` | `/reservations/:id` | Public | Melihat detail tiket aktif (Timer, Biaya). |
| `PUT` | `/reservations/:id/swap` | **Ticket Holder** | **[Ganti Slot]** Pindah slot tanpa membatalkan sesi tiket. |
| `PATCH`| `/reservations/:id/cancel` | **Ticket Holder** | **[Batal Manual]** User membatalkan pesanan. Slot kembali hijau. |
| `PATCH`| `/reservations/:id/arrive` | **Ticket Holder** | **[Tombol "Sudah Sampai"]** Validasi Sensor IoT.<br>📡 **IoT:** Kirim `buzzerOff`. |
| `PATCH`| `/reservations/:id/complete`| **Ticket Holder** | **[Tombol "Keluar"]** Selesai parkir & tutup tiket.<br>📡 **IoT:** Kirim `reset` state. |
| `GET` | `/users/:userId/reservations`| **User Only** | **[History]** Riwayat parkir User (Tamu tidak akses ini). |

---

## ⚡ 2. WebSocket Events (Real-Time)
Digunakan oleh Frontend untuk update UI tanpa refresh halaman.

**Connection URL:** `[WS URL MENYUSUL]`

### 📥 Listen (Server ➔ Client)

| Event Name | Target | Payload Contoh | Deskripsi |
| :--- | :--- | :--- | :--- |
| `slotUpdate` | Broadcast | `{ "slotId":"A1", "status":"occupied" }` | Update warna denah real-time (Hijau/Kuning/Merah). |
| `bookingTimer`| Unicast | `{ "timeLeftSeconds": 1500 }` | Hitung mundur batas waktu **30 Menit** (On-Site Rule). |
| `forceRelease`| Unicast | `{ "reason": "timeout" }` | Notifikasi auto-cancel jika waktu habis. |
| `adminStats` | Admin Web | `{ "occupancy": "85%", "alerts": 1 }` | Update dashboard monitoring admin. |

---

## 🤖 3. MQTT Specifications (For IoT Team)
Protokol komunikasi antara Hardware (ESP32) dan Server Cloud.

**Broker:** `[MQTT BROKER URL MENYUSUL]`

### 📡 Topics & Payloads

| Arah | Topik | Payload | Logika Backend / Aksi |
| :--- | :--- | :--- | :--- |
| **IoT ➔ Server** | `parkfinder/sensor/{area}/{slot}` | `0` (Kosong)<br>`1` (Ada) | **Single Source of Truth:** Validasi fisik keberadaan mobil. |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `buzzerOn` | **Peringatan Anomali:**<br>Dikirim jika:<br>1. Parkir Liar (Sensor=1, App=Available).<br>2. Salah Parkir (Sensor=1, App=Reserved by Other). |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `buzzerOff`| **Validasi Sukses:**<br>Dikirim saat user valid tekan tombol "Sudah Sampai". |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `reset` | **Reset:**<br>Dikirim saat user "Keluar" untuk reset state hardware. |

---

## 🔄 4. Alur Logika Sistem (System Flows)

Berikut adalah skenario logika backend dalam menangani kasus di lapangan.

### ✅ Skenario Normal
1. User ambil tiket fisik -> Scan QR (`POST /access/verify`).
2. User booking slot -> Status Slot: `'reserved'` (Kuning).
3. User parkir -> Sensor detect `1`.
4. User tekan "Sudah Sampai" -> Status Slot: `'occupied'` (Merah).

### 🔄 Skenario Ganti Slot (Swap)
1. User sudah booking Slot A, tapi ingin pindah.
2. User pilih Slot B -> Klik "Pindah Sini" (`PUT /swap`).
3. **Backend:** Slot A jadi `'available'`, Slot B jadi `'reserved'`.

### ⏱️ Skenario Auto-Cancel (Timeout)
1. User booking tapi tidak parkir dalam **30 Menit**.
2. **Backend:** Otomatis batalkan reservasi.
3. Kirim event WS `forceRelease` ke aplikasi user.
4. Slot kembali `'available'`.

### ⚠️ Skenario Anomali (Salah Parkir)
1. User A booking Slot 01.
2. User B (orang lain) parkir di Slot 01.
3. **Sensor:** Detect `1`.
4. **Backend:** Cek reservasi tidak cocok -> Kirim MQTT `buzzerOn` (Alarm Bunyi).
5. User A tidak bisa check-in sampai slot kosong atau pindah slot.

### 🚫 Skenario Parkir Liar
1. Slot 02 kosong (`available`).
2. Ada mobil masuk tanpa booking -> Sensor detect `1`.
3. **Backend:** Tandai slot `'occupied'` (Merah - Unauthorized).
4. Kirim MQTT `buzzerOn`.