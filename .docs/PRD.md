# PRD & Technical Documentation — Laravel Boilerplate

> **Status:** Draft — belum dieksekusi.
> **Last updated:** 2026-09-27
> **Aturan:** Dokumen ini WAJIB diupdate setiap kali ada penambahan/perubahan keputusan teknis, struktur, atau scope.

---

## 1. Tujuan

Membangun boilerplate Laravel yang bisa dipakai berulang sebagai starting point untuk proyek fullstack (Blade + Inertia), dengan struktur controller, auth, dan database yang sudah tertata sejak awal.

## 2. Tipe Proyek

- **Fullstack hybrid**: Blade untuk landing/marketing page, Inertia (React) untuk area dashboard/app.
- Multi-tenant: **belum diputuskan**. Database didesain "tenant-ready" (kolom `organization_id` nullable + indexed di tabel domain) tanpa menerapkan tenant-filtering logic dulu.

## 2.1 MVP Scope (v1)

Fokus MVP dipersempit menjadi 3 alur inti saja:

1. **Login** — Breeze (Inertia + React), auth session standar
2. **Dashboard** — halaman kosong/shell setelah login, layout Inertia + shadcn/ui
3. **Log Viewer** — halaman untuk melihat log aplikasi (opcodesio/log-viewer), bisa diakses user yang sudah login (auth middleware standar; pembatasan by role ditunda ke post-MVP)

**Post-MVP (ditunda, tapi struktur sudah disiapkan di dokumen ini):**
- Tabel & logic `organizations` (multi-tenant skeleton)
- Spatie Permission — role/permission granular (MVP cukup pakai auth middleware biasa, belum perlu role check)
- Spatie Activitylog (audit trail)
- `UserObserver` & `OrganizationObserver`
- Controller API (`Controllers/Api/V1`) — MVP hanya butuh Web controller
- Pemisahan Requests/Resources Api vs Web — cukup Web dulu

## 3. Tech Stack

| Layer | Pilihan |
|---|---|
| Framework | Laravel 12, PHP 8.3+ |
| Auth pages | Laravel Breeze (Inertia + React preset) |
| API auth | Laravel Sanctum |
| Role & Permission | Spatie Laravel-Permission |
| Audit trail | Spatie Activitylog |
| Frontend dashboard | Inertia + React + shadcn/ui + Tailwind |
| Frontend landing | Blade + Tailwind (custom minimal) |
| Testing | Pest |
| Static analysis | Larastan (PHPStan) |
| Code style | Laravel Pint |
| Queue | Redis + Laravel Horizon |
| Cache | Redis |
| Log Viewer | opcodesio/log-viewer |

## 4. Struktur Folder

```
app/
  Actions/
  Data/                      # DTO (spatie/laravel-data)
  Enums/
  Http/
    Controllers/
      Controller.php         # base abstract controller
      Api/
        V1/                  # semua controller API, versioned
      Web/
        Auth/                # dipindah dari Breeze default
        DashboardController.php
        ProfileController.php
    Middleware/
    Requests/
      Api/V1/
      Web/
    Resources/
      Api/V1/
  Models/
  Observers/
  Policies/
  Providers/
  Services/

resources/
  views/                     # Blade: landing, email
  js/
    Pages/
      Auth/                  # dari Breeze
      Dashboard/             # Inertia + shadcn/ui
    Components/
      ui/                    # komponen shadcn/ui
    Layouts/

routes/
  web.php                    # -> Controllers\Web\*  (Blade + Inertia)
  api.php                    # -> Controllers\Api\V1\*

database/
  migrations/
  factories/
  seeders/

tests/
  Feature/
  Unit/
```

### Prinsip Pemisahan Controller
- `Controllers/Api/*`: selalu return JSON (Resource/response()->json()), guard Sanctum, versioned.
- `Controllers/Web/*`: selalu return `Inertia::render()` atau Blade `view()`, guard session.
- Form Request & API Resource ikut dipisah per konteks (`Requests/Api` vs `Requests/Web`).

## 5. Desain Database Awal

| Tabel | Kolom Kunci | Catatan |
|---|---|---|
| `users` | id, name, email(unique), password, avatar, is_active, timestamps, soft deletes | base Breeze + tambahan |
| `organizations` | id, name, slug, owner_id (FK users), settings(json), timestamps | skeleton, tenant-ready, belum aktif dipakai |
| `roles`, `permissions`, `model_has_roles`, `model_has_permissions`, `role_has_permissions` | — | auto dari Spatie Permission |
| `personal_access_tokens` | — | dari Sanctum |
| `activity_log` | — | dari Spatie Activitylog |
| `settings` | key(unique), value(json), group | app-level config |
| Tabel domain lain (future) | + `organization_id` nullable, indexed | disiapkan untuk kemungkinan multi-tenant |

## 6. Model Observers

| Observer | Model | Event & Aksi |
|---|---|---|
| `UserObserver` | `User` | `created`: assign role default (mis. `member`); `deleted`: cleanup relasi terkait |
| `OrganizationObserver` | `Organization` | `creating`/`updating`: auto-generate `slug` dari `name` |

