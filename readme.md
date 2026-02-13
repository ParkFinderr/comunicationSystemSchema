# 📡 ParkFinder Backend API, Websocket & IoT Protocols

Dokumentasi ini mencakup spesifikasi **RESTful API**, **WebSocket Events**, dan **MQTT Topics** untuk sistem ParkFinder.
Dokumen ini menjadi acuan utama bagi Tim Frontend (Mobile & Web) dan Tim IoT Engineer.

> **Base URL API:** `BASE URL MENYUSUL`
> **WebSocket Events:** `WS URL MENYUSUL`
> **IoT Protocol:** `MQTT BROKER URL`
> **Auth Header:** `Authorization: Bearer <jwt_token>`

---

## 🏗️ Arsitektur Sistem

| Komponen | Teknologi | Fungsi Utama |
| :--- | :--- | :--- |
| **API Service** | Node.js, Express, Firebase | Menangani User Auth, Booking Logic, CRUD Database, Cron Jobs. |
| **Realtime Service** | Node.js, Socket.io, Aedes/Mosquitto | Gateway untuk IoT (MQTT) dan update live ke Frontend (WebSocket). |
| **Database** | Google Firestore | Menyimpan data user, slot, reservasi, dan log transaksi. |
| **Message Broker** | Redis | Jalur komunikasi internal (Pub/Sub) antar API Service dan Realtime Service. |
| **IoT Protocol** | MQTT | Jalur komunikasi data sensor dan kontrol aktuator (Lampu/Buzzer). |

---

## 🔌 1. RESTful API Endpoints

### ⏬ Tabel Modul Download Aplikasi Mobile
Menangani proses download aplikasi mobile.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `GET` | `/system/app/download` | Public | Mendowload aplikasi parkfinder. |

<br>

