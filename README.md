# 9Chain Community Bot — LYID

Automation for **9Chain Community** (`9chain.com`) — the community hub of the 9Chain
Layer 1 project. Runs the daily loop automatically: boot the simulated node, spend all
1,000 taps, open the daily gift, and reinvest LOVE9 Points into the highest-ROI node
component upgrades.

Built and verified live on 2026-11-09 against `api.9chain.com` v2.

```
LYID - Layer Airdrop ID
9Chain Community Bot v1.0.0
```

---

## Proof it runs

Every screenshot below is the live output of the bot's own read-only commands on this machine — not a mockup. Account names and emails are masked; the numbers are untouched.

**`--status` reads every account straight from 9Chain's API: node tier, points/hour, unspent taps, team points, next-tier cost.**

![`--status` reads every account straight from 9Chain's API: node tier, points/hour, unspent taps, team points, next-tier cost.](docs/screenshots/status.png)

**`--plan` is a dry run: it ranks upgrades by ROI per point spent and buys nothing — the `[dry-run]` lines below prove that.**

![`--plan` is a dry run: it ranks upgrades by ROI per point spent and buys nothing — the `[dry-run]` lines below prove that.](docs/screenshots/plan.png)

**`--history` is written by the bot itself after every run: what type, which account, rate before and after, and what it actually bought.**

![`--history` is written by the bot itself after every run: what type, which account, rate before and after, and what it actually bought.](docs/screenshots/history.png)

---

## Apa yang dikerjakan bot ini

| Langkah | Endpoint | Keterangan |
|---|---|---|
| Login email + password + TOTP | `POST /v2/auth/login` | TOTP digenerate otomatis dari `totpSecret` (tidak perlu app authenticator) |
| Refresh token 14 hari | `POST /v2/auth/refresh` | Cookie `ola_rt` disimpan, jadi tidak perlu TOTP tiap run |
| Boot node harian | `POST /v2/program/enter` | Node berhenti accruing kalau tidak diboot |
| Tap 1.000x | `POST /v2/program/tap` | Server membatasi **maks 100 per request** → bot mengirim 10 batch |
| Daily gift | `POST /v2/me/check-in` | Streak harian, hadiah naik tiap hari |
| Upgrade komponen | `POST /v2/program/upgrade` | Strategi ROI: `powerGainRate / nextCost` tertinggi dibeli dulu |
| Naik tier node | `POST /v2/program/node/upgrade` | Hanya kalau semua komponen sudah mentok di cap tier |

## Fitur

- **Multi-akun** — satu `config.json`, semua akun diproses berurutan dengan jeda acak
  (6–18 detik antar akun) dan urutan akun diacak tiap run.
- **Registrasi otomatis** (`tools/register-browser.js`) — Chrome headful membuka
  `9chain.com/ref/<kode>`, mengisi form, Turnstile lolos sendiri, lalu verifikasi email
  diambil dari Gmail catch-all `@ralies.id`, country di-set, node diboot. Satu perintah
  menghasilkan akun siap farm.
- **Humanisasi per akun** (`src/persona.js`) — device id, user agent, locale, ritme tap,
  dan peluang skip upgrade diturunkan deterministik dari email akun, jadi tiap akun punya
  "sidik jari" sendiri yang stabil antar-run (nilai yang berubah-ubah justru sinyal).
- **TOTP otomatis** — RFC 6238 (SHA1/6 digit/30 detik) diimplementasi tanpa dependency,
  dengan toleransi clock skew ±30 detik. Akun tanpa 2FA tetap bisa login (field kosong).
- **Session cache** — token 15 menit disimpan di `data/session-*.json` dan di-refresh
  lewat cookie `ola_rt`, jadi login penuh (yang butuh TOTP) jarang terjadi.
- **Auto re-auth** — 401 → refresh → retry; refresh gagal → login ulang pakai TOTP.
- **Auto-unlock program** — akun baru punya `profile.country = null` dan SEMUA endpoint
  `/program/*` membalas 403. Bot mendeteksi ini dan mengeset country sendiri.
- **Upgrade berbasis ROI** — bukan "beli yang termurah": membandingkan
  `powerGainRate / nextCost` tiap komponen, memakai `costLadder` untuk level berikutnya,
  dan berhenti saat poin tidak cukup atau semua komponen cap.
- **Tier guard** — tier baru dibeli hanya kalau tidak ada komponen terjangkau lagi
  (ROI tier ~1.317 vs komponen ~2.320–2.611 poin/jam per 1 juta poin).
- **Dry-run** — `--plan` menampilkan rencana pembelian tanpa memotong poin.
- **Idempotent** — aman dijalankan berkali-kali dalam sehari: tap 0 tersisa, gift sudah
  diambil, dan tidak ada pembelian yang tidak perlu.
- **Retry cerdas** — GET retry 2x dengan backoff pada 429/5xx; POST (tap/upgrade/claim)
  **tidak** di-retry otomatis karena bukan idempotent.
- **Laporan lokal** — `data/last-run.json` menyimpan 60 run terakhir.

## Kebutuhan

| Item | Wajib? | Biaya | Catatan |
|---|---|---|---|
| Akun 9Chain (email terverifikasi) | Ya | Gratis | Daftar di https://www.9chain.com/register |
| Node.js 18+ | Ya | Gratis | Sudah terpasang di mesin ini (v22) |
| `totpSecret` akun | Ya | Gratis | Kode 2FA yang didapat saat mengaktifkan TOTP di settings |
| Proxy | Tidak | — | Project tidak memblokir IP Indonesia |
| Gmail catch-all `@ralies.id` | Opsional | Rp 0 | Hanya untuk registrasi akun baru otomatis (IMAP) |
| Modal / deposit | **Tidak** | **Rp 0** | Project ini murni poin, tanpa wallet, tanpa on-chain |

**Kategori modal: FREE (Category A).** Tidak ada transaksi on-chain, tidak ada gas,
tidak ada deposit. Yang dibutuhkan hanya email + TOTP.

## Instalasi

```bash
cd ~/LYID-BOTS/9chain-bot
npm install
cp config.json.example config.json
```

Isi `config.json`:

```json
{
  "accounts": [
    {
      "label": "akun-utama",
      "email": "emailkamu@gmail.com",
      "password": "passwordkamu",
      "totpSecret": "JBSWY3DPEHPK3PXP"
    }
  ]
}
```

`totpSecret` adalah kode base32 yang muncul saat kamu mengaktifkan 2FA di halaman
settings akun 9Chain (biasanya ditampilkan di sebelah QR code).

## Cara pakai

### Untuk yang tidak nyaman CLI

Klik dua kali:

- `JALANKAN.bat` — menu interaktif (semua pilihan)
- `CEK-POIN.bat` — cek status semua akun, langsung tampil lalu berhenti
- `LIHAT-UPGRADE.bat` — rencana upgrade (dry-run, tidak membeli poin)
- `LIHAT-JADWAL.bat` — riwayat 10 run terakhir + jadwal otomatis
- `DAFTAR-AKUN.bat` — daftar akun baru (buka Chrome, isi form, verifikasi otomatis)

### Command line

```bash
node index.js            # menu interaktif
node index.js --daily    # daily + upgrade otomatis (cocok untuk cron)
node index.js --status   # status semua akun (read-only)
node index.js --history  # riwayat run terakhir
node index.js --upgrade  # upgrade otomatis saja
node index.js --plan     # rencana upgrade, tidak membeli
node index.js --stats    # statistik publik project
```

### Jadwal otomatis

Sudah terpasang di cron Hermes: **job `9chain-daily`, setiap hari 08:00 WIB**
(`no_agent` — menjalankan `9chain_daily.py` → `node index.js --daily`, output disimpan lokal).

- Lihat hasil: `LIHAT-JADWAL.bat` atau `node index.js --history`
- File hasil mentah: `data/last-run.json` (60 run terakhir)
- Jam 08:00 WIB = 01:00 UTC, jadi daily gift dan tap sudah reset untuk hari baru
- Kalau mau ubah jam: `cronjob_manage action=update job_id=9905dadcab74 schedule="every day at 9am"`

### Menu interaktif

```
1. Cek status semua akun           (read-only)
2. Jalankan daily                  (boot node + 1000 tap + daily gift)
3. Upgrade otomatis                (beli komponen ROI tertinggi)
4. Daily + upgrade sekaligus
5. Rencana upgrade                 (dry-run, tidak membeli)
6. Statistik project               (read-only)
0. Keluar
```

## Nuyul / multi-akun (verified 2026-11-09)

Registrasi manual memblokir karena Turnstile, tapi jalur otomatisnya terbukti:

```bash
node tools/register-browser.js 1        # 1 akun baru, otomatis sampai siap farm
node tools/register-browser.js 5        # 5 akun (jeda 6-12 detik antar akun)
node tools/register-browser.js 3 --headless   # tanpa jendela (Turnstile sering gagal)
```

Yang terjadi di dalamnya:

1. **Chrome headful** membuka `https://www.9chain.com/ref/<kode-referral>` → Turnstile
   lolos sendiri di browser asli (tanpa solver, tanpa biaya).
2. Form diisi: display name acak dari pool, email `9chainxxxx@ralies.id` (catch-all →
   Gmail), password, dan kode referral kalau field-nya kosong.
3. Halaman berubah ke **"Check your inbox"** → akun sudah dibuat, belum aktif.
4. `tools/verify_account.py` membaca email aktivasi dari Gmail (IMAP), memanggil
   `POST /v2/auth/verify-email {token}`, lalu `PATCH /v2/me/profile {country:"ID"}`
   dan `POST /v2/program/enter`.
5. Akun ditambahkan ke `config.json` dengan device id unik.

Hasil uji: 2 akun baru masuk, **F1 akun utama naik 6 → 8** (atribusi referral dari URL
`/ref/` bekerja otomatis), tier 1, rate 150/jam, 1.000 tap tersedia.

### Humanisasi (dan batas jujurnya)

`src/persona.js` menurunkan per akun dari hash email:

| Sinyal | Perilaku bot |
|---|---|
| `x-device-id` | UUID unik per akun (dari browser saat registrasi), stabil antar-run |
| User agent | 6 varian Chrome/Edge/Firefox + macOS, tetap per akun |
| Locale | en-US / id-ID / en-GB / ms-MY |
| Target tap | 92–100% dari 1.000 (tidak selalu penuh) |
| Ukuran batch | 60–100 acak, bukan selalu 100 |
| Jeda antar batch | 0,9–3,1 detik, 25% peluang jeda panjang 3,5–6,5 detik |
| Upgrade | 10% peluang skip per hari, jeda 1,5–7 detik antar pembelian |
| Urutan akun | diacak tiap run, jeda 6–18 detik antar akun |

**Yang TIDAK diperbaiki humanisasi:** semua akun keluar dari **satu IP publik** dan
dijalankan pada jam yang sama. Itu sinyal terkuat untuk sistem anti-sybil. Kalau mau
lebih aman, jalankan akun di jaringan berbeda (VPS/proxy residensial) dan sebar jadwal
cron — tapi jangan harap ini "tidak terdeteksi". Project ini punya modul `anti_sybil`
di dalam game dan `fraudRiskScore` di `/me/reputation`, jadi mereka memang mengukur.

## Strategi upgrade (kenapa beli komponen dulu, bukan tier)

Diukur langsung dari `catalog` pada 2026-11-09:

| Aksi | Biaya (poin) | Gain | Payback | ROI (poin/jam per 1M) |
|---|---|---|---|---|
| Komponen (L11–L14) | 2,8–6,5 juta | +6.667…+15.430/jam | ~16–18 hari | **2.320–2.611** |
| Tier 23 → 24 | 29,37 juta | +38.681/jam | ~31,6 hari | **1.317** |

Jadi bot selalu menghabiskan poin ke komponen lebih dulu. Tier hanya dibeli kalau semua
komponen sudah mentok di `maxLevel` (cap mengikuti tier) atau `upgrade.autoTier: true`.

Hasil nyata setelah bot dijalankan pada akun uji:

```
Rate awal   : 172.706 /jam
Setelah daily + 4 upgrade: 233.904 /jam  (+35%, +61.198/jam)
Poin terpakai: 20,07 juta (dari 24,15 juta hasil tap + gift)
```

## Referensi konfigurasi

| Key | Default | Arti |
|---|---|---|
| `baseUrl` | `https://api.9chain.com/v2` | Base API |
| `requestTimeoutMs` | 45000 | Timeout request (upgrade bisa lambat) |
| `daily.autoEnter` | true | Boot node tiap run |
| `daily.tapEnabled` | true | Kirim tap |
| `daily.tapsPerRun` | 1000 | Batas tap per run (server: 1000/hari) |
| `daily.tapBatchSize` | 100 | Maks 100 — **jangan dinaikkan**, server menolak >100 |
| `daily.autoCheckin` | true | Buka daily gift |
| `upgrade.enabled` | true | Auto upgrade |
| `upgrade.reservePoints` | 0 | Poin yang disisakan (tidak dibelanjakan) |
| `upgrade.maxUpgradesPerRun` | 12 | Batas pembelian per run |
| `upgrade.autoTier` | false | Beli tier walau komponen masih bisa dibeli |
| `timing.delayMinMs/MaxMs` | 900/2600 | Jeda antar batch tap |
| `timing.accountDelayMinMs/MaxMs` | 4000/9000 | Jeda antar akun |

## Struktur file

```
9chain-bot/
├── index.js               # entry point, menu, flag CLI
├── src/
│   ├── api.js             # client API + auth (login TOTP, refresh cookie)
│   ├── actions.js         # daily: country -> enter -> tap -> check-in, status
│   ├── upgrade.js         # strategi ROI + eksekusi upgrade/tier
│   ├── persona.js         # humanisasi per akun (device id, UA, ritme)
│   └── totp.js            # TOTP RFC 6238 tanpa dependency
├── tools/
│   ├── register-browser.js # daftar akun baru via Chrome headful + verifikasi
│   ├── verify_account.py   # baca email aktivasi (IMAP) -> verify -> country -> enter
│   └── debug-register-page.js # dump input/tombol halaman register (diagnosa selector)
├── utils/
│   ├── logger.js          # logger berwarna + helper sleep/jitter
│   └── banner.js          # banner LYID
├── config.json            # kredensial (gitignored)
├── config.json.example    # template aman untuk repo
├── data/                  # session cache + last-run.json (gitignored)
├── JALANKAN.bat           # menu klik-dan-jalan
├── CEK-POIN.bat           # cek status cepat
├── README.md
└── CHECKPOINT.md
```

## Project info

| Item | Nilai |
|---|---|
| Nama | 9Chain (9Chain Community) |
| Situs | https://www.9chain.com · https://9chain.org |
| Explorer | https://9scan.org · https://c1.9scan.org · https://a1.9scan.org |
| Kategori | Layer 1 (Cosmos SDK C1 + Avalanche A1 testnet) + program poin komunitas |
| Status | Pilot / testnet publik (chain C1 mulai 2026-08-27) |
| Poin | LOVE9 Points — **bukan aset, tidak bisa ditransfer/dikonversi di app** |
| Distribusi token | Belum ada; rasio & kriteria hanya setelah proposal + voting komunitas |
| Biaya farm | Rp 0 |
| Syarat wilayah | Tidak tersedia untuk EEA, UK, Swiss, USA |
| Skor LYID | 66/100 — layak dibuild, imbalan spekulatif |

> Catatan jujur: 9Chain secara eksplisit menyebut LOVE9 **bukan uang** dan belum ada
> token. Nilai ekonomi bot ini sepenuhnya bergantung pada keputusan distribusi di masa
> depan. Yang pasti: poin, tier, dan posisi leaderboard naik, dan biayanya nol.

## Troubleshooting

| Pesan | Arti | Solusi |
|---|---|---|
| `TOTP_INVALID` / login 401 | Jam mesin bergeser atau secret salah | Bot sudah mencoba ±30 detik; sinkronkan jam Windows, cek `totpSecret` |
| `MALFORMED_BODY` | Body POST kosong dikirim sebagai `null` | Sudah diperbaiki: POST tanpa body mengirim `{}` |
| `VALIDATION_FAILED` pada `count` | Batch tap >100 | Turunkan `tapBatchSize` (maks 100) |
| `PROGRAM_QUESTS_DISABLED` | Quest sedang dimatikan project | Normal — bot tidak memakai quest |
| `PROGRAM_NOT_ENTERED` | Node belum diboot | Jalankan menu 2 (`--daily`) |
| `403` di semua `/program/*` | Akun baru, `profile.country` masih null | Bot sekarang set country otomatis; manual: `PATCH /v2/me/profile {"country":"ID"}` |
| `CAPTCHA_REQUIRED` saat daftar | Turnstile aktif di endpoint register | Daftar lewat `tools/register-browser.js` (Chrome headful), bukan HTTP langsung |
| `INVALID_REFRESH_TOKEN` | Cookie refresh kadaluarsa (14 hari) | Login ulang otomatis pakai TOTP — pastikan `totpSecret` benar |
| Upgrade lambat / timeout | Server butuh ~40 detik saat ramai | Naikkan `requestTimeoutMs`, jangan spam ulang |
| `enter gagal` tapi daily tetap jalan | Node sudah aktif | Aman, tidak mempengaruhi accrual |

## Keamanan

- `config.json` berisi password + TOTP secret → sudah masuk `.gitignore`, **jangan** commit.
- `data/session-*.json` berisi access + refresh token → gitignored juga.
- Bot tidak menyimpan private key apa pun karena project ini tidak memakai wallet.
- `config.json.example` hanya berisi placeholder (`xxxuser@gmail.com`).

## Lisensi & dukungan

Dibuat untuk komunitas **Layer Airdrop ID (LYID)**.
Telegram: https://t.me/layerairdrop · Diskusi: https://t.me/layerairdropdiskusi
GitHub: https://github.com/layerairdrop · Kontak: @rmndkyl

Script ini dijual sebagai paket premium LYID. Dilarang menjual ulang tanpa izin.