Observer didaftarkan di `AppServiceProvider::boot()` via `Model::observe()`.

## 7. Log Viewer

- Package: **opcodesio/log-viewer**
- Route: `/dashboard/logs` (Web, Inertia layout tidak dipakai — package ini render UI sendiri via route bawaan)
- **MVP**: akses cukup `auth` middleware (user login apapun bisa lihat)
- **Post-MVP**: dibatasi lebih lanjut via role (mis. `admin`) memakai Spatie Permission (`LogViewer::auth()` callback)
- Tidak perlu tabel database tambahan — baca langsung dari file log di `storage/logs`

## 8. Keputusan yang Sudah Difinalkan

- [x] Fullstack hybrid: Blade (landing) + Inertia React (dashboard) — *dashboard jadi fokus MVP, landing Blade ditunda*
- [x] Auth: Breeze (session) untuk MVP; Sanctum + Spatie Permission ditunda ke post-MVP
- [x] Starter kit auth: Laravel Breeze (Inertia + React)
- [x] Dashboard UI: shadcn/ui
- [x] Controller dipisah: `Api/V1` vs `Web` — *struktur disiapkan, tapi MVP hanya isi `Web`*
- [ ] Multi-tenant: **ditunda ke post-MVP**
- [x] Observer: `UserObserver` & `OrganizationObserver` — **ditunda ke post-MVP**
- [x] Log viewer: opcodesio/log-viewer — MVP akses basic auth, role-restriction post-MVP

## 9. Urutan Eksekusi — MVP (v1)

1. [x] `laravel new` project (Laravel 13.33) + install Breeze (Inertia + React preset) → user bisa register/login
2. [x] Setup struktur folder `Controllers/Web` (pindahkan auth controller Breeze ke `Web/Auth`, `ProfileController` ke `Web/`)
3. [x] Buat `DashboardController` (Web) + dashboard shell bawaan Breeze sebagai halaman setelah login (shadcn/ui ditunda — lihat catatan di bawah)
4. [x] Install & konfigurasi opcodesio/log-viewer, route `/dashboard/logs`, protect dengan middleware `auth`, link ditambahkan di navigasi dashboard
5. [x] Jalankan Laravel Pint

**Definition of Done MVP:** user register/login → landing di dashboard shell → bisa klik menu "Logs" / akses `/dashboard/logs` dan melihat isi log aplikasi. **Terverifikasi end-to-end** (register → login → GET /dashboard 200 → GET /dashboard/logs 200).

### Catatan eksekusi & penyesuaian dari rencana awal

- **PHP**: mesin ini semula default ke PHP 7.1. **Sudah diperbaiki** — `~/.zshrc` diubah supaya PHP 8.4 (Homebrew) jadi default di PATH. PHP 7.1 tetap terinstall untuk project lama, akses via path eksplisit `/opt/homebrew/opt/php@7.1/bin/php` kalau perlu.
- **Laravel version**: terinstall 13.33 (versi terbaru saat eksekusi), bukan 12 seperti draft awal — bukan masalah, hanya update catatan versi.
- **npm/vite conflict**: `laravel-vite-plugin@^3.1` resolve ke 3.2.0 yang minta `vite@^8`, sedangkan `@vitejs/plugin-react@4.7` masih declare peer `vite@<=7`. Di-resolve permanen dengan `legacy-peer-deps=true` di `.npmrc` (bukan downgrade versi), build & dev server jalan normal.
- **Breeze bug**: stub `resources/js/app.jsx` React mengimpor `./bootstrap` tapi file tersebut tidak di-generate oleh preset Breeze React (kehapus dari skeleton dasar Laravel). Dibuat ulang manual (`resources/js/bootstrap.js`, setup axios standar) — tanpa ini build gagal total.
- **Database**: pakai SQLite (`database/database.sqlite`) untuk kecepatan setup lokal, bukan MySQL/PostgreSQL — cocok untuk MVP, migrasi (`users`, `cache`, `jobs`) sudah jalan.
- **Sanctum, Spatie Permission, Activitylog, Organizations, Observers, Controllers/Api** belum dikerjakan — tetap di backlog post-MVP (lihat section 10).

### shadcn/ui — sudah terpasang (2026-09-28)

- Diinstall via `npx shadcn@latest init --template laravel --base radix --preset nova`
- **Migrasi Tailwind v3 → v4 (CSS-first)**: project awalnya scaffold Breeze pakai Tailwind v3 (`tailwind.config.js` + `postcss.config.js` + `@tailwind` directives), padahal shadcn CLI generate CSS gaya v4. Untuk konsistensi dengan arah resmi Tailwind & starter kit Laravel terbaru, project dimigrasi penuh ke v4:
  - `tailwind.config.js` & `postcss.config.js` **dihapus**
  - `@tailwindcss/vite` didaftarkan sebagai plugin di `vite.config.js`
  - `resources/css/app.css` ditulis ulang CSS-first: `@import "tailwindcss"`, `@theme inline { ... }` untuk mapping token warna shadcn (`--color-*`) & font, `@plugin "@tailwindcss/forms"`, `@source` untuk path Blade yang sebelumnya ada di `content` config v3
