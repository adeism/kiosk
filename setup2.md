1. **Gateway** diubah menjadi `152.118.24.2`.
2. **Waktu Idle** diubah menjadi **15 menit** (900.000 ms).
3. **Penjelasan Dynamic IP** sudah disertakan dalam komentar script agar mudah dipahami.

Silakan copy-paste script ini ke file baru (misal: `setup_kiosk_final_v2.sh`) dan jalankan dengan `sudo`.

```bash
#!/bin/bash

# ==============================================================================
# SCRIPT SETUP LUBUNTU 24.04 KIOSK (FINAL V2 - 15 MIN IDLE)
# ==============================================================================

# --- [BAGIAN 1] KONFIGURASI (EDIT DI SINI) ---

# >> KONFIGURASI JARINGAN <<
# CARA UBAH KE DYNAMIC IP (DHCP):
# Ubah nilai NET_MODE di bawah ini dari "static" menjadi "dynamic".
# Contoh: NET_MODE="dynamic"
# (Jika memilih "dynamic", settingan IP, Prefix, Gateway diabaikan otomatis)

NET_MODE="static"

# Detail Static IP (Hanya terpakai jika NET_MODE="static"):
NET_IP="152.118.116.112"
NET_PREFIX="24"               # Subnet mask /24
NET_GATEWAY="152.118.24.2"    # <--- Gateway Baru
NET_DNS="8.8.8.8, 1.1.1.1"    # DNS Google/Cloudflare

# >> KONFIGURASI KIOSK <<
KIOSK_USER="opacpsb"
HOME_URL="https://psb.feb.ui.ac.id"

# Waktu Idle sebelum restart browser (dalam milidetik)
# Rumus: Menit x 60 x 1000
# 15 Menit = 15 * 60 * 1000 = 900000
IDLE_TIME_MS=900000           # <--- 15 Menit

# ------------------------------------------------------------------------------

# Cek Root
if [ "$EUID" -ne 0 ]; then
  echo "ERROR: Harap jalankan script ini dengan sudo!"
  exit
fi

echo "--- [1/8] Mengkonfigurasi Jaringan ($NET_MODE) ---"

# Mendeteksi nama interface network secara otomatis
NET_INT=$(ip -o link show | awk -F': ' '{print $2}' | grep -v "lo" | head -n 1)

if [ -z "$NET_INT" ]; then
    echo "WARNING: Interface network tidak ditemukan! Melewati konfigurasi network."
else
    echo "Interface terdeteksi: $NET_INT"
    
    # Hapus konfigurasi netplan lama agar bersih
    rm -f /etc/netplan/*.yaml

    if [ "$NET_MODE" == "static" ]; then
        # === KONFIGURASI STATIC ===
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
        echo "Konfigurasi STATIC diterapkan: IP $NET_IP | GW $NET_GATEWAY"

    else
        # === KONFIGURASI DYNAMIC (DHCP) ===
        cat <<EOF > /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    $NET_INT:
      dhcp4: true
EOF
        echo "Konfigurasi DYNAMIC (DHCP) diterapkan."
    fi

    # Terapkan Netplan
    chmod 600 /etc/netplan/01-netcfg.yaml
    netplan apply
    sleep 5 # Tunggu koneksi refresh
fi

echo "--- [2/8] Mematikan Fitur Sleep & Suspend Sistem (Permanen) ---"
# Mematikan sleep level kernel/systemd
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
# Mematikan sleep level user session
gsettings set org.gnome.desktop.session idle-delay 0 2>/dev/null

echo "--- [3/8] Membersihkan Notifikasi & Update Notifier ---"
apt remove -y update-notifier update-notifier-common gnome-software-plugin-snap unattended-upgrades

echo "--- [4/8] Mengkonfigurasi Firefox (No Updates/Popups) ---"
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
# Geser jadwal update Snap ke jam 3 pagi (Maintenance Window)
snap set system refresh.timer=03:00-05:00

echo "--- [5/8] Install Paket & Membuat User ---"
apt update -qq
apt install -y firefox xprintidle

if id "$KIOSK_USER" &>/dev/null; then
    echo "User $KIOSK_USER sudah ada."
else
    useradd -m -s /bin/bash $KIOSK_USER
    passwd -d $KIOSK_USER
fi

echo "--- [6/8] Konfigurasi Autologin SDDM ---"
mkdir -p /etc/sddm.conf.d
cat <<EOF > /etc/sddm.conf.d/autologin.conf
[Autologin]
User=$KIOSK_USER
Session=lxqt
Relogin=true
EOF

echo "--- [7/8] Membuat Logic Script (Watchdog 15 Menit) ---"
USER_HOME="/home/$KIOSK_USER"
mkdir -p $USER_HOME/bin

cat <<EOF > $USER_HOME/bin/kiosk_loop.sh
#!/bin/bash

# === SETTING DISPLAY: PAKSA ON (ANTI-SLEEP) ===
xset s off
xset s noblank
xset -dpms

# === BERSIHKAN DESKTOP ===
sleep 5
killall lxqt-panel 2>/dev/null
killall pcmanfm-qt 2>/dev/null
killall lxqt-notificationd 2>/dev/null
xsetroot -solid black 2>/dev/null

# === LOOP UTAMA ===
while true; do
    
    # 1. Pastikan Firefox Hidup
    if ! pgrep -x "firefox" > /dev/null; then
        firefox --kiosk --private-window "$HOME_URL" &
    fi

    # 2. Cek Idle Time
    IDLE=\$(xprintidle)
    
    # Cek apakah idle melebihi batas waktu (IDLE_TIME_MS dari variabel setup)
    # Nilai hardcode di sini disesuaikan dengan input script setup ($IDLE_TIME_MS)
    if [ "\$IDLE" -gt $IDLE_TIME_MS ]; then
        echo "Idle 15 menit tercapai. Restarting browser..."
        pkill -x firefox
        # Tunggu sampai benar-benar mati (cegah black screen)
        while pgrep -x "firefox" > /dev/null; do 
            sleep 0.5
        done
    fi

    # 3. REFRESH DISPLAY POWER (Agar layar tidak pernah mati)
    xset s off -dpms
    
    sleep 2
done
EOF

chmod +x $USER_HOME/bin/kiosk_loop.sh
chown -R $KIOSK_USER:$KIOSK_USER $USER_HOME/bin

echo "--- [8/8] Menambahkan ke Autostart ---"
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

echo "==================================================="
echo " SETUP SELESAI (VERSI FINAL V2)"
echo " Network Mode : $NET_MODE"
echo " Gateway      : $NET_GATEWAY"
echo " Idle Reset   : 15 Menit"
echo " Silakan restart: sudo reboot"
echo "==================================================="

```

### Panduan Mengubah ke Dynamic IP (DHCP)

Untuk mengubah settingan ke DHCP di masa depan, Anda tidak perlu menulis ulang seluruh script Netplan manual. Cukup edit baris paling atas script ini:

**Cari baris:**

```bash
NET_MODE="static"

```

**Ubah menjadi:**

```bash
NET_MODE="dynamic"

```

Lalu jalankan ulang scriptnya. Script akan mendeteksi perubahan variabel tersebut dan otomatis membuat konfigurasi Netplan yang menggunakan `dhcp4: true` serta mengabaikan isian IP/Gateway static di bawahnya.
