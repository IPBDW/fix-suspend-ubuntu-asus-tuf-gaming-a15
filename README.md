# Fix suspend Ubuntu — ASUS TUF Gaming A15 FA506NFR

Tutup lid → laptop nge-hang di suspend: layar mati tapi keyboard masih menyala, panas naik, baterai boros, dan satu-satunya jalan keluar hard reset. Akar masalahnya ada di jalur S0ix NVIDIA, dan perbaikannya satu baris.

## Mesin yang terkena

| Komponen | Nilai |
|---|---|
| Laptop | ASUS TUF Gaming A15 FA506NFR, BIOS 304 |
| CPU | AMD Ryzen 7 7435HS, 8 core / 16 thread, **tanpa iGPU** |
| GPU | NVIDIA RTX 2050 GA107 `[10de:25ad]`, dGPU-only |
| OS | Ubuntu 26.04.1 LTS, kernel 7.0.0-43-generic |
| Driver | NVIDIA proprietary 595.99.02 |

Kasus identik (model, CPU, dan GPU yang sama persis) tercatat di [Omarchy PR #8546](https://github.com/omacom/omarchy/pull/8546), jadi fix ini kemungkinan besar berlaku untuk FA506NFR lain dengan kombinasi hardware yang sama.

## Gejala

- Lid ditutup, layar mati, tapi keyboard masih menyala dan mesin tetap panas.
- Journal berhenti di `PM: suspend entry (s2idle)` tanpa `PM: suspend exit` yang menyertainya.
- Boot berikutnya fsck membersihkan `Clearing orphaned inode ...` — tanda hard power-off, bukan shutdown bersih.
- `last_hw_sleep` hanya ~0,1 detik: kernel masuk s2idle lalu hang, hardware praktis tidak pernah tidur.

## Diagnosa

Jalankan ini dulu sebelum mengubah apa pun:

```bash
cat /sys/power/mem_sleep
# [s2idle]   <- hanya s2idle, tidak ada deep/S3

journalctl --no-pager | grep 'ACPI: PM: (supports'
# ACPI: PM: (supports S0 S4 S5)   <- firmware tidak punya S3

cat /proc/driver/nvidia/gpus/*/power
# Runtime D3 status:          Enabled (fine-grained)
#  Video Memory Self Refresh: Supported
# S0ix Power Management:
#  Platform Support:          Supported
#  Status:                    Disabled     <- ini masalahnya

grep -R PreserveVideoMemory /etc/modprobe.d/
# options nvidia NVreg_PreserveVideoMemoryAllocations=1
```

Ciri khasnya: GPU dan platform **mendukung** S0ix, tapi statusnya **Disabled**, sementara driver disuruh menyimpan VRAM lewat jalur preserve.

## Akar masalah

Dengan `NVreg_PreserveVideoMemoryAllocations=1`, setiap kali suspend driver NVIDIA menyimpan isi VRAM melalui panggilan GSP RPC. Karena dGPU berada di runtime D3 (`Enabled (fine-grained)`), GSP kadang berhenti merespons. Proses suspend lalu nongkrong di dalam PM notifier driver dan tidak pernah kembali — mesin tidak pernah mencapai low power, layar mati tapi sistem tetap hidup dan panas.

Ini **bukan** masalah hybrid graphics: Ryzen 7 7435HS tidak punya iGPU, dan kasus yang sama justru terjadi di laptop dGPU-only.

## Perbaikan

Tambahkan opsi S0ix ke modprobe, regenerate initramfs, lalu reboot:

```bash
echo 'options nvidia NVreg_EnableS0ixPowerManagement=1' | \
  sudo tee /etc/modprobe.d/nvidia-s0ix.conf

sudo update-initramfs -u
sudo systemctl reboot -i
```

`systemctl reboot -i` dipakai karena GNOME bisa menolak `sudo reboot` dengan pesan `Operation inhibited by ... user session inhibited`. Simpan kerjaan dan tutup browser dulu: `-i` mengabaikan semua inhibitor, termasuk "Saving browser state".

Di distro dengan `mkinitcpio` (Arch/CachyOS), ganti langkah initramfs dengan `sudo mkinitcpio -P`; dengan dracut, `sudo dracut --regenerate-all --force`.

Dengan opsi ini, VRAM di bawah threshold 256 MB dibiarkan dalam self-refresh saat suspend, sehingga evansi VRAM lewat GSP RPC tidak terjadi.

## Verifikasi

Setelah reboot:

```bash
grep -i 'Status' /proc/driver/nvidia/gpus/*/power
# Status: Enabled     <- harus berubah dari Disabled

for f in success fail last_hw_sleep; do
  printf '%s: ' "$f"; cat /sys/power/suspend_stats/$f
done
# success: 1
# fail: 0
# last_hw_sleep: 5599609    <- mikrodetik, ~5,6 detik beneran tidur

journalctl -b --no-pager | grep 'PM: suspend entry\|PM: suspend exit'
```

Tutup-buka lid 3–4 kali. Setiap `PM: suspend entry` harus punya `PM: suspend exit` pasangan. Anggap beres hanya kalau `last_hw_sleep` naik dan laptop bisa dibangunkan tanpa hard reset.

## Yang tidak akan membantu

- **`mem_sleep_default=deep`** — firmware FA506NFR hanya menyediakan S0/S4/S5, tidak ada S3.
- **Mengubah `logind.conf` / GNOME power settings** — journal tidak punya `PM: suspend exit`, berarti masalahnya di kernel/driver, bukan di siapa yang meminta sleep.
- **Hibernate sebagai pengganti** — di mesin ini `/sys/power/disk` = `[disabled]`: swap cuma 4 GB (RAM 14 GiB), Secure Boot aktif dengan lockdown `[integrity]`, dan tidak ada `resume=UUID=...` di kernel cmdline.
- **`NVreg_DynamicPowerManagement=0`** — dilaporkan memperparah kasus related pada kernel 7.x; jangan dicoba sebagai fix pertama.

## Catatan

- Jangan masukkan laptop ke tas saat masih dalam kondisi suspend. Kalau kebangun sendiri di dalam tas, panas tidak punya jalan keluar. Untuk dibawa, shutdown total.
- File `nvidia-s0ix.conf` sengaja tidak pakai awalan angka. File modprobe dibaca berurutan secara leksikal dan yang terakhir menang; nama ini sortirnya setelah `nvidia-graphics-drivers-kms.conf` dari distro.

## Sumber

- [Omarchy PR #8546 — Enable NVIDIA S0ix power management on s2idle systems](https://github.com/omacom/omarchy/pull/8546) (berisi hardware FA506NFR / RTX 2050 / Ryzen 7 7435HS yang identik).
- [NVIDIA Linux driver README 595.99.02 — Configuring Power Management Support](https://http.download.nvidia.com/XFree86/Linux-x86_64/595.99.02/README/powermanagement.html).
- [ArchWiki — NVIDIA](https://wiki.archlinux.org/title/NVIDIA) dan [ASUS Linux](https://wiki.archlinux.org/title/ASUS_Linux).