### 🔐 Tabel Modul Autentikasi
Menangani proses registrasi, login, dan logout sesi pengguna aplikasi.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/register` | Public | Registrasi akun baru (Sync dengan Firebase Auth). |
| `POST` | `/auth/login` | Public | Login pengguna & simpan **FCM Token** (untuk notifikasi). |
| `POST` | `/auth/logout` | User | Hapus sesi dan FCM Token dari database. |

<br>

### 🎫 Tabel Modul Akses & Tiket
Menangani validasi tiket fisik untuk izin akses "On-Site".

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `POST` | `/gate/generateTicket` | **Admin/Mesin** | Backend men-generate ID Tiket unik untuk dicetak mesin karcis. |
| `POST` | `/access/verify` | Public | Menerima ticketId dari hasil scan, menambahkan tiket ke akun user dan membuat sesi untuk tamu. |
| `GET` | `/access/activeTicket` | User | Mengecek apakah perangkat memiliki sesi tiket yang sedang aktif. |
| `GET` | `/access/activeTicket?guestSessionId={{guestSessionId}}` | Tamu | Mengecek apakah perangkat memiliki sesi tiket yang sedang aktif. |

<br>

### 👤 Tabel Modul Manajemen Pengguna dan kendaraan
Mengelola data profil dan kendaraan.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `GET` | `/users/profile` | User | Mengambil detail data pengguna yang sedang login. |
| `PUT` | `/users/profile` | User | Mengubah data profil pengguna. |
| `POST` | `/users/vehicles`| User | Menambahkan data kendaraan baru. |
| `DELETE`| `/users/vehicles/:id`| User | Menghapus kendaraan menggunakan plat nomor. |
| `GET` | `/admin/users` | Admin | Monitoring seluruh pengguna terdaftar. |
| `GET` | `/admin/users/:id` | Admin | Mengambil id user. |
| `DELETE`| `/admin/users/:id` | Admin | Hapus pengguna. |

<br>

### 🅿️ Tabel Modul Manajemen Lahan Parkir
Menampilkan informasi gedung dan slot.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `GET` | `/areas` | Public | Menampilkan daftar gedung dan sisa kuota (Real-time Counter). |
| `POST` | `/areas` | Admin | Mendaftarkan gedung/area parkir baru. |
| `GET` | `/areas/:id` | Public | Menampilkan detail satu gedung (untuk pre-fill form edit). |
| `PUT` | `/areas/:id` | Admin | Mengubah nama/alamat gedung. |
| `DELETE`| `/areas/:id` | Admin | Menghapus gedung (Hanya bisa jika gedung sudah kosong/tanpa slot). |
| `GET` | `/areas/:id/slots`| Public |  Data awal status slot di area tertentu saat aplikasi dibuka. |
| `GET` | `/areas/slots/:id`| Public | Menampilkan detail satu slot (Info fisik & Status). |
| `PUT` | `/areas/slots/:id` | Admin | Mengubah detail slot. |
| `DELETE`| `/areas/slots/:id`| Admin | Menghapus slot permanen jika slot sedang terisi/booking tidak bisa di hapus. |
| `POST` | `/areas/slots` | Admin | Menambahkan slot baru. |

<br>

### 📝 Tabel Modul Reservasi
Menangani alur booking, swap, cancel, dan konfirmasi IoT.
**Syarat:** User/Tamu harus memiliki Tiket Valid.

| Method | Endpoint | Hak Akses | Deskripsi & Trigger |
| :--- | :--- | :--- | :--- |
| `POST` | `/reservations` | User/Tamu Memiliki Tiket | Memesan slot parkir yang tersedia |
| `GET` | `/reservations/:id` | Public | Melihat detail tiket aktif berdasarkan id. |
| `PUT` | `/reservations/:reservationId/swap` | User/Tamu Memiliki Tiket | Pindah slot tanpa membatalkan sesi tiket. |
| `PATCH`| `/reservations/:reservationId/cancel` | User/Tamu Memiliki Tiket | User membatalkan pesanan. |
| `PATCH`| `/reservations/:reservationId/arrive` | User/Tamu Memiliki Tiket | User menekan tombol telah sampai. |
| `PATCH`| `/reservations/:reservationId/complete`| User/Tamu Memiliki Tiket | User menekan tombol keluar |
| `GET` | `/users/:userId/reservations`| User | Riwayat parkir User (Tamu tidak akses ini). |

---

## ⚡ 2. WebSocket Events (Real-Time)

### 📥 Listen (Server ➔ Client)

| Event Name | Target | Payload Contoh | Deskripsi |
| :--- | :--- | :--- | :--- |
| `slotUpdate` | Broadcast | `{"areaId":"G1", "slotId":"A01", "status":"reserved"}` | Update warna denah real-time (Hijau/Kuning/Merah). |
| `bookingTimer`| Unicast | `{ "timeLeftSeconds": 1500 }` | Hitung mundur batas waktu **30 Menit** (On-Site Rule). |
| `forceRelease`| Unicast | `{ "reason": "timeout" }` | Notifikasi auto-cancel jika waktu habis. |
| `adminStats` | Admin Web | `{"totalVehicles": 45, "occupancy": "80%", "alerts": 2}` | Update dashboard monitoring admin. |

---

## 🤖 3. MQTT Specifications (For IoT Team)

### 📡 Topics & Payloads

| Arah | Topik | Payload | Logika Backend / Aksi Hardware |
| :--- | :--- | :--- | :--- |
| **IoT ➔ Server** | `parkfinder/sensor/{area}/{slot}` | `0` / `1` | Laporan sensor IR/Ultrasonic. |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `setReserved` | **Aksi Booking:** LED Kuning, LCD "RESERVED". |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `setAvailable`| **Aksi Cancel/Swap/Keluar:** LED Hijau, LCD "AVAILABLE". |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `setAccupied` | **Aksi Check-in:** LED Merah, LCD "OCCUPIED". |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `buzzerOn` | **Aksi Anomali:** LED Merah Kedip, LCD "ALERT!", Buzzer ON. |

---

## 🔄 Alur Logika Sistem (System Flows & Scenarios)

**12 Skenario Logika** yang menangani alur normal hingga kasus anomali (kecurangan/error) di lapangan.

### ✅ Skenario Normal

#### 1️⃣ Masuk & Scan Tiket
1.  User mengambil tiket fisik/digital.
2.  User scan QR Code tiket via App (`POST /access/verify`).
3.  **Backend:** Validasi tiket. Jika valid, status tiket menjadi `claimed`. User mendapat akses ke menu denah parkir.

#### 2️⃣ Booking Slot
1.  User memilih slot **Hijau** (Available) -> Tekan "Booking" (`POST /reservations`).
2.  **Backend API:**
    * Validasi: Apakah user punya tiket aktif? Apakah slot masih kosong?
    * Update DB: Slot `reserved`, Reservasi `pending`.
    * **Redis Pub:** Kirim perintah `reserveSlot`.
3.  **Realtime Service:** Menerima perintah Redis -> **MQTT Pub:** `setReserved` (Lampu Kuning).
4.  **WebSocket:** Broadcast ke semua user lain bahwa slot ini sedang dipesan.

#### 3️⃣ Konfirmasi Kedatangan (Check-in)
1.  User memarkirkan mobil di slot yang dipesan.
2.  **Sensor IoT:** Mendeteksi objek -> Kirim `1` ke MQTT.
3.  User tekan tombol "Saya Sudah Sampai" (`PATCH /arrive`).
4.  **Backend API:**
    * **VALIDASI FISIK:** Cek database `sensorStatus`. Jika `0`, **TOLAK** request (mencegah check-in palsu).
    * Jika `1` (Ada mobil): Update DB Slot `occupied`, Reservasi `active`.
    * **Redis Pub:** Kirim perintah `occupySlot`.
5.  **Realtime Service:** **MQTT Pub:** `setOccupied` (Lampu Merah).

#### 4️⃣ Ganti Slot (Swap)
*User sudah booking Slot A, ingin pindah ke Slot B.*
1.  User pilih Slot B -> "Pindah ke sini" (`PUT /swap`).
2.  **Backend API:**
    * Slot A (Lama): Reset jadi `available` -> **Redis:** `leaveSlot` (Lampu Hijau).
    * Slot B (Baru): Set jadi `reserved` -> **Redis:** `reserveSlot` (Lampu Kuning).
    * Update data reservasi user dengan Slot ID baru.

#### 5️⃣ Pembatalan Manual (Cancel)
*User berubah pikiran dan membatalkan pesanan.*
1.  User tekan "Batalkan Pesanan" (`PATCH /cancel`).
2.  **Backend API:**
    * Ubah status reservasi: `cancelled`.
    * Reset Slot: `available`.
    * **Redis Pub:** `cancelSlot`.
3.  **Realtime Service:** **MQTT Pub:** `setAvailable` (Lampu Hijau).

#### 6️⃣ Checkout (Keluar Normal)
1.  User akan keluar -> Tekan "Selesai Parkir" (`PATCH /complete`).
2.  **Backend API:**
    * Tutup sesi reservasi (`completed`) & Tiket (`closed`).
    * Reset Slot `available`.
    * **Redis Pub:** `leaveSlot`.
3.  **Realtime Service:** **MQTT Pub:** `setAvailable` (Lampu Hijau).

---

### ⚠️ Skenario Otomatis (Cron Job & Timer)

#### 7️⃣ Auto-Cancel (Booking Expired)
*User booking tapi tidak datang dalam 30 menit.*
1.  **Cron Job:** Cek reservasi `pending` yang `created_at > 30 menit`.
2.  **Action:**
    * Ubah status reservasi: `cancelled`.
    * Reset Slot: `available`.
    * **Redis Pub:** `cancelSlot` (Lampu Hijau).

#### 8️⃣ Auto-Checkout (Lupa Tekan Tombol)
*User pergi (Sensor 0) tapi lupa tekan "Selesai Parkir".*
1.  **Sensor Listener / Cron:** Mendeteksi Slot `occupied` tapi sensor `0` selama > 2 menit.
2.  **Action:**
    * Otomatis selesaikan reservasi (`completed`).
    * Kirim notifikasi ke user: "Sesi parkir Anda diakhiri otomatis."
    * Reset Slot: `available` (Lampu Hijau).

---

### 🚨 Skenario Anomali & Keamanan

#### 9️⃣ Salah Parkir (Penyusup di Slot Booked)
*User A booking Slot 01, tapi User B (orang lain) parkir di situ.*
1.  **Kondisi:** Slot status `booked` (Kuning).
2.  **Sensor:** Mendeteksi mobil (`1`). User A **BELUM** check-in.
3.  **Backend Logic:**
    * Mendeteksi ada mobil masuk saat status `booked` tanpa trigger API `/arrive`.
4.  **Action:**
    * **MQTT Pub:** `buzzerOn` (Lampu Merah Kedip + Suara Alarm).
    * **Admin Alert:** Kirim notifikasi "Penyusup di Slot 01".

#### 🔟 Parkir Liar (Masuk Tanpa Booking)
*Slot kosong (Available), tiba-tiba ada mobil masuk.*
1.  **Kondisi:** Slot status `available` (Hijau).
2.  **Sensor:** Mendeteksi mobil (`1`).
3.  **Action:**
    * Update DB Slot: `occupied` (agar di map jadi merah/unavailable).
    * Label di App: **"UNAUTHORIZED"**.
    * **MQTT Pub:** `buzzerOn` (Alarm Nyala).

#### 1️⃣1️⃣ Ghost Swap (Celah Keamanan)
*Mobil A keluar, Mobil B langsung masuk dalam hitungan detik untuk menghindari sistem.*
1.  **Kondisi:** Slot `occupied` (Mobil A).
2.  **Event:** Sensor berubah `1` -> `0` (Mobil A keluar) -> `1` (Mobil B masuk) dalam waktu < 1 menit.
3.  **Backend Logic:**
    * Mendeteksi perubahan cepat ini sebagai anomali.
    * Sistem tidak menganggap Mobil B adalah Mobil A.
4.  **Action:**
    * **Paksa Selesai:** Sesi Mobil A otomatis di-checkout (`completed`).
    * **Block Mobil B:** Slot dianggap `occupied` oleh "Hantu/Liar".
    * **Alarm:** Nyalakan Buzzer untuk mengusir Mobil B.

#### 1️⃣2️⃣ Maintenance
*Admin mematikan slot untuk perbaikan.*
1.  Admin set status `maintenance`.
2.  **MQTT Pub:** `maintenanceSlot` (Lampu Mati/Tanda Khusus).
3.  User tidak bisa memilih slot ini di aplikasi.