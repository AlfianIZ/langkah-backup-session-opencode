# Pemulihan Sesi opencode

Backup ini dibuat dari **opencode 1.19.0** pada 15 November 2026 dan diperbarui pada 16 November 2026.

## Isi backup

| File | Isi |
|---|---|
| `opencode-data.tar.gz` | `~/.local/share/opencode` — database sesi, storage, tool output, log |
| `opencode-state.tar.gz` | `~/.local/state/opencode` — prompt history dan pilihan model |
| `opencode-config.tar.gz` | `~/.config/opencode` — konfigurasi dan plugin |
| `SHA256SUMS` | Checksum ketiga arsip |

Verifikasi saat refresh terakhir:

- Integritas database: `ok`
- Sesi: **11**
- Pesan: **475**
- Part: **1.737**

## Prasyarat

- Pop!_OS sudah terpasang.
- opencode **1.19.0 atau lebih baru** sudah diinstal.
- **Jangan jalankan opencode sebelum proses restore selesai**, karena opencode akan membuat database baru.
- Backup berada di partisi NTFS dengan label `New Volume`. Di Windows biasanya terlihat sebagai `D:\backupSessionOpenCode` (huruf drive bisa berbeda).

## Langkah pemulihan

### 1. Mount partisi backup

Jika partisi belum termount otomatis:

```bash
sudo mkdir -p /mnt/winbackup
sudo mount -t ntfs-3g -o ro /dev/nvme0n1p5 /mnt/winbackup

DEST="/mnt/winbackup/backupSessionOpenCode"
```

Jika sudah termount otomatis di `/media/alfian/New Volume`:

```bash
DEST="/media/alfian/New Volume/backupSessionOpenCode"
```

### 2. Verifikasi checksum

```bash
cd "$DEST"
sha256sum -c SHA256SUMS
```

Expected:

```text
opencode-data.tar.gz: OK
opencode-state.tar.gz: OK
opencode-config.tar.gz: OK
```

Checksum yang benar:

```text
9bcf064f184490a1ce2b0544f9b449536037f258d1369e9b78039ab9191a467d  opencode-data.tar.gz
c1b4582927db9bf3f9fc91b4f97085f473f53478c9de66ae9c9052112764ba8a  opencode-state.tar.gz
74d9b6423f991ff5e25c7b8202385b0e39d833db9941401e3da16a40c1248572  opencode-config.tar.gz
```

### 3. Pastikan opencode tidak berjalan

```bash
pgrep -af opencode || true
```

Jika masih ada proses:

```bash
pkill -TERM -x opencode
sleep 2
pgrep -af opencode || echo "opencode sudah berhenti"
```

### 4. Amankan data lama (jika ada)

Langkah ini mencegah data baru menimpa backup.

```bash
STAMP=$(date +%Y%m%d-%H%M%S)

for d in \
  "$HOME/.local/share/opencode" \
  "$HOME/.local/state/opencode" \
  "$HOME/.config/opencode"
do
  [ -e "$d" ] && mv "$d" "$d.before-restore-$STAMP"
done
```

### 5. Ekstrak backup

```bash
mkdir -p "$HOME/.local/share" "$HOME/.local/state" "$HOME/.config"

tar -C "$HOME/.local/share" -xzf "$DEST/opencode-data.tar.gz"
tar -C "$HOME/.local/state" -xzf "$DEST/opencode-state.tar.gz"
tar -C "$HOME/.config"       -xzf "$DEST/opencode-config.tar.gz"
```

### 6. Perbaiki ownership dan permission

```bash
sudo chown -R "$USER:$USER" \
  "$HOME/.local/share/opencode" \
  "$HOME/.local/state/opencode" \
  "$HOME/.config/opencode"

chmod 700 \
  "$HOME/.local/share/opencode" \
  "$HOME/.local/state/opencode" \
  "$HOME/.config/opencode"

chmod 600 "$HOME/.local/share/opencode/opencode.db"
```

### 7. Verifikasi hasil restore

```bash
opencode db "select count(*) from session" --format tsv
opencode db "pragma integrity_check" --format tsv
```

Alternatif dengan Python:

```bash
python3 - <<'PY'
import sqlite3, os

p = os.path.expanduser("~/.local/share/opencode/opencode.db")
con = sqlite3.connect(p)
print("integrity:", con.execute("pragma integrity_check").fetchone()[0])
print("sessions :", con.execute("select count(*) from session").fetchone()[0])
print("messages :", con.execute("select count(*) from message").fetchone()[0])
print("parts    :", con.execute("select count(*) from part").fetchone()[0])
con.close()
PY
```

Expected:

```text
integrity: ok
sessions : 11
messages : 475
parts    : 1737
```

### 8. Jalankan opencode

```bash
opencode
```

Cek daftar sesi:

```bash
opencode session list
```

### 9. Login provider (opsional)

Saat backup dibuat, `opencode auth list` menunjukkan **0 credentials**, jadi tidak ada login yang perlu dipulihkan. Model gratis (termasuk model berakhiran `-free`) dapat dipakai tanpa login.

Jika nanti menggunakan provider berbayar:

```bash
opencode auth login
```

Atau gunakan `/connect` di dalam TUI opencode.

Kredensial opencode 1.19.0 disimpan di `~/.local/share/opencode/auth.json` (bukan keyring sistem). File ini tidak ada di backup karena belum ada kredensial saat backup dibuat. Setelah login, file tersebut berisi token/API key; jangan disalin sembarangan. Paling aman login ulang di sistem baru.

## Troubleshooting

### Database terkunci

Pastikan semua proses opencode sudah berhenti:

```bash
pgrep -af opencode
pkill -TERM -x opencode
```

### Error skema database atau migrasi

Gunakan opencode **1.19.0 atau lebih baru**. Jangan memakai versi lebih lama dari versi pembuat backup.

### Plugin gagal dimuat

Jika opencode yang dipakai lebih baru dan plugin `@opencode-ai/plugin` tidak kompatibel:

```bash
rm -rf "$HOME/.config/opencode/node_modules"
cd "$HOME/.config/opencode" && npm install
```

### Sesi tidak muncul

Periksa bahwa ekstraksi `opencode-data.tar.gz` menghasilkan:

```text
~/.local/share/opencode/opencode.db
~/.local/share/opencode/storage/
~/.local/share/opencode/tool-output/
```

Lalu ulangi pengecekan jumlah sesi pada langkah 7.

### Ingin memulihkan sesi saja

Cukup ekstrak `opencode-data.tar.gz`. Arsip `state` dan `config` bersifat opsional.

## Catatan

- Snapshot database dibuat dengan SQLite backup API, jadi konsisten meskipun saat backup opencode sedang berjalan.
- Arsip data **tidak memuat** `opencode.db-wal` dan `opencode.db-shm`; database di dalam arsip sudah berdiri sendiri.
- Jangan mencampur database hasil restore dengan file `opencode.db-wal` dari sumber lain.
- `~/.local/share/opencode/auth.json` tidak disertakan karena saat backup belum ada kredensial; backup ini hanya memuat sesi dan konfigurasi.
- Setelah selesai, unmount partisi backup:

  ```bash
  sudo umount /mnt/winbackup
  ```
