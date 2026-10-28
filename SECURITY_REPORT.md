# Bloom v3.10.1 — Security & Data Audit Report

Audit independen terhadap installer `bloom_3.10.1_x64-setup.exe` dan source code v3.10.1.

**Tujuan:** menjawab pertanyaan — apakah Bloom mencuri data, mengandung malware/backdoor, atau mengakses hal sensitif tanpa izin?

**Kesimpulan singkat:** **Bersih.** Bloom bukan malware. Ia mengakses data sistem (keyboard, audio, window) secara fungsional untuk fitur taskbar/notch, dan tidak ada telemetry/pencurian data. Satu catatan privasi: geolocation IP lewat HTTP.

---

## 1. Verifikasi Integritas Installer

| Item | Nilai |
|---|---|
| SHA256 installer | `5cb4d8fe87aaf26f161be61954aebfacb4c9f481b12c5bbcfc48aa6b56db7231` |
| SHA256 upstream resmi | `5cb4d8fe87aaf26f161be61954aebfacb4c9f481b12c5bbcfc48aa6b56db7231` |
| Status | ✅ **identik byte-for-byte** dengan rilis resmi |

Installer = salinan resmi, tidak dimodifikasi. Tidak ada backdoor yang disisipkan.

## 2. Isi Installer (NSIS)

```
bloom_3.10.1_x64-setup.exe (14.6 MB)
├── bloom.exe                          (33.1 MB) — app utama (Rust + WebView2)
├── MicrosoftEdgeWebview2Setup.exe     (1.8 MB)  — bootstrapper WebView2 (standar Tauri)
├── uninstall.exe                      — uninstaller
└── $PLUGINSDIR/                       — plugin NSIS standar (nsDialogs, StartMenu, System)
```

Installer hanya mengekstrak `bloom.exe` + WebView2, membuat shortcut, dan menjalankan app. Tidak ada tindakan mencurigakan (tidak menulis registry autorun tersembunyi, tidak menginstal service, tidak men-download payload tambahan).

**Digital signature:** tidak ada (wajar untuk open source; README asli juga mencatat ini).

## 3. Data Sensitif yang Diakses

| Data | Cara Akses | Tujuan | Disimpan? | Dikirim Keluar? |
|---|---|---|---|---|
| **Keyboard** (Win+1..9) | `SetWindowsHookExW(WH_KEYBOARD_LL)` | Ganti window di dock (pengganti taskbar Explorer) | ❌ state in-memory | ❌ |
| **Audio sistem (loopback)** | WASAPI `AUDCLNT_STREAMFLAGS_LOOPBACK` | Visualizer musik (FFT 5-band) | ❌ in-memory, dibuang | ❌ |
| **Mikrofon** | — | **TIDAK PERNAH diakses** | ❌ | ❌ |
| **Judul & thumbnail window lain** | `GetWindowTextW`, `PrintWindow` | Preview window di dock | cache in-memory (bounded) | ❌ |
| **Foreground window** | `GetForegroundWindow` | Fokus & overlap detection | ❌ | ❌ |
| **Lokasi (kota)** | IP geolocation (`ipapi.co`, `ip-api.com`) | Widget cuaca | ✅ lat/lon/city di settings lokal | ✅ ke 2 provider |
| **Bluetooth devices** | WinRT Bluetooth API | Status panel | ❌ in-memory | ❌ |
| **WiFi (daftar + connect)** | `wlanapi.dll` | Panel WiFi | ❌ | ❌ |
| **Battery, brightness, volume** | WMI / Win32 | Status panel | ❌ | ❌ |
| **Proses & window** | Toolhelp32 / Win32 | Dock + overlap | ❌ | ❌ |

## 4. Endpoint Outbound (8 domain, semua fungsional)

| Endpoint | Tujuan | Skema |
|---|---|---|
| `api.open-meteo.com` | Data cuaca | HTTPS |
| `geocoding-api.open-meteo.com` | Pencarian kota | HTTPS |
| `ipapi.co` | Geolocation IP (utama) | HTTPS |
| `ip-api.com` | Geolocation IP (fallback) | **HTTP ⚠️** |
| `fonts.googleapis.com` / `fonts.gstatic.com` | Font UI | HTTPS |
| `github.com` | Auto-update (signed minisign) | HTTPS |
| `bloom.sehaz.space` | Pengumuman (announcements.json) | HTTPS |

Tidak ada endpoint ke server pihak ketiga yang tidak dijelaskan, tidak ada telemetry, tidak ada analytics.

## 5. Pemeriksaan Malware / Backdoor

