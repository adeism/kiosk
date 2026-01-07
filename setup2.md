Berikut adalah **Script Final V3** dengan perubahan sesuai permintaan Anda:

1. **Gateway:** Diubah ke `152.118.116.1`.
2. **DNS:** Diubah ke `152.118.24.2` (Sepertinya ini DNS lokal UI).
3. **Auto Shutdown:** Diset ke jam **16:00** (Jam 4 Sore).

### Script: `setup_kiosk_opacku_v3.sh`

Silakan copy, simpan, dan jalankan dengan `sudo`.

```bash
#!/bin/bash

# ==============================================================================
# SCRIPT SETUP LUBUNTU 24.04 KIOSK (FINAL V3)
# USER: opacku
# ==============================================================================

# --- [BAGIAN 1] KONFIGURASI (EDIT DI SINI) ---

# >> A. JARINGAN
NET_MODE="static"             # "static" atau "dynamic"

# Detail IP (Hanya dipakai jika mode static)
NET_IP="152.118.116.112"
NET_PREFIX="24"
NET_GATEWAY="152.118.116.1"   # <--- Gateway Baru
NET_DNS="152.118.24.2"        # <--- DNS Baru (Lokal)

# >> B. KIOSK SETTINGS
KIOSK_USER="opacku"
HOME_URL="https://psb.feb.ui.ac.id"
IDLE_TIME_MINUTES=15          # Reset browser jika diam 15 menit

# >> C. SCHEDULED SHUTDOWN
# Masukkan jam (format 24 jam) kapan PC harus mati.
# "16" berarti tepat jam 16:00 (4 sore) PC akan shutdown.
SHUTDOWN_HOUR="16"

# ------------------------------------------------------------------------------

# Hitung milidetik untuk idle
IDLE_TIME_MS=$((IDLE_TIME_MINUTES * 60 * 1000))

if [ "$EUID" -ne 0 ]; then
  echo "Harap jalankan dengan SUDO!"
  exit
fi

echo "==================================================="
echo " SETUP KIOSK 'opacku' V3 (Auto Shutdown 16:00)"
echo "==================================================="

# --- [1/8] Setting Jaringan ---
echo "[+] Konfigurasi Network: $NET_MODE"
NET_INT=$(ip -o link show | awk -F': ' '{print $2}' | grep -v "lo" | head -n 1)

if [ -z "$NET_INT" ]; then
    echo "WARNING: LAN Card tidak terdeteksi!"
else
    rm -f /etc/netplan/*.yaml

    if [ "$NET_MODE" == "static" ]; then
        # Config Static dengan DNS Baru
        cat <<EOF > /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    $NET_INT:
      dhcp4: no
      addresses:
        - $NET_IP/$NET_PREFIX
      routes:
        - to: default
          via: $NET_GATEWAY
      nameservers:
        addresses: [$NET_DNS]
EOF
        echo "   IP: $NET_IP | GW: $NET_GATEWAY | DNS: $NET_DNS"
    else
        # Config DHCP
        cat <<EOF > /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    $NET_INT:
      dhcp4: true
EOF
    fi

    chmod 600 /etc/netplan/01-netcfg.yaml
    netplan apply
    sleep 3
fi

# --- [2/8] Sistem Hardening (Anti Sleep Total) ---
echo "[+] Mematikan Sleep & Notifikasi..."
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
apt remove -y update-notifier update-notifier-common gnome-software-plugin-snap unattended-upgrades
snap set system refresh.timer=03:00-05:00

# --- [3/8] Install Paket ---
echo "[+] Install Firefox, xprintidle, unclutter..."
apt update -qq
apt install -y firefox xprintidle unclutter pulseaudio-utils

# --- [4/8] Kebijakan Firefox ---
echo "[+] Menerapkan Policy Firefox..."
mkdir -p /etc/firefox/policies
cat <<EOF > /etc/firefox/policies/policies.json
{
  "policies": {
    "DisableAppUpdate": true,
    "DisableSystemAddonUpdate": true,
    "DisableTelemetry": true,
    "DisablePocket": true,
    "UserMessaging": {
      "ExtensionRecommendations": false,
      "FeatureRecommendations": false,
      "UrlbarInterventions": false,
      "SkipOnboarding": true,
      "MoreFromMozilla": false,
      "WhatsNew": false
    },
    "OverrideFirstRunPage": "",
    "OverridePostUpdatePage": ""
  }
}
EOF

# --- [5/8] User Setup ---
if id "$KIOSK_USER" &>/dev/null; then
    echo "[!] User $KIOSK_USER sudah ada."
else
    echo "[+] Membuat User $KIOSK_USER..."
    useradd -m -s /bin/bash $KIOSK_USER
    passwd -d $KIOSK_USER
    usermod -aG audio $KIOSK_USER
fi

# --- [6/8] Autologin SDDM ---
mkdir -p /etc/sddm.conf.d
cat <<EOF > /etc/sddm.conf.d/autologin.conf
[Autologin]
User=$KIOSK_USER
Session=lxqt
Relogin=true
EOF

# --- [7/8] Logic Script (Shutdown & Watchdog) ---
echo "[+] Membuat Script Logic Kiosk..."
USER_HOME="/home/$KIOSK_USER"
mkdir -p $USER_HOME/bin

cat <<EOF > $USER_HOME/bin/kiosk_loop.sh
#!/bin/bash

# VARIABEL
LIMIT_IDLE=$IDLE_TIME_MS
SHUTDOWN_AT="$SHUTDOWN_HOUR" # Jam 16

# 1. SETUP TAMPILAN
xset s off
xset s noblank
xset -dpms
sleep 5
killall lxqt-panel 2>/dev/null
killall pcmanfm-qt 2>/dev/null
killall lxqt-notificationd 2>/dev/null
xsetroot -solid black 2>/dev/null

# Sembunyikan mouse idle 5 detik & Mute Audio
unclutter -idle 5 &
pactl set-sink-mute @DEFAULT_SINK@ 1 2>/dev/null

# 2. LOOP UTAMA
while true; do
    
    # A. CEK SHUTDOWN (JAM 16:00 KE ATAS)
    if [ ! -z "\$SHUTDOWN_AT" ]; then
        CURRENT_HOUR=\$(date +%H)
        # Jika jam sekarang >= 16 (dan bukan pagi buta misal jam 0-7, opsional logic)
        # Di sini logikanya: Jika sudah masuk jam 16, 17, 18 dst -> MATI.
        if [ "\$CURRENT_HOUR" -ge "\$SHUTDOWN_AT" ]; then
             # Tambahan: Cek biar tidak mati pas baru nyala pagi (misal salah jam)
             # Kita asumsi perpustakaan buka jam 8 pagi, jadi shutdown berlaku jam 16-23.
             if [ "\$CURRENT_HOUR" -le 23 ]; then
                echo "Jam Operasional Habis. Shutdown..."
                poweroff
             fi
        fi
    fi

    # B. CEK FIREFOX
    if ! pgrep -x "firefox" > /dev/null; then
        rm -rf /home/$KIOSK_USER/.cache/mozilla/firefox/* 2>/dev/null
        firefox --kiosk --private-window "$HOME_URL" &
    fi

    # C. CEK IDLE
    IDLE_NOW=\$(xprintidle)
    if [ "\$IDLE_NOW" -gt \$LIMIT_IDLE ]; then
        pkill -x firefox
        while pgrep -x "firefox" > /dev/null; do sleep 0.5; done
    fi

    # D. ANTI SLEEP
    xset s off -dpms
    sleep 5
done
EOF

chmod +x $USER_HOME/bin/kiosk_loop.sh
chown -R $KIOSK_USER:$KIOSK_USER $USER_HOME/bin

# --- [8/8] Autostart ---
AUTOSTART_DIR="$USER_HOME/.config/autostart"
mkdir -p $AUTOSTART_DIR
cat <<EOF > $AUTOSTART_DIR/kiosk.desktop
[Desktop Entry]
Type=Application
Name=Kiosk
Exec=$USER_HOME/bin/kiosk_loop.sh
X-LXQt-Need-Tray=false
EOF
chown -R $KIOSK_USER:$KIOSK_USER $USER_HOME/.config

echo "==================================================="
echo " SELESAI. Silakan Reboot: sudo reboot"
echo "==================================================="

```

### Penjelasan Logika Shutdown:

Dalam script di atas, saya menggunakan logika:
`if [ "$CURRENT_HOUR" -ge 16 ]; then poweroff`

Artinya:

* Begitu jam sistem menunjukkan pukul **16:00, 16:01, ... sampai 23:59**, komputer akan otomatis menjalankan perintah `poweroff`.
* Jika komputer dinyalakan kembali pada jam 16:30, ia akan hidup sebentar, script berjalan, mendeteksi jam >= 16, lalu mati lagi. Ini fitur keamanan agar tidak dipakai di luar jam kerja.
* Komputer baru bisa dipakai normal besok pagi (misal jam 08:00) karena 08 lebih kecil dari 16.
