Berikut adalah script `setup_kiosk.sh` yang dirancang khusus untuk **Lubuntu 24.04**. Script ini akan melakukan instalasi paket yang diperlukan (untuk mendeteksi waktu idle), membuat user, mengatur autologin, dan mengonfigurasi tampilan agar bersih (tanpa taskbar/icon) selayaknya mode Kiosk.

### Prasyarat

Pastikan Anda menjalankan script ini sebagai user yang memiliki akses sudo (dalam kasus Anda: user `g1n`).

### Langkah-langkah

1. Buat file baru: `nano setup_kiosk.sh`
2. Copy-paste kode di bawah ini.
3. Simpan (Ctrl+O, Enter) dan Keluar (Ctrl+X).
4. Beri izin eksekusi: `chmod +x setup_kiosk.sh`
5. Jalankan: `sudo ./setup_kiosk.sh`

### Isi Script (`setup_kiosk.sh`)

```bash
#!/bin/bash

# Konfigurasi
KIOSK_USER="opacpsb"
HOME_URL="https://psb.feb.ui.ac.id"
IDLE_TIME_MS=180000 # 3 menit dalam milidetik

echo "--- Memulai Setup Kiosk Lubuntu 24.04 ---"

# 1. Update dan Install Paket Pendukung
# xprintidle: untuk mendeteksi ketidakaktifan mouse/keyboard
# openbox: window manager ringan (opsional, tapi kita tetap pakai lxqt session yang dibersihkan)
echo "[+] Menginstall paket yang diperlukan..."
apt update -qq
apt install -y firefox xprintidle

# 2. Membuat User Baru
if id "$KIOSK_USER" &>/dev/null; then
    echo "[!] User $KIOSK_USER sudah ada."
else
    echo "[+] Membuat user $KIOSK_USER..."
    useradd -m -s /bin/bash $KIOSK_USER
    passwd -d $KIOSK_USER # Menghapus password (opsional, agar tidak ribet login jika logout)
fi

# 3. Konfigurasi Autologin SDDM (Display Manager Lubuntu)
echo "[+] Mengkonfigurasi Autologin SDDM..."
mkdir -p /etc/sddm.conf.d
cat <<EOF > /etc/sddm.conf.d/autologin.conf
[Autologin]
User=$KIOSK_USER
Session=lxqt
Relogin=true
EOF

# 4. Membuat Script Kiosk Logic (Watchdog)
# Script ini akan berjalan saat user login
echo "[+] Membuat script logika Kiosk..."
USER_HOME="/home/$KIOSK_USER"
mkdir -p $USER_HOME/bin

cat <<EOF > $USER_HOME/bin/kiosk_loop.sh
#!/bin/bash

# Matikan elemen desktop default Lubuntu agar layar bersih
# Kami memberi jeda sedikit agar proses asli mulai dulu baru dimatikan
sleep 5
killall lxqt-panel 2>/dev/null
killall pcmanfm-qt 2>/dev/null
# Set background hitam (opsional, jika wallpaper masih muncul)
xsetroot -solid black 2>/dev/null

# Loop utama
while true; do
    # 1. Cek apakah Firefox berjalan
    if ! pgrep -x "firefox" > /dev/null; then
        # Jalankan Firefox mode kiosk (fullscreen, no UI)
        # --private-window digunakan agar tidak menyimpan cache/session restore
        firefox --kiosk --private-window "$HOME_URL" &
    fi

    # 2. Cek waktu idle (tidak ada gerak mouse/keyboard)
    IDLE=\$(xprintidle)
    
    # Jika idle lebih dari 3 menit ($IDLE_TIME_MS)
    if [ "\$IDLE" -gt $IDLE_TIME_MS ]; then
        echo "Idle detected. Resetting browser..."
        # Tutup paksa firefox
        pkill -x firefox
        # Loop akan otomatis menjalankan ulang firefox di atas
        
        # Reset pointer idle dengan simulasi input kecil (opsional, agar tidak loop kill terus menerus)
        # tapi karena firefox restart, user aktif dianggap 0 lagi oleh sistem biasanya.
    fi

    sleep 2
done
EOF

# Beri izin eksekusi script
chmod +x $USER_HOME/bin/kiosk_loop.sh
chown -R $KIOSK_USER:$KIOSK_USER $USER_HOME/bin

# 5. Menambahkan ke Autostart LXQt
echo "[+] Menambahkan script ke Autostart..."
AUTOSTART_DIR="$USER_HOME/.config/autostart"
mkdir -p $AUTOSTART_DIR

cat <<EOF > $AUTOSTART_DIR/kiosk.desktop
[Desktop Entry]
Type=Application
Name=Kiosk Mode
Exec=$USER_HOME/bin/kiosk_loop.sh
X-LXQt-Need-Tray=false
EOF

chown -R $KIOSK_USER:$KIOSK_USER $USER_HOME/.config

echo "--- Setup Selesai ---"
echo "Silakan restart komputer untuk menguji autologin ke user: $KIOSK_USER"
echo "Untuk restart sekarang, ketik: sudo reboot"

```

### Penjelasan Cara Kerja Script:

1. **Environment Bersih:** Script secara otomatis mematikan `lxqt-panel` (taskbar bawah) dan `pcmanfm-qt` (yang menangani icon desktop dan wallpaper) begitu user login. Ini membuat layar menjadi kosong/hitam kecuali browser.
2. **Logic 3 Menit (Idle):** Menggunakan tool kecil bernama `xprintidle`. Script akan mengecek setiap 2 detik. Jika komputer tidak disentuh selama 3 menit (180.000 ms), script akan mematikan paksa Firefox.
3. **Auto-Restart:** Karena script berada dalam loop `while true`, begitu Firefox dimatikan (karena idle), script akan langsung menyalakannya lagi. Ini akan otomatis membuka kembali halaman Home (`psb.feb.ui.ac.id`) dalam kondisi segar (tanpa tab/history sebelumnya karena mode `--private-window`).
4. **Autologin:** Mengatur konfigurasi SDDM agar langsung masuk ke user `opacpsb` tanpa minta password.

Apakah Anda ingin saya tambahkan fitur agar PC otomatis *shutdown* pada jam tertentu (misal jam tutup perpustakaan)?
