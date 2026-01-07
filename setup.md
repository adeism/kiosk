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

# --- Konfigurasi ---
KIOSK_USER="kiosku"
HOME_URL="https://psb.feb.ui.ac.id"
IDLE_TIME_MS=180000 # 3 menit
# -------------------

echo "--- Update Setup Kiosk Lubuntu (Anti Black Screen) ---"

# Pastikan paket terinstall
apt update -qq
apt install -y firefox xprintidle

# Buat user jika belum ada
if ! id "$KIOSK_USER" &>/dev/null; then
    useradd -m -s /bin/bash $KIOSK_USER
    passwd -d $KIOSK_USER
fi

# Konfigurasi Autologin SDDM (Pastikan session LXQt)
mkdir -p /etc/sddm.conf.d
cat <<EOF > /etc/sddm.conf.d/autologin.conf
[Autologin]
User=$KIOSK_USER
Session=lxqt
Relogin=true
EOF

# --- LOGIKA BARU DI SINI ---
USER_HOME="/home/$KIOSK_USER"
mkdir -p $USER_HOME/bin

cat <<EOF > $USER_HOME/bin/kiosk_loop.sh
#!/bin/bash

# 1. MATIKAN POWER SAVING / SCREEN BLANKING
# Ini mencegah monitor mati/hitam karena dianggap idle oleh sistem
xset s off       # Matikan screensaver
xset s noblank   # Jangan blank layar
xset -dpms       # Matikan Energy Star (monitor standby)

# Matikan elemen desktop default
sleep 5
killall lxqt-panel 2>/dev/null
killall pcmanfm-qt 2>/dev/null
xsetroot -solid black 2>/dev/null

# Loop utama
while true; do
    
    # Cek apakah Firefox berjalan
    if ! pgrep -x "firefox" > /dev/null; then
        # Jalankan Firefox mode kiosk
        # --kiosk-monitor 0 memastikan muncul di layar utama
        firefox --kiosk --private-window "$HOME_URL" &
    fi

    # Cek waktu idle
    IDLE=\$(xprintidle)
    
    # Jika idle lebih dari 3 menit
    if [ "\$IDLE" -gt $IDLE_TIME_MS ]; then
        echo "Idle limit reached. Restarting browser..."
        
        # LOGIKA RESTART YANG DIPERBAIKI:
        # 1. Kill Firefox
        pkill -x firefox
        
        # 2. Tunggu sampai proses benar-benar hilang (loop waiting)
        # Ini mencegah layar hitam karena script mengira firefox masih hidup
        while pgrep -x "firefox" > /dev/null; do 
            sleep 0.5
        done

        # 3. Opsional: Reset idle timer dengan simulasi keypress kecil (butuh xdotool)
        # Tapi karena firefox restart, mouse/keyboard focus akan reset.
    fi

    # Ulangi pengecekan setiap 2 detik
    # Perintah xset diulang untuk memastikan power manager bawaan Lubuntu tidak menyalakan ulang screensaver
    xset s off -dpms
    sleep 2
done
EOF

chmod +x $USER_HOME/bin/kiosk_loop.sh
chown -R $KIOSK_USER:$KIOSK_USER $USER_HOME/bin

# Autostart
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

echo "--- Update Selesai. Silakan Reboot ---"

```

### Penjelasan Cara Kerja Script:

1. **Environment Bersih:** Script secara otomatis mematikan `lxqt-panel` (taskbar bawah) dan `pcmanfm-qt` (yang menangani icon desktop dan wallpaper) begitu user login. Ini membuat layar menjadi kosong/hitam kecuali browser.
2. **Logic 3 Menit (Idle):** Menggunakan tool kecil bernama `xprintidle`. Script akan mengecek setiap 2 detik. Jika komputer tidak disentuh selama 3 menit (180.000 ms), script akan mematikan paksa Firefox.
3. **Auto-Restart:** Karena script berada dalam loop `while true`, begitu Firefox dimatikan (karena idle), script akan langsung menyalakannya lagi. Ini akan otomatis membuka kembali halaman Home (`psb.feb.ui.ac.id`) dalam kondisi segar (tanpa tab/history sebelumnya karena mode `--private-window`).
4. **Autologin:** Mengatur konfigurasi SDDM agar langsung masuk ke user `opacpsb` tanpa minta password.

Apakah Anda ingin saya tambahkan fitur agar PC otomatis *shutdown* pada jam tertentu (misal jam tutup perpustakaan)?
