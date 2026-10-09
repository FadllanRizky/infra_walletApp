# WalletKu — E-Wallet (Spring Boot + React TypeScript)

Aplikasi dompet digital: registrasi, top up, transfer antar pengguna, dan pembayaran tagihan.
Backend Java Spring Boot 4 + PostgreSQL, frontend React 19 + TypeScript + Vite.

---

## Menjalankan

### 1. Database

Butuh PostgreSQL berjalan di `localhost:5432` dengan database `wallet_db`:

```sql
CREATE DATABASE wallet_db;
```

### 2. Backend

Kredensial **tidak** disimpan di kode — set lewat environment variable dulu:

```bash
cd backend/app
cp .env.example .env        # lalu isi nilainya

export DB_PASSWORD='password-postgres-kamu'
export JWT_SECRET="$(openssl rand -base64 48)"   # wajib >= 32 karakter

./mvnw spring-boot:run
```

Backend jalan di `http://localhost:8080`.
`JWT_SECRET` di bawah 32 karakter membuat aplikasi **gagal start** dengan pesan yang jelas — itu disengaja, bukan bug.

### 3. Frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend jalan di `http://localhost:5173`. Untuk deploy, set `VITE_API_URL` (lihat `.env.example`).

### Akun demo

Dibuat otomatis oleh `DataSeeder` saat database masih kosong — **aktifkan dulu** dengan `SEED_ENABLED=true`, karena default-nya mati demi keamanan:

| Username | Password | PIN |
|---|---|---|
| `budi` | `password123` | `246813` |
| `siti` | `password123` | `246813` |

Seeder memakai nilai dari kolom dengan nama yang sama di database, jadi tetap mati saat database sudah terisi.

---

## Keamanan

Yang sudah diterapkan di sisi Java:

| Lapisan | Implementasi |
|---|---|
| Password & PIN | BCrypt strength 12, tidak pernah disimpan/di-log dalam bentuk asli |
| Access token | JWT 15 menit, klaim `jti`/`iss`/`aud` diverifikasi |
| Refresh token | Token terpisah bertipe `typ=refresh`, ditolak kalau dipakai sebagai access token |
| Pencabutan token | Kolom `tokensValidFrom` — logout & ganti password langsung mematikan token lama |
| Brute force PIN | Kunci 15 menit setelah 5 percobaan salah (`PinService`) — counter tercatat di transaksi terpisah sehingga tetap bertahan saat transfer di-rollback |
| Brute force login | Kunci akun + rate limit per IP di endpoint `/api/auth/**` |
| Enumerasi user | Pesan error login identik untuk user salah & password salah, plus dummy hash agar timing seragam |
| Batas transaksi | Minimum, maksimum per transaksi, dan akumulasi harian (`TransactionLimitService`) |
| Audit trail | Setiap login, transfer, top up, bayar tagihan, dan kegagalan PIN tercatat dengan IP + user agent |
| Race condition | Id wallet di-resolve dulu tanpa memuat entitas, lalu pessimistic lock terurut berdasarkan id agar transfer silang tidak deadlock ataupun bentrok versi optimistik |
| Header HTTP | HSTS, CSP, X-Frame-Options DENY, X-Content-Type-Options, Referrer-Policy |
| Kebocoran data | Nama penerima dimask (`bud*`), stack trace tidak pernah dikirim ke client |
| Otorisasi objek | Query detail transaksi & tagihan di-scope per user — id orang lain menghasilkan 404 |

Konfigurasi ada di `application.properties` (prefix `app.security.*` dan `app.limits.*`).

---

## Struktur

```
backend/app/src/main/java/com/wallet/app/
├── config/       SecurityConfig (header, CORS, BCrypt), DataSeeder
├── controller/   Auth, Wallet, BillPayment, User
├── dto/          Request/response, validasi Bean Validation
├── entity/       User, Wallet, Transaction, Biller, BillPayment, AuditLog
├── exception/    ApiException + GlobalExceptionHandler
├── repository/   Spring Data JPA
├── security/     JwtService, JwtAuthFilter, RateLimitFilter
└── service/      Wallet, User, BillPayment, Pin, TransactionLimit, Audit

frontend/src/
├── api/          Axios + auto-refresh token saat 401
├── components/   BottomNav, PageHeader, PinInput, AmountInput, BalanceCard, ConfirmDialog, TransactionRow, EmptyState, Skeleton
├── hooks/        useWallet (saldo + sisa limit)
├── lib/          Format rupiah, tanggal relatif, nomor rekening
├── pages/        15 halaman
└── store/        Zustand (auth, persisted)
```

---

## API

| Method | Endpoint | Auth | Keterangan |
|---|---|---|---|
| POST | `/api/auth/register` | – | Daftar + buat wallet |
| POST | `/api/auth/login` | – | Login |
| POST | `/api/auth/refresh` | – | Tukar refresh token |
| POST | `/api/auth/logout` | ✓ | Cabut semua token |
| GET | `/api/wallet/me` | ✓ | Saldo + sisa limit harian |
| GET | `/api/wallet/inquiry?accountNumber=` | ✓ | Cek nama pemilik rekening |
| POST | `/api/wallet/topup` | ✓ | Top up (butuh PIN) |
| POST | `/api/wallet/transfer` | ✓ | Transfer (butuh PIN) |
| GET | `/api/wallet/transactions` | ✓ | Riwayat, filter `type`, paginasi |
| GET | `/api/wallet/transactions/{id}` | ✓ | Detail transaksi |
| GET | `/api/bill/billers` | ✓ | Semua biller aktif |
| GET | `/api/bill/billers/{category}` | ✓ | Biller per kategori |
| POST | `/api/bill/check` | ✓ | Cek tagihan |
| POST | `/api/bill/pay` | ✓ | Bayar (butuh PIN) |
| GET | `/api/bill/history` | ✓ | Riwayat tagihan |
| GET | `/api/bill/history/{id}` | ✓ | Detail pembayaran (di-scope per user) |
| GET | `/api/user/profile` | ✓ | Profil |
| GET | `/api/user/activity` | ✓ | Riwayat keamanan (login, ganti PIN, dst.) |
| POST | `/api/user/change-pin` | ✓ | Ganti PIN |
| POST | `/api/user/change-password` | ✓ | Ganti password (cabut sesi lain) |

---

## Catatan teknis

- **Skema database** dibuat lewat `ddl-auto=update`. Untuk produksi sebaiknya pindah ke Flyway agar perubahan kolom bisa direview dan di-rollback.
- **Rate limit** disimpan in-memory, jadi hanya efektif untuk satu instance. Di belakang load balancer perlu Redis atau gateway.
- **Simulasi tagihan** (`simulateBillAmount`) menghasilkan nominal deterministik dari nomor pelanggan — ini stub, bukan integrasi biller sungguhan.
