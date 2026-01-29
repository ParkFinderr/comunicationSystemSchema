# 📡 ParkFinder Backend API, Websocket & IoT Protocols

Dokumentasi ini mencakup spesifikasi **RESTful API**, **WebSocket Events**, dan **MQTT Topics** untuk sistem ParkFinder.
Dokumen ini menjadi acuan utama bagi Tim Frontend (Mobile & Web) dan Tim IoT Engineer.

**Base URL:** `base urlnya nanti menyusul`
**Auth Header:** `Authorization: Bearer <jwt>`

---

## 🔌 1. RESTful API Endpoints

### 🔐 Tabel Modul Autentikasi
Menangani proses registrasi, login, dan logout sesi pengguna.

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/register` | Public | Registrasi pengguna baru dan menerima ID Token dari Firebase setelah user sign up di App. |
| `POST` | `/auth/login` | Public | Login sinkronisasi pengguna dan menyimpan **FCM Token** (untuk notifikasi). |
| `POST` | `/auth/logout` | Public | Logout pengguna dan menghapus FCM Token dari database. |

<br>

### 👤 Tabel Modul Manajemen Pengguna & Kendaraan
Mengelola data profil, kendaraan, dan monitoring pengguna (Admin).

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `GET` | `/users/profile` | Pengguna | Mengambil detail data pengguna yang sedang login. |
| `PUT` | `/users/profile` | Pengguna | Mengubah data profil pengguna (Nama/No HP). |
| `POST` | `/users/vehicles`| Pengguna | Menambahkan data kendaraan baru ke dalam akun. |
| `DELETE`| `/users/vehicles/:id`| Pengguna | Menghapus kendaraan yang sudah tidak digunakan. |
| `GET` | `/admin/users` | **Admin** | Melihat seluruh pengguna terdaftar untuk keperluan monitoring. |
| `DELETE`| `/admin/users/:id` | **Admin** | Menghapus pengguna (Ban User). |

<br>

### 🅿️ Tabel Modul Manajemen Lahan Parkir
Menampilkan informasi gedung, slot, dan manajemen teknis slot (Admin).

| Method | Endpoint | Hak Akses | Deskripsi |
| :--- | :--- | :--- | :--- |
| `GET` | `/areas` | Public | Menampilkan daftar gedung dan sisa kuota parkir. |
| `GET` | `/areas/:id/slots`| Public | **[Snapshot]** Mengambil data status slot saat aplikasi pertama kali dibuka (untuk inisialisasi denah). Perubahan selanjutnya via WebSocket. |
| `POST` | `/areas/slots` | **Admin** | Menambahkan slot baru pada area tertentu. |
| `PUT` | `/areas/slots/:id`| **Admin** | Mengubah nama slot atau memperbaiki data lantai. |
| `PATCH`| `/areas/slots/:id/status`| **Admin** | Mengubah status slot menjadi `'maintenance'` (Rusak/Perbaikan). |

<br>

### 🎟️ Tabel Modul Reservasi & Transaksi
Menangani logika inti booking, tombol konfirmasi, dan integrasi IoT Trigger.

| Method | Endpoint | Hak Akses | Deskripsi & IoT Trigger |
| :--- | :--- | :--- | :--- |
| `POST` | `/reservations` | Public | Membuat tiket reservasi baru dan mengubah status slot jadi `'booked'`. |
| `GET` | `/reservations/:id` | Public | Melihat detail tiket aktif. |
| `PATCH`| `/reservations/:id/arrive` | Pengguna | **Tombol "Sudah Sampai"**. Mengubah status jadi `'occupied'` dan mengirim perintah **`buzzerOff`** ke IoT. |
| `PATCH`| `/reservations/:id/complete`| Pengguna | **Tombol "Keluar"**. Mengubah status jadi `'available'` dan mengirim perintah **`reset`** ke IoT. |
| `GET` | `/reservations/history` | Pengguna | Menampilkan daftar parkir yang sudah selesai. |

---
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

**Connection URL:** `base urlnya nanti menyusul`

### 📥 Listen (Server ➔ Client)

| Arah | Event Name | Payload Contoh | Deskripsi |
| :--- | :--- | :--- | :--- |
| **Server ➔ Client** | `slotUpdate` | `{ "areaId":"G1", "slotId":"A01", "status":"occupied" }` | Memperbarui warna denah saat ada booking atau sensor mendeteksi mobil. |
| **Server ➔ Client** | `adminStats` | `{ "totalIncome": 50000, "occupancy": "80%" }` | Memperbarui angka statistik di dashboard admin. |
| **Server ➔ Client** | `bookingTimer`| `{ "timeLeft": 280 }` | Menghitung mundur batas waktu check-in di layar pengguna. |

---

## 🤖 3. MQTT Specifications (For IoT Team)
Protokol komunikasi antara Hardware (ESP32/STM32) dan Server Cloud.

**Broker:** `base urlnya nanti menyusul` (GCE)

### 📡 Topics & Payloads

| Arah | Topik | Payload | Logika Backend / Aksi |
| :--- | :--- | :--- | :--- |
| **IoT ➔ Server** | `parkfinder/sensor/{area}/{slot}` | `0` atau `1` | Laporan sensor update status fisik (0=Kosong, 1=Ada). |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `buzzerOn` | Dikirim server jika sensor = `1` tapi status aplikasi masih `'available'` (Peringatan Parkir Liar). |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `buzzerOff`| Dikirim server setelah pengguna menekan tombol **"Sudah Sampai"** (Valid). |
| **Server ➔ IoT** | `parkfinder/control/{area}/{slot}`| `reset` | Server memerintahkan IoT untuk **Reset State** karena pengguna menekan tombol **"Keluar"**. |

> **⚠️ Note for IoT **
> Pastikan perangkat *subscribe* ke topik `parkfinder/control/...` segera setelah nyala untuk menerima perintah Buzzer/Reset dari server.

---