- **Bug ditemukan & diperbaiki**: preset "laravel" dari shadcn CLI belum menyertakan blok `@theme inline` yang memetakan variabel semantik (`--background`, `--border`, dst) ke namespace `--color-*` yang dibutuhkan Tailwind v4 — tanpa ini utility seperti `border-border`/`bg-background` tidak dikenali dan build gagal. Ditambahkan manual.
- **Case-sensitivity bug dicegah**: filesystem macOS ini case-insensitive, sehingga folder default shadcn (`resources/js/components/`, huruf kecil) akan "menyatu" dengan folder Breeze yang sudah ada (`Components/`, huruf besar) — aman di macOS tapi **akan pecah di server Linux** (case-sensitive) saat deploy. Diperbaiki dengan set alias di `components.json` ke `@/Components` (mengikuti konvensi Breeze yang sudah ada), bukan `@/components`.
- Komponen terpasang: `button`, `card`, `table`, `dialog`, `sidebar`, `input`, `separator`, `sheet`, `skeleton`, `tooltip` (semua `.jsx`, bukan `.tsx`, sesuai `"tsx": false` di `components.json`)
- Dashboard (`resources/js/Pages/Dashboard.jsx`) sudah dipakaikan komponen `Card` shadcn/ui sebagai bukti integrasi, bukan cuma terinstall tanpa dipakai
- Diverifikasi: `npm run build` sukses, 25 test PHPUnit tetap lulus, Pint tetap bersih, alur login → dashboard (200) → `/dashboard/logs` (200) tetap berfungsi

## 10. Urutan Eksekusi — Post-MVP (backlog, belum dijadwalkan)

- [x] ~~Install shadcn/ui di `resources/js`, restyle dashboard shell~~ — selesai 2026-09-28
- [x] ~~Set PHP 8.4 sebagai default di environment lokal~~ — selesai 2026-09-27
- Install & konfigurasi Sanctum + Spatie Permission, migrasi ke role-based access (termasuk restrict log viewer by role, gantikan basic `auth` middleware)
- Install Spatie Activitylog
- Buat tabel & seeder `organizations`, `settings`
- Buat `UserObserver` & `OrganizationObserver`, daftarkan di `AppServiceProvider`
- Tambahkan `Controllers/Api/V1` + Requests/Resources Api, buat landing page Blade
- Larastan + Pest setup lengkap

Status: **MVP v1 selesai dieksekusi dan terverifikasi** (2026-09-27). shadcn/ui + migrasi Tailwind v4 selesai (2026-09-28). Backlog di atas menunggu prioritas berikutnya.

## 11. Changelog

| Tanggal | Perubahan |
|---|---|
| 2026-09-27 | Dokumen dibuat. Mencatat seluruh keputusan awal: stack, struktur folder, pemisahan controller Api/Web, desain database, pilihan shadcn/ui untuk dashboard. |
| 2026-09-27 | Tambah section Model Observers: `UserObserver` (assign role default, cleanup saat deleted) & `OrganizationObserver` (auto-generate slug). |
| 2026-09-27 | Tambah halaman Log Viewer menggunakan package opcodesio/log-viewer, akses dibatasi role admin. |
| 2026-09-27 | Persempit scope jadi MVP v1: login → dashboard shell → log viewer. Sanctum, Spatie Permission, Activitylog, Observers, Organizations, Controllers/Api, dan landing page Blade ditunda ke backlog post-MVP. |
| 2026-09-27 | MVP v1 dieksekusi & diverifikasi end-to-end. Laravel 13.33 (bukan 12), PHP 8.4 via Homebrew (default sistem masih 7.1), npm pakai `--legacy-peer-deps` karena conflict vite/plugin-react, `resources/js/bootstrap.js` dibuat ulang (bug hilang dari stub Breeze React), shadcn/ui ditunda ke post-MVP, DB pakai SQLite. |
| 2026-09-27 | Audit best practice Laravel 13: struktur streamlined (tanpa Kernel.php/RouteServiceProvider), model pakai PHP attributes `#[Fillable]`/`#[Hidden]`, semua sesuai — tidak ada perbaikan kode diperlukan. PHP 8.4 dijadikan default di `~/.zshrc` (sebelumnya 7.1). |
| 2026-09-28 | shadcn/ui terpasang (template Laravel, base Radix, preset Nova). Migrasi penuh Tailwind v3 → v4 (CSS-first via `@tailwindcss/vite`, hapus `tailwind.config.js`/`postcss.config.js`). Perbaiki 2 bug: (1) preset CLI tidak menyertakan `@theme inline` mapping token warna ke `--color-*`, ditambahkan manual; (2) alias default CLI (`@/components`, huruf kecil) akan bentrok dengan folder Breeze (`Components`, huruf besar) di filesystem case-sensitive (Linux) — diarahkan ke `@/Components`. Dashboard dipakaikan komponen `Card` sungguhan. Semua test & Pint tetap lulus. |
