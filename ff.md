
```bash
#!/bin/bash

# Cek root
if [ "$EUID" -ne 0 ]; then
  echo "Jalankan dengan sudo!"
  exit
fi

echo "--- Memperbaiki Profil Firefox & Startup ---"

# 1. Hentikan semua proses Firefox milik kiosk
pkill -u kiosk -f firefox
pkill -u kiosk -f browser-monitor

# 2. Reset Konfigurasi Firefox agar Bersih
# Kita akan membuat struktur folder yang standar agar Snap bisa membacanya tanpa bingung
rm -rf /home/kiosk/.mozilla
rm -rf /home/kiosk/snap/firefox/common/.mozilla # Hapus config Snap yang mungkin korup

# Buat struktur folder standar
mkdir -p /home/kiosk/.mozilla/firefox/kiosk_profile/chrome

# 3. Buat profiles.ini yang Benar (Menggunakan Relative Path)
# Ini kunci agar Firefox tidak bingung mencari path absolut
cat > /home/kiosk/.mozilla/firefox/profiles.ini <<EOF
[Profile0]
Name=default
IsRelative=1
Path=kiosk_profile
Default=1

[General]
StartWithLastProfile=1
Version=2
EOF

# 4. Inject Config (User.js & CSS) ke Profil tersebut
PROFILE_DIR="/home/kiosk/.mozilla/firefox/kiosk_profile"

# User.js (Settingan)
cat > "$PROFILE_DIR/user.js" <<EOF
user_pref("toolkit.legacyUserProfileCustomizations.stylesheets", true);
user_pref("browser.startup.homepage", "https://psb.feb.ui.ac.id");
user_pref("browser.toolbars.bookmarks.visibility", "never");
user_pref("browser.shell.checkDefaultBrowser", false);
user_pref("browser.sessionstore.resume_from_crash", false);
user_pref("browser.sessionstore.max_resumed_crashes", 0);
EOF

# UserChrome.css (Tampilan)
cat > "$PROFILE_DIR/chrome/userChrome.css" <<EOF
/* Sembunyikan URL Bar */
#urlbar-container { display: none !important; visibility: collapse !important; }
#search-container { display: none !important; }
/* Pastikan Tombol Back/Home muncul */
#nav-bar { visibility: visible !important; }
EOF

# Fix Permission lagi
chown -R kiosk:kiosk /home/kiosk/.mozilla
chown -R kiosk:kiosk /home/kiosk/snap 2>/dev/null || true

# 5. UPDATE Script Monitoring (HAPUS argument -P)
# Kita hapus "-P kiosk_profile" agar Firefox memuat profil Default yang sudah kita set di atas.
cat > /home/kiosk/.local/bin/browser-monitor.sh <<'EOF'
#!/bin/bash

TARGET_URL="https://psb.feb.ui.ac.id"
IDLE_LIMIT=300000 # 5 menit

start_firefox() {
    # HAPUS "-P kiosk_profile". Biarkan Firefox mengambil default.
    # Tambahkan --no-remote agar tidak konflik dengan instance lain jika ada
    firefox --no-remote "$TARGET_URL" &
    
    sleep 5
    # Maximize
    wmctrl -x -r firefox -b add,maximized_vert,maximized_horz
    wmctrl -x -a firefox
}

start_firefox

while true; do
    # 1. Cek apakah Firefox mati (Auto-Respawn)
    if ! pgrep -u kiosk -f "firefox" > /dev/null; then
        sleep 3
        start_firefox
        xdotool mousemove_relative 1 1
        continue
    fi

    # 2. Cek Idle
    IDLE_TIME=$(xprintidle)
    if [ "$IDLE_TIME" -gt "$IDLE_LIMIT" ]; then
        # Kill & Restart untuk bersihkan sesi
        pkill -u kiosk -f firefox
        sleep 2
        start_firefox
        xdotool mousemove_relative 1 1
    fi
    
    sleep 5
done
EOF

# Pastikan permission script tetap oke
chmod +x /home/kiosk/.local/bin/browser-monitor.sh
chown kiosk:kiosk /home/kiosk/.local/bin/browser-monitor.sh

echo "--- Perbaikan Selesai ---"
echo "Silakan restart komputer: sudo reboot"

```