| Cek | Hasil |
|---|---|
| Strings: keylogger / `GetAsyncKeyState` untuk logging | ✅ Hanya untuk Win+1..9, bukan logger |
| Strings: `wallet.dat`, browser password, credential steal | ✅ tidak ada |
| Strings: miner (`xmrig`, `monero`, bitcoin address) | ✅ tidak ada |
| Strings: telemetry (`sentry`, `telemetry`, `upload_data`) | ✅ tidak ada |
| Strings: exfiltrasi (`exfiltrate`, `steal`) | ✅ tidak ada |
| DLL import mencurigakan | ✅ hanya Win32 standar (user32, gdi32, wlanapi, ws2_32, dll) |
| Win32 API: `ShellExecute`, `CreateProcess` | ✅ hanya untuk buka file shelf (user intent) |
| Auto-update | ✅ signature minisign diverifikasi sebelum apply |

## 6. Keamanan Arsitektur (source code)

- **CSP ketat**: `script-src 'self'` — tidak bisa injeksi script dari luar
- **`connect-src` allowlist**: hanya 8 domain di atas
- **Tidak ada `eval()` / `innerHTML` dari input user**: 2 titik `innerHTML` hanya render SVG statik
- **Anti path traversal**: `sanitize_filename()` untuk custom icons
- **Anti XML injection**: `escape_xml()` untuk WLAN profile WiFi
- **Tidak ada secret/API key hardcoded** di source
- **13 vulnerability** di `package.json` semuanya **dev-only** (vite, esbuild, dll) — tidak masuk ke app yang dikirim

## 7. ⚠️ Temuan & Rekomendasi

### 7.1 [Medium] Geolocation IP via HTTP (plaintext)

- **Lokasi:** `src/hooks/useWeather.ts:162`
- **Masalah:** fallback `ip-api.com` dipanggil lewat `http://`. Di jaringan publik (WiFi cafe, hotel), pihak ketiga bisa melihat request geolocation (kota + lat/lon kasar).
- **Risiko:** privasi lokasi level kota terbocor, bukan lokasi presisi.
- **Fix:** ganti ke `https://ip-api.com` (free tier sekarang mendukung HTTPS) atau hapus fallback dan hanya pakai `ipapi.co`.

### 7.2 [Low] `shelf_open` buka file tanpa scope

- **Lokasi:** `src-tauri/src/commands.rs:75`
- **Masalah:** `open_path` bisa membuka file apapun yang path-nya ada di shelf. Path berasal dari drag-drop (user intent) & disimpan di `localStorage`.
- **Risiko:** rendah — CSP memblokir injeksi, tapi kalau localStorage termodifikasi bisa trigger open arbitrary file.
- **Fix:** tambahkan konfirmasi atau allowlist ekstensi file.

### 7.3 [Info] Diagnostics log lokal

- **Lokasi:** `%LOCALAPPDATA%/com.sehaz.bloom/diagnostics.log`
- **Isi:** event startup/shutdown, panic, AppBar registration, dll. Maks 1 MB, auto-rotate.
- **Dikirim keluar?** Tidak. Hanya lokal.

### 7.4 [Info] Dev dependencies punya vulnerability

- 13 finding (5 high, 5 moderate, 3 low) semuanya di build-time deps (vite 7.3.6, esbuild, browserslist, picomatch, source-map-js).
- Tidak masuk ke app user. Bisa di-update dengan `bun update` sebelum rilis berikutnya.

## 8. Hak Akses yang Diminta di Windows

Bloom adalah app desktop yang mereplace taskbar, jadi wajar meminta:
- Global keyboard hook (Win+1..9)
- Akses audio loopback (visualizer)
- Enum window + thumbnail (dock preview)
- Akses WiFi + Bluetooth (status panel)
- AppBar registration (sembunyikan taskbar native)

Semua di atas **tidak memerlukan Administrator** dan tidak mengakses:
- ❌ Mikrofon / kamera
- ❌ File browser / password manager
- ❌ Wallet / credential vault
- ❌ Clipboard (kecuali explicit `clipboard-manager:allow-write-text` untuk fitur tertentu)
- ❌ Lokasi GPS (hanya inferensi IP)

## 9. Verifikasi Mandiri

```powershell
# Hash installer
certutil -hashfile bloom_3.10.1_x64-setup.exe SHA256
# Harus: 5cb4d8fe87aaf26f161be61954aebfacb4c9f481b12c5bbcfc48aa6b56db7231

# Cek endpoint outbound (saat app berjalan)
Get-NetTCPConnection -State Established | Select-Object RemoteAddress, OwningProcess
```

---

**Ringkasan:** Bloom v3.10.1 aman dipakai. Tidak ada pencurian data, malware, atau backdoor. Semua akses data sensitif bersifat fungsional untuk tugas taskbar/notch, dan tidak ada telemetry. Yang perlu diperbaiki hanya HTTP→HTTPS untuk fallback geolocation IP.
