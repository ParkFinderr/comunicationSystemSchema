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
| `POST` | `/gate/generateTicket` | **Admin/Mesin** | **[Printer]** Backend men-generate ID Tiket unik untuk dicetak mesin karcis. |
| `POST` | `/access/verify` | Public | **[Scan QR]** User memindai tiket fisik.<br>1. Mengikat tiket ke User ID (jika Login).<br>2. Membuat *Guest Session* (jika Tamu).<br>3. Membuka akses fitur reservasi. |
| `GET` | `/access/activeTicket` | User/Tamu | Mengecek apakah perangkat memiliki sesi tiket yang sedang aktif. |

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
| `PUT` | `/areas/slots/:id` | **Admin** | Mengubah detail slot (Nama slot, Lantai, dll). |
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
| `slotUpdate` | Broadcast | `{"areaId":"G1", "slotId":"A01", "status":"reserved"}` | Update warna denah real-time (Hijau/Kuning/Merah). |
| `bookingTimer`| Unicast | `{ "timeLeftSeconds": 1500 }` | Hitung mundur batas waktu **30 Menit** (On-Site Rule). |
| `forceRelease`| Unicast | `{ "reason": "timeout" }` | Notifikasi auto-cancel jika waktu habis. |
| `adminStats` | Admin Web | `{"totalVehicles": 45, "occupancy": "80%", "alerts": 2}` | Update dashboard monitoring admin. |

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

Berikut adalah **10 Skenario Lengkap** logika backend dalam menangani kasus di lapangan.

### 1️⃣ Skenario Masuk & Scan Tiket
1. User mengambil tiket fisik dari mesin karcis.
2. User memindai QR Code tiket menggunakan aplikasi (`POST /access/verify`).
3. **Backend:** Mengikat `ticketId` ke akun user. Tiket status menjadi `'claimed'`.
4. User mendapat akses ke menu denah parkir.

### 2️⃣ Skenario Booking Slot (Normal)
1. User memilih slot kosong (Hijau) di aplikasi.
2. User tekan "Booking" (`POST /reservations`).
3. **Backend:** Mengubah status slot menjadi `'reserved'` (Kuning).
4. **WebSocket:** Update peta ke seluruh user lain agar slot tersebut tidak dipilih orang lain.

### 3️⃣ Skenario Konfirmasi Kedatangan (Check-in)
1. User memarkirkan mobil di slot yang dipesan.
2. **Sensor IoT:** Mendeteksi objek dan mengirim `1` ke server.
3. User menekan tombol "Sudah Sampai" di aplikasi (`PATCH /arrive`).
4. **Backend:** Memvalidasi `sensorStatus == 1`.
5. Jika valid, status slot berubah menjadi `'occupied'` (Merah).
6. **MQTT:** Server mengirim `buzzerOff` ke alat untuk mematikan potensi alarm.

### 4️⃣ Skenario Ganti Slot (Swap)
*User sudah booking Slot A, tapi ingin pindah ke Slot B.*
1. User memilih Slot B di aplikasi.
2. Pilih opsi "Pindah ke sini" (`PUT /swap`).
3. **Backend:**
   - Melepas Slot A (Kembali Hijau/'Available').
   - Mengunci Slot B (Menjadi Kuning/'Reserved').
   - Memperbarui data reservasi tanpa menghapus sesi tiket fisik.

### 5️⃣ Skenario Pembatalan Manual
*User berubah pikiran dan tidak jadi parkir.*
1. User menekan tombol "Batalkan Pesanan" (`PATCH /cancel`).
2. **Backend:**
   - Mengubah status reservasi menjadi `'cancelled'`.
   - Mengubah status slot kembali menjadi `'available'` (Hijau).
3. **Catatan:** Tiket fisik **TIDAK HANGUS**. User masih bisa melakukan booking ulang di slot lain selama belum keluar gerbang.

### 6️⃣ Skenario Auto-Cancel (Timeout 30 Menit)
*User booking tapi tidak kunjung parkir.*
1. **Backend Timer:** Menghitung waktu sejak booking dibuat.
2. Jika `(Waktu Sekarang - Waktu Booking) > 30 Menit`:
3. **Backend Action:**
   - Otomatis membatalkan reservasi.
   - Mengirim WebSocket `forceRelease` ke aplikasi user.
   - Slot kembali menjadi `'available'` (Hijau).

### 7️⃣ Skenario Checkout (Keluar)
1. User menuju pintu keluar.
2. User menekan tombol "Selesai Parkir" (`PATCH /complete`) atau scan tiket keluar.
3. **Backend:**
   - Menutup sesi tiket (`'closed'`).
   - Melepas `activeTicketId` dari profil user.
   - Mereset slot menjadi `'available'` dan `sensorStatus` dianggap `0`.
4. **MQTT:** Mengirim perintah `reset` ke perangkat IoT slot tersebut.

### 8️⃣ Skenario Anomali: Salah Parkir
*User A booking Slot 01, tapi User B (orang lain) parkir di Slot 01.*
1. **Sensor IoT:** Mendeteksi mobil (`1`) di Slot 01.
2. **Backend:** Mengecek reservasi aktif Slot 01 adalah milik User A, tapi User A belum konfirmasi "Sudah Sampai".
3. **Action:**
   - Menganggap mobil tersebut adalah penyusup.
   - Mengirim MQTT `buzzerOn` (Alarm berbunyi).
   - Mengirim notifikasi `alerts` ke Admin Dashboard.

### 9️⃣ Skenario Anomali: Parkir Liar
*Slot 02 kosong (Available), tiba-tiba ada mobil masuk tanpa booking.*
1. **Sensor IoT:** Mendeteksi mobil (`1`) di Slot 02.
2. **Backend:** Mengecek status Slot 02 adalah `'available'` (tidak ada yang booking).
3. **Action:**
   - Mengubah tampilan App menjadi `'occupied'` (Merah) dengan label **"UNAUTHORIZED"**.
   - Mengirim MQTT `buzzerOn`.

### 🔟 Skenario Maintenance (Perbaikan)
*Slot rusak atau sedang dicat ulang.*
1. Admin login ke Web Dashboard.
2. Admin mengubah status slot menjadi "Maintenance" (`PATCH /status`).
3. **Backend:** Mengupdate `appStatus` menjadi `'maintenance'`.
4. **Frontend:** Slot tampil berwarna Abu-abu dan tombol booking dinonaktifkan (disable).