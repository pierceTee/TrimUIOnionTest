# Below are my findings from digging arounnd the TrimUI brick filesystem.
An application to track your time played on trimUI devices

## Dev notes

### 1-15-25
#### Activity Tracker

- on MinUI Piping the output of ps when a game is running vs not into txt files reveals these two processes
```

+ 3821 root      3840 S    {launch.sh} /bin/sh /mnt/SDCARD/.system/tg3040/paks/Emus/SFC.pak/launch.sh /mnt/SDCARD/Roms/Super Nintend
+ 3826 root      314m R    minarch.elf /mnt/SDCARD/.system/tg3040/cores/snes9x2005_plus_libretro.so /mnt/SDCARD/Roms/Super Nintendo

```
Unfortunately it looks like the rom name is cutoff so we can't use this correctly. 

First thoughts are to:
- Override all the Emus/../launch.sh files ( not cross plat, hacky and will mess up when MinUI updates). 
- See if I can get the full filenames from `ps` run a service to monitor for those strings. (Technically annoying to write and pretty hacky)

## Brick Firmware General

- `echo 1 >/tmp/stay_awake` to keep the device from going to sleep.

- Analyzing the usb_storage app (`usr/trimui/apps/usb_storage`) it would be sweet if I could mount the brick as an external storage device to swap files in and out.

- Brick used OpenWRT where all the system package information is stored in `/usr/lib/opkg/info/`, files here tell OpenWRT what files are associated with what packages, eg `/usr/lib/opkg/info/wifimanager-demo.list` has all the associated packages to `wifimanager`

- All OS-level resources are in `/usr/trimui/res`, have fun swapping images and fonts out!

To get device info 
```

root@TinaLinux:/tmp# cat sysinfo/board_name
allwinner,a133
root@TinaLinux:/tmp# cat sysinfo/model
sun50iw10

```
## Firmware

The brick is running OpenWRT, a...router firmware aimed to give folks a fully configurable install but it primarliy focuses on networking funtionality, this is interesting because it is a single user OS (only root) so you can really get in there and mess things up, and it doesn't use any of the standard package managers. 

### Available commands (from /bin)
### System Commands

#### Core Utils

- `ash`, `busybox`, `sh` - Shells
- `cat`, `cp`, `ls`, `mkdir`, `mv`, `rm`, `rmdir` - File operations
- `chmod`, `chgrp`, `chown` - Permissions
- `date`, `df`, `dmesg`, `hostname` - System info
- `mount`, `umount` - Filesystem management
- `sync`, `sleep`, `usleep` - System control
- `touch`, `ln`, `mknod`, `mktemp` - File creation

#### Text Processing
- `base64` - Encoding
- `grep`, `egrep`, `fgrep`, `sed` - Text searching/manipulation
- `tar`, `gzip`, `gunzip`, `zcat` - Compression

#### Network Tools
- `adb_shell`, `adbd` - Android debug
- `ping`, `ping6` - Network testing
- `traceroute`, `traceroute6` - Route tracing
- `netstat`, `netmsg` - Network statistics
- `wget`, `uclient-fetch` - Download tools

#### Process Management
- `kill`, `pidof`, `ps` - Process control
- `mpstat`, `nice` - Process monitoring
- `lock`, `login` - Access control

#### WiFi Management
```
wifi_connect_ap_test               wifi_on_test
wifi_connect_ap_with_netid_test   wifi_reconnect_ap_test
wifi_connect_chinese_ap_test      wifi_remove_all_networks_test
wifi_disconnect_ap_test           wifi_remove_network_test
wifi_get_connection_info_test     wifi_scan_results_test
wifi_get_netid_test              wifi_wps_pbc_test
wifi_list_networks_test          wifi_longtime_scan_test
wifi_off_test                    wifi_longtime_test
wifi_on_off_test
```

#### Other Utilities
- `ipcalc.sh` - IP calculator
- `opkg` - Package manager
- `passwd` - Password management
- `setusbconfig` - USB configuration
- `softap_*` - Software AP tools
- `ubus` - System/process communication
- `vi` - Text editor


### Here's an AI summary of what these commands do:

===== AI START =====
#### Standard Unix Commands

**File and Directory Management:**
- `cp`, `mv`, `rm`, `ln`, `touch`, `mkdir`, `rmdir`
  - Used for copying, moving, removing files, creating links, and managing directories.

**Text Processing:**
- `cat`, `echo`, `grep`, `sed`, `awk`
  - Used for displaying, searching, and processing text.

**System Information:**
- `uname`, `df`, `du`, `free`, `uptime`
  - Provide information about the system, disk usage, memory usage, and system uptime.

**Process Management:**
- `ps`, `kill`, `top`
  - Used for listing processes, terminating processes, and monitoring system performance.

**Archiving and Compression:**
- `tar`, `gzip`, `gunzip`, `zcat`
  - Used for creating and extracting archives, and compressing and decompressing files.

#### Networking Tools

**Network Configuration and Testing:**
- `ifconfig`, `iwconfig`, `ping`, `traceroute`, `traceroute6`
  - Used for configuring network interfaces, testing network connectivity, and tracing network routes.

**Package Management:**
- `opkg`
  - The package manager used in OpenWRT for installing, updating, and removing software packages.

#### Wi-Fi Management Scripts

**Wi-Fi Connection and Management:**
- `wifi_connect_ap_test`, `wifi_connect_ap_with_netid_test`, `wifi_connect_chinese_ap_test`, `wifi_disconnect_ap_test`
  - Used for connecting to and disconnecting from Wi-Fi access points.

**Wi-Fi Information and Scanning:**
- `wifi_get_connection_info_test`, `wifi_get_netid_test`, `wifi_list_networks_test`, `wifi_scan_results_test`
  - Provide information about the current Wi-Fi connection, list available networks, and scan for Wi-Fi networks.

**Wi-Fi Control:**
- `wifi_on_test`, `wifi_off_test`, `wifi_on_off_test`, `wifi_reconnect_ap_test`, `wifi_remove_all_networks_test`, `wifi_remove_network_test`
  - Used for turning Wi-Fi on and off, reconnecting to access points, and managing saved networks.

#### System Management

**Synchronization and Sleep:**
- `sync`, `sleep`, `usleep`
  - Used for synchronizing cached writes to persistent storage and pausing execution for a specified duration.

**System Utilities:**
- `ubus`, `uclient-fetch`, `umount`, `mount`
  - Used for inter-process communication, fetching files from the network, and mounting and unmounting filesystems.

===== AI END =====

Looks like it uses `opkg` package manager but `opkg update` returns 
```
root@TinaLinux:/bin# opkg update
Downloading /base/Packages.gz.
wget: bad address ''
Downloading /kernel/Packages.gz.
wget: bad address ''
Collected errors:
 * opkg_download: Failed to download /base/Packages.gz, wget returned 1.
 * opkg_download: Failed to download /kernel/Packages.gz, wget returned 1.

``` 
even after enabling the wifi (pings working)...

Whats strang is that TrimUI itself is installed as an opkg (see `usr/lib/opkg/trimUI.list`) that shows all OS related files.
## Opkg
`opkg list installed` gives: 

```

MtpDaemon - 1-1
adb - 0.0.1-1
alsa-conf-aw - 1.0.0-1
alsa-lib - 1.1.4.1-1
alsa-utils - 1.1.0-1
app_usb_storage - 1.0.0-1
avahi-dbus-daemon - 0.6.31-12
aw869b-rftest - 1.0.0-1
base-files - 167-1730823064
benchmarks - 1
bgcmdhandler - 1.0-1
block-mount - 2016-01-10-96415afecef35766332067f4205ef3b2c7561d21
bluez-alsa - 20180913-1
bluez-daemon - 5.54-1
bluez-libs - 5.54-1
bluez-utils - 5.54-1
bluez-utils-extra - 5.54-1
boost - 1.64.0-1
boost-atomic - 1.64.0-1
boost-chrono - 1.64.0-1
boost-container - 1.64.0-1
boost-context - 1.64.0-1
boost-coroutine - 1.64.0-1
boost-date_time - 1.64.0-1
boost-fiber - 1.64.0-1
boost-filesystem - 1.64.0-1
boost-graph - 1.64.0-1
boost-iostreams - 1.64.0-1
boost-libs - 1.64.0-1
boost-log - 1.64.0-1
boost-math - 1.64.0-1
boost-program_options - 1.64.0-1
boost-python - 1.64.0-1
boost-random - 1.64.0-1
boost-regex - 1.64.0-1
boost-serialization - 1.64.0-1
boost-signals - 1.64.0-1
boost-system - 1.64.0-1
boost-test - 1.64.0-1
boost-thread - 1.64.0-1
boost-timer - 1.64.0-1
boost-wave - 1.64.0-1
brcm_patchram_plus - 0.0.1-1
btmanager-core - 0.0.0-1
btmanager-demo - 0.0.0-1
busybox - 1.27.2-3
common - 0.0.1-1
cpu_monitor - 1-1
curl - 7.54.1-1
dbus - 1.10.4-1
dbus-utils - 1.10.4-1
decodertest - 2.0-1
dnsmasq - 2.78-1
e2fsprogs - 1.42.12-1
eudev - 3.2.9-1
fbscreencap - 1.0.0-1
fbtest - 1
fdk-aac - 0.1.6-2
fneditor - 1.0-1
fontconfig - 2.12.1-3
fstools - 2016-01-10-96415afecef35766332067f4205ef3b2c7561d21
g2d_demo - 1
getevent - 1
glib2 - 2.50.1-1
gptfdisk - 1.0.10-1010
hardwareservice - 1
hostapd - 2017-11-08-2
iozone3 - 489-1
iperf - 2.0.10-1
iptables - 1.4.21-2
iw - 4.3-1
jshn - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
jsonfilter - 2014-06-19-cdc760c58077f44fc40adbbe41e1556a67c1b9a9
kernel - 4.9.191-1-c18835c982db6de7f484e357e2754835
keymon - 1
kmod-cfg80211 - 4.9.191-1
kmod-crypto-aead - 4.9.191-1
kmod-crypto-cmac - 4.9.191-1
kmod-crypto-ecb - 4.9.191-1
kmod-crypto-hash - 4.9.191-1
kmod-crypto-manager - 4.9.191-1
kmod-crypto-pcompress - 4.9.191-1
kmod-fuse - 4.9.191-1
kmod-ge8300-km - 4.9.191-2
kmod-hid - 4.9.191-1
kmod-input-core - 4.9.191-1
kmod-input-evdev - 4.9.191-1
kmod-ipt-core - 4.9.191-1
kmod-lib-crc16 - 4.9.191-1
kmod-net-xr829 - 4.9.191-1
kmod-nls-base - 4.9.191-1
kmod-sunxi-vin - 4.9.191-1
kmod-touchscreen-gt9xxnew_ts - 4.9.191-1
liballwinner-base - 1-1
libarelink - 0.1-1
libatomic - linaro-7.4-2019.02-1
libattr - 20150922-1
libavahi-client - 0.6.31-12
libavahi-compat-libdnssd - 0.6.31-12
libavahi-dbus-support - 0.6.31-12
libavahi-nodbus-support - 0.6.31-12
libblkid - 2.25.2-4
libblobmsg-json - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
libbz2 - 1.0.6-2
libc - 2.29-1
libcap - 2.24-1
libcedarx - 2.8-1
libconfig - 1.4.9-1
libcurl - 7.54.1-1
libcutils - 1-1
libdaemon - 0.14-5
libdbus - 1.10.4-1
libevdev - 1.4.6-1
libexpat - 2.1.0-3
libext2fs - 1.42.12-1
libffi - 3.3-1
libflac - 1.3.1-3
libfreetype - 2.6.1-2
libgamename - 0.1-1
libgcc - linaro-7.4-2019.02-1
libgcrypt - 1.7.0-1
libgmp - 6.1.0-1
libgnutls - 3.6.5-1
libgomp - 2.29-1
libgpg-error - 1.27-1
libgpu - 1
libical - 1.0-1
libid3tag - 0.15.1b-4
libinput - 1.5.0-1
libip4tc - 1.4.21-2
libip6tc - 1.4.21-2
libjpeg - 9a-1
libjson-c - 0.13.1-1
libjson-script - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
liblzma - 5.2.2-1
libmad - 0.15.1b-3
libncurses - 5.9-3
libncursesw - 5.9-3
libnettle - 3.4.1-2
libnghttp2 - 1.24.0
libnl-tiny - 0.1-5
libnss - 3.26-2
libogg - 1.3.2-2
liboil - 0.3.17-2
libopenssl - 1.1.0i-1
libopus - 1.1.3-2
libpcre - 8.38-2
libpixman - 0.34.0-1
libpng - 1.2.56-1
libpopt - 1.16-1
libpthread - 2.29-1
libreadline - 6.3-1
librt - 2.29-1
libscenemanager - 1-1
libsdl - 1.2.15-15
libsdl-image - 1.2.12-12
libsdl-mixer - 1.2.12-12
libsdl-ttf - 2.0.11-11
libsdl2 - 2.30.8-29
libsdl2-image - 2.0.1-20
libsdl2-mixer - 2.0.1-21
libsdl2-ttf - 2.0.13-13
libsdl_test - 1.2.15-15
libsec_key - 1-2
libsec_key-demo - 1-2
libshmvar - 0.1-1
libsndfile - 1.0.28-1
libsocket_db - 0.1-1
libsoup - 2.54.1-2
libsqlite3 - 3120200-1
libssp - linaro-7.4-2019.02-1
libstdcpp - linaro-7.4-2019.02-1
libtheora - 1.1.1-1
libthread-db - 2.29-1
libtmenu - 0.1-1
libuapi - 1-1
libubox - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
libubus - 2016-01-26-619f3a160de4f417226b69039538882787b3811c
libuci - 2016-02-02.1-1
libuclient - 2016-01-28-2e0918c7e0612449024caaaa8d44fb2d7a33f5f3
libump - ec0680628744f30b8fac35e41a7bd8e23e59c39f-1
libuuid - 2.25.2-4
libv4l - 1.20.0-1
libvorbis - 1.3.5-1
libvpx - 1.6.0-1
libxml2 - 2.9.3-1
libxtables - 1.4.21-2
libzstd - 1.4.4-2
live - 2019.02.27
logd - 2016-03-07-fd4bb41ee7ab136d25609c2a917beea5d52b723b
memtester - 4.3.0-1
moonlight - 2.6.1
moonlightui - 1.0-1
mtd - 21
mtdev - 1.1.5-1
netifd - 2016-02-01-3610a24b218974bdf2d2f709a8af9e4a990c47bd
nspr - 4.12-1
ntfs-3g - 2015.3.14-1-fuseint
ntfsprogs_ntfs-3g - 2015.3.14-1-fuseint
opengles_demo - 0.0.1-1
opkg - 9c97d5ecd795709c8584e972bfdf3aee3a5b846d-10
procd - 2016-02-08-57fe34225206c572b5e049826505f738191e4db5
resize2fs - 1.42.12-1
rwcheck - 1-1
sbc - 1.3-2
sdformater - 1.0.0-1
sdl2play - 1.0.0-1
sdl2teartest - 1.0.0-1
sdldisplay - 1.0.0-1
sdlteartest - 1.0.0-1
sdltest - 1.0.0-1
smartlinkd-demo - 0.2.1-1
smartlinkd-lib - 0.2.1-1
softap - 0.0.1-1
softap-demo - 0.0.1-1
stress - 1.0.4-1
systemval - 1.0-1
terminfo - 5.9-3
tinyalsa-lib - 1.1.1-34ffa583936aeb6938636c9c0a26a322b69b0d26
tinyalsa-utils - 1.1.1-34ffa583936aeb6938636c9c0a26a322b69b0d26
tplayerdemo - 1-1
tplayerslave - 1-1
trimui_btmanager - 1.0.0-1
trimui_inputd - 1.0.0-1
trimui_player - 1.0-1
trimui_scened - 1.0.0-1
trimusUI - 1.0-1 <---- weird, maybe I should be instally my stuff as an opkg?>
tslib - 1.15-2
tune2fs - 1.42.12-1
uboot-envtools - 2018.03-2
ubox - 2016-03-07-fd4bb41ee7ab136d25609c2a917beea5d52b723b
ubus - 2016-01-26-619f3a160de4f417226b69039538882787b3811c
ubusd - 2016-01-26-619f3a160de4f417226b69039538882787b3811c
uci - 2016-02-02.1-1
uclibcxx - 0.2.4-3
uclient-fetch - 2016-01-28-2e0918c7e0612449024caaaa8d44fb2d7a33f5f3
udpbcast - 1.0-1
usign - 2015-05-08-cf8dcdb8a4e874c77f3e9a8e9b643e8c17b19131
wget - 1.20.1-3
wifimanager - 0.0.1-1
wifimanager-demo - 0.0.1-1
wireless-tools - 29-5
wpa-cli - 2017-11-08-2
wpa-supplicant - 2017-11-08-2
x264 - 1
xr829-firmware - 1
xr829-rftest - 1.0.0-1
xradio - 0.0.1-1
zlib - 1.2.8-1
zoneinfo-africa - 2018i-1
zoneinfo-asia - 2018i-1
zoneinfo-atlantic - 2018i-1
zoneinfo-australia-nz - 2018i-1
zoneinfo-core - 2018i-1
zoneinfo-europe - 2018i-1
zoneinfo-india - 2018i-1
zoneinfo-northamerica - 2018i-1
zoneinfo-pacific - 2018i-1
zoneinfo-poles - 2018i-1
zoneinfo-simple - 2018i-1
zoneinfo-southamerica - 2018i-1
```

### TrimUI Shell Scripts

*Found in `usr/trimui/bin`*


### Universally Available Compiled C Programs

The following compiled C programs are available in the `usr/trimui/bin` directory. Their usage can be found in other scripts in the same directory along with MinUI files:

- `MainUI`
- `bgcmdhandler`
- `cgdisk`
- `fbscreencap`
- `fixparts`
- `gdisk`
- `hardwareservice`
- `keymon`
- `sdl2play`
- `sdldisplay/sdldisplay`
- `sdlteartest`
- `sdltest`
- `sgdisk`
- `shmvar`
- `systemval`
- `testbitmap`
- `testsprite`
- `trimui_btmanager`
- `trimui_inputd`
- `trimui_scened`
- `udpbcast`



### /usr/trimui/bin/init_lang.sh
* This script sets the system language based on the presence of a user guide.
* If a Chinese guide exists, it sets the system to Chinese.

```sh
#!/bin/sh

sleep 2

if [ -d /mnt/SDCARD/Apps/user_guide_cn ]; then
  echo "set language chinese"

  if [ -e /mnt/UDISK/system.json ]; then
     if [ -e /mnt/UDISK/.inited.lang ]; then
         echo "skip copy"
     else
         touch /mnt/UDISK/.inited.lang
         echo "copy system.json from factory reset"
         cp /usr/trimui/bin/system.json.ch.lang /mnt/UDISK/system.json
         killall -9 MainUI
     fi
  else
     touch /mnt/UDISK/.inited.lang
     echo "copy system.json"
     cp /usr/trimui/bin/system.json.ch.lang /mnt/UDISK/system.json
     killall -9 MainUI
  fi
else
  echo "skip language set"
fi
```
----------------------------------------
### /usr/trimui/bin/init_leds.sh (my version)
* This script configures LED settings based on a configuration file.
* It reads settings such as brightness, color, duration, and effect for each LED and applies them.
```sh
#!/bin/sh

export INSTALL_DIR="/etc/led_controller"
export SETTINGS_FILE="$INSTALL_DIR/settings.ini"
export SYS_FILE_PATH="/sys/class/led_anim"
export SERVICE_PATH="/etc/led_controller"
export BASE_LED_PATH="/sys/class/led_anim"
export SCRIPT_NAME=$(basename "$0")
export LOG_FILE="$INSTALL_DIR/settings_daemon.log"

# Function to apply settings for a specific LED
apply_led_settings() {
    local led=$1
    local brightness=$2
    local color=$3
    local duration=$4
    local effect=$5

    echo "[$SCRIPT_NAME]: Writing $led LED information to configuration files ..." | tee -a "$LOG_FILE"
    if [ "$led" = "f1f2" ]; then
        echo $brightness > "$SYS_FILE_PATH/max_scale_$led"
        echo $color > "$SYS_FILE_PATH/effect_rgb_hex_f1"
        echo $color > "$SYS_FILE_PATH/effect_rgb_hex_f2"
        echo $duration > "$SYS_FILE_PATH/effect_duration_f1"
        echo $duration > "$SYS_FILE_PATH/effect_duration_f2"
        echo $effect > "$SYS_FILE_PATH/effect_f1"
        echo $effect > "$SYS_FILE_PATH/effect_f2"
    else
        echo $brightness > "$SYS_FILE_PATH/max_scale_$led"
        echo $color > "$SYS_FILE_PATH/effect_rgb_hex_$led"
        echo $duration > "$SYS_FILE_PATH/effect_duration_$led"
        echo $effect > "$SYS_FILE_PATH/effect_$led"
    fi
}

# == Start of the script ==
echo "[$SCRIPT_NAME]: LED settings daemon started ..." | tee -a "$LOG_FILE"

# Enable write permissions on LED files
chmod a+w $BASE_LED_PATH/* | tee -a "$LOG_FILE"

echo "[$SCRIPT_NAME]: Writing LED information to configuration files ..." | tee -a "$LOG_FILE"
# Read settings from the settings file and apply 
while IFS= read -r line; do
    if echo "$line" | grep -qE '^\[([a-zA-Z0-9]+)\]$'; then
        led=$(echo "$line" | sed -n 's/^\[\([a-zA-Z0-9]\+\)\]$/\1/p')
    elif echo "$line" | grep -qE '^brightness=([0-9]+)$'; then
        brightness=$(echo "$line" | sed -n 's/^brightness=\([0-9]\+\)$/\1/p')
    elif echo "$line" | grep -qE '^color=0x([0-9A-Fa-f]+)$'; then
        color=$(echo "$line" | sed -n 's/^color=0x\([0-9A-Fa-f]\+\)$/\1/p')
    elif echo "$line" | grep -qE '^duration=([0-9]+)$'; then
        duration=$(echo "$line" | sed -n 's/^duration=\([0-9]\+\)$/\1/p')
    elif echo "$line" | grep -qE '^effect=([0-9]+)$'; then
        effect=$(echo "$line" | sed -n 's/^effect=\([0-9]\+\)$/\1/p')
        apply_led_settings $led $brightness $color $duration $effect
    fi
done < "$SETTINGS_FILE"

# Disable write permissions on LED files
chmod a-w $BASE_LED_PATH/* | tee -a "$LOG_FILE"

echo "[$SCRIPT_NAME]: Finished writing LED information to configuration files ..." | tee -a "$LOG_FILE"
echo "[$SCRIPT_NAME]: LED settings daemon stopped ..." | tee -a "$LOG_FILE"
echo "[$SCRIPT_NAME]: exiting ..." | tee -a "$LOG_FILE"
```
----------------------------------------
### /usr/trimui/bin/init_leds_original.sh (stock OS version)
* This script sets the LED frame color to white and writes it to the LED animation system.
* Parameters:
  - $1: Value to set for the LED (not used in the script)
```sh
#exit 0
value=$1
#valstr=`printf "%02X%02X%02X" $value $value $value`
r=255
g=255
b=255
valstr=`printf "%02X%02X%02X" $r $g $b`
echo "$valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr "\
      "$valstr $valstr $valstr $valstr $valstr $valstr $valstr" > /sys/class/led_anim/frame_hex

#echo 5 > /sys/class/leds/sunxi_led0r/brightness
#usleep 10000
#echo 5 > /sys/class/leds/sunxi_led0g/brightness
#usleep 10000
#echo 5 > /sys/class/leds/sunxi_led0b/brightness
#usleep 10000

#echo $value > /sys/class/led_anim/max_scale
```
----------------------------------------
### /usr/trimui/bin/kill_apps.sh
* This script kills a process and all its child processes.
* Parameters:
  - $1: PID of the process to kill
```sh
#!/bin/sh

#pid=`ps | grep cmd_to_run | grep -v grep | sed 's/[ ]\+/ /g' | cut -d' ' -f1`
pid=$1

ppid=$pid
echo pid is $pid
while [ "" != "$pid" ]; do
   ppid=$pid
   pid=`pgrep -P $ppid`
done

if [ "" != "$ppid" ]; then
   kill -9 $ppid
fi
echo ppid $ppid quit
```
----------------------------------------
### /usr/trimui/bin/list_script.sh (my script used to compile this list)
* This script lists the contents of all .sh files in a specified directory.
* Parameters:
  - $1: Directory to list files from (default is current directory)
```sh
#!/bin/sh
#Directory to list files from
DIR=${1:-.}

# List all .sh files in the directory
for file in "$DIR"/*.sh; do
    if [ -f "$file" ]; then
        echo "Contents of $file:"
        cat "$file"
        echo "----------------------------------------"
    fi
done
```
----------------------------------------

### /usr/trimui/bin/low_battery_led.sh (Stock fimware)
* This script sets the LED animation for low battery indication.
* Parameters:
* Particularly it sets the back LEDs to blink red.
  - $1: 1 to enter low battery mode, 0 to exit low battery mode
```sh
case "$1" in
1 )
        echo "enter low battery"
        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_lr
        echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        echo "2000" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        echo "6" >  /sys/class/led_anim/effect_lr
        ;;
0 )
        echo "exit low battery"
        echo "0" >  /sys/class/led_anim/effect_lr
        # echo "00FFFF " >  /sys/class/led_anim/effect_rgb_hex_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        # echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        # echo "500" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        ;;
*)
        ;;
esac
```


### /usr/trimui/bin/low_battery_led.sh (Firmware  1.0.6 )
* This script sets the LED animation for low battery indication.
* Particularly it sets the front and back LEDs to blink red.
* Parameters:
  - $1: 1 to enter low battery mode, 0 to exit low battery mode
```sh
case "$1" in
1 )
        echo "enter low battery"
        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_lr
        echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        echo "2000" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
         echo "6" >  /sys/class/led_anim/effect_lr

        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_f1
        echo "30000" >  /sys/class/led_anim/effect_cycles_f1
        echo "2000" >  /sys/class/led_anim/effect_duration_f1
        echo "6" >  /sys/class/led_anim/effect_f1

        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_f2
        echo "30000" >  /sys/class/led_anim/effect_cycles_f2
        echo "2000" >  /sys/class/led_anim/effect_duration_f2
        echo "6" >  /sys/class/led_anim/effect_f2
        ;;
0 )
        echo "exit low battery"
        echo "0" >  /sys/class/led_anim/effect_lr
        echo "0" >  /sys/class/led_anim/effect_f1
        echo "0" >  /sys/class/led_anim/effect_f2
        # echo "00FFFF " >  /sys/class/led_anim/effect_rgb_hex_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        # echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        # echo "500" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        ;;
*)
        ;;
esac

```
----------------------------------------
### /usr/trimui/bin/preload.sh
* This script sets the CPU frequency scaling governor and min/max frequencies.

```sh
#!/bin/sh

echo ondemand > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo 408000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
echo 2000000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```
----------------------------------------
### /usr/trimui/bin/premainui.sh
* This script removes temporary input configuration files.
```sh
#!/bin/sh

rm -f /tmp/trimui_inputd/input_no_dpad
rm -f /tmp/trimui_inputd/input_dpad_to_joystick
```
----------------------------------------
### /usr/trimui/bin/runtrimui-original.sh (pre MinUI startup script)
* This script initializes various system settings and runs the main UI application.
* It also handles updating the system if an update file is present.
```sh
#!/bin/sh

#echo 0 > /sys/class/graphics/fbcon/cursor_blink
#echo 0 > /sys/class/vtconsole/vtcon1/bind

export PATH=/usr/trimui/bin:$PATH
chmod a+x /usr/bin/notify
UPDATE_FILE_NAME=/tmp/.try_upgrade_file
UPDATE_DEST=/mnt/UDISK/trimui.img
UPDATE_LOG=/mnt/SDCARD/update.log
UDISK_TRIMUI_DIR=/mnt/UDISK/trimui

SDCARD_TRIMUI_DIR=/mnt/SDCARD/trimui

SDCARD_START_SCRIPTS_DIR=/mnt/SDCARD/System/starts

INPUTD_SETTING_DIR_NAME=/tmp/trimui_inputd

#PD11 pull high for VCC-5v
echo 107 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio107/direction
echo -n 1 > /sys/class/gpio/gpio107/value

#rumble motor PH3
echo 227 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio227/direction
echo -n 0 > /sys/class/gpio/gpio227/value

##//TRIMUI Smart Pro IO board power
#Left/Right Pad PD14/PD18
# echo 110 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio110/direction
# echo -n 1 > /sys/class/gpio/gpio110/value

# echo 114 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio114/direction
# echo -n 1 > /sys/class/gpio/gpio114/value

#DIP Switch PH19
echo 243 > /sys/class/gpio/export
echo -n in > /sys/class/gpio/gpio243/direction

killprocess(){
  #pid=`ps | grep $1 | grep -v grep | cut -d' ' -f3`
  pid=`pgrep $1`
  while [ "$pid" != "" ] ; do
    echo kill -9 $pid
    kill -9 $pid
    sleep 1
    pid=`pgrep $1`
  done
}

runifnecessary(){
   #a=`ps | grep $1 | grep -v grep`
   a=`pgrep $1`
   if [ "$a" == "" ] ; then
      $2 &
   fi
}

#wait for SDCARD mounted
mounted=`cat /proc/mounts | grep SDCARD`
cnt=0
while [ "$mounted" == "" ] && [ $cnt -lt 6 ] ; do
  sleep 0.5
  cnt=`expr $cnt + 1`
  mounted=`cat /proc/mounts | grep SDCARD`
done

if [ -d $SDCARD_START_SCRIPTS_DIR ] ; then
  for f in `ls $SDCARD_START_SCRIPTS_DIR/*.sh` ; do
        chmod a+x $f
        $f
  done
fi

mkdir $INPUTD_SETTING_DIR_NAME

echo 1 > /sys/class/led_anim/effect_enable
echo "FFFFFF" > /sys/class/led_anim/effect_rgb_hex_lr
echo 1 > /sys/class/led_anim/effect_cycles_lr
echo 1000 > /sys/class/led_anim/effect_duration_lr
echo 1 >  /sys/class/led_anim/effect_lr

syslogd -S

/etc/bluetooth/bluetoothd start
#/usr/bin/bluealsa -p a2dp-source&
#touch /tmp/bluetooth_ready

usbmode=`/usr/trimui/bin/systemval usbmode`

if [ "$usbmode" == "dock" ] ; then
   /usr/trimui/bin/usb_dock.sh
elif [ "$usbmode" == "host" ] ; then
   /usr/trimui/bin/usb_host.sh
else
   /usr/trimui/bin/usb_device.sh
fi

wifion=`/usr/trimui/bin/systemval wifi`
if [ "$wifion" != "1" ] ; then
      ifconfig wlan0 down
      killall -15 wpa_supplicant
      killall -9 udhcpc
fi

while [ 1 ]; do
  if [ -f $UPDATE_FILE_NAME ] ; then
    upfile=`cat $UPDATE_FILE_NAME`
    if [ -f ${upfile} ] ; then
        killprocess keymon
        cd /
        umount  "$UDISK_TRIMUI_DIR"
        sleep 1
        mounted=`mount | grep "$UDISK_TRIMUI_DIR"`
        echo mounted is $mounted

        cat /proc/mounts
        echo start updating $upfile $? | tee $UPDATE_LOG
        updateui >> $UPDATE_LOG &
        sleep 1
        notify 0 extracting package
        cp $upfile $UPDATE_DEST
        notify 100 quit
        sleep 1
        killprocess updateui
       else
          echo $upfile not found | tee $UPDATE_LOG
       fi
    rm -f $UPDATE_FILE_NAME
    rm -f /tmp/state.json
  else
    #exit 0
    udiskOK=0
    uiRunned=0
    ls /dev/mmcblk1* > /tmp/imgrun.log
    cat /proc/mounts >> /tmp/imgrun.log
    if [ -d $SDCARD_TRIMUI_DIR ] ; then
       echo "run from sdcard" >> /tmp/imgrun.log
       export LD_LIBRARY_PATH=${SDCARD_TRIMUI_DIR}/lib
       runifnecessary "keymon" ${SDCARD_TRIMUI_DIR}/app/keymon
       runifnecessary "inputd" ${SDCARD_TRIMUI_DIR}/app/trimui_inputd
       runifnecessary "scened" ${SDCARD_TRIMUI_DIR}/app/trimui_scened
       runifnecessary "trimui_btmanager" ${SDCARD_TRIMUI_DIR}/app/trimui_btmanager
       runifnecessary "hardwareservice" ${SDCARD_TRIMUI_DIR}/app/hardwareservice
       cd ${SDCARD_TRIMUI_DIR}/app/

       #absolute path for ps
       ${SDCARD_TRIMUI_DIR}/app/premainui.sh
       ${SDCARD_TRIMUI_DIR}/app/MainUI
       ${SDCARD_TRIMUI_DIR}/app/preload.sh

       if [ $? -eq 0 ] || [ $? -eq 137 ] || [ $? -eq 143 ] ; then
          uiRunned=1
       else
          uiRunned=0
       fi
    else
       echo "no $SDCARD_TRIMUI_DIR" >> /tmp/imgrun.log
    fi

    if [ $uiRunned -eq 0 ] ; then
       if [ -f $UPDATE_DEST ] ; then
          mkdir -p $UDISK_TRIMUI_DIR
          mounted=`mount | grep $UDISK_TRIMUI_DIR`
          if [ "$mounted" == "" ] ; then
            mount -o loop $UPDATE_DEST $UDISK_TRIMUI_DIR
            if [ $? -eq 0 ] ; then
               echo "new img mounted" >> /tmp/imgrun.log
               udiskOK=1
             fi
          else
            echo "img mounted" >> /tmp/imgrun.log
               udiskOK=1
          fi
       else
          mkdir -p $UDISK_TRIMUI_DIR
          echo "$UPDATE_DEST exists" >> /tmp/imgrun.log
       fi
    fi

   tinymix set 9 1
   tinymix set 1 0
#    tinymix set 7 175 175

    if [ $udiskOK -eq 1 ] && [ -d $UDISK_TRIMUI_DIR ] ; then
       echo "run from img" >> /tmp/imgrun.log
       export LD_LIBRARY_PATH=${UDISK_TRIMUI_DIR}/lib
       runifnecessary "keymon" ${UDISK_TRIMUI_DIR}/bin/keymon
       runifnecessary "inputd" ${UDISK_TRIMUI_DIR}/bin/trimui_inputd
       runifnecessary "scened" ${UDISK_TRIMUI_DIR}/bin/trimui_scened
       runifnecessary "trimui_btmanager" ${UDISK_TRIMUI_DIR}/bin/trimui_btmanager
       runifnecessary "hardwareservice" ${UDISK_TRIMUI_DIR}/bin/hardwareservice
       cd ${UDISK_TRIMUI_DIR}/bin/

       #absolute path for ps
       ${UDISK_TRIMUI_DIR}/app/premainui.sh
       ${UDISK_TRIMUI_DIR}/bin/MainUI
       ${UDISK_TRIMUI_DIR}/bin/preload.sh
       if [ $? -eq 0 ] || [ $? -eq 137 ] || [ $? -eq 143 ] ; then
          uiRunned=1
       else
          uiRunned=0
       fi
    fi

    if [ $uiRunned -eq 0 ] ; then
       export LD_LIBRARY_PATH=/usr/trimui/lib
          cd /usr/trimui/bin
       runifnecessary "keymon" keymon
       runifnecessary "inputd" trimui_inputd
       runifnecessary "scened" trimui_scened
       runifnecessary "trimui_btmanager" trimui_btmanager
       runifnecessary "hardwareservice" hardwareservice
       premainui.sh
       MainUI
       preload.sh
    fi

    if [ -f /tmp/trimui_inputd_restart ] ; then
      #restart before emulator run
      killall -9 trimui_inputd
      sleep 0.2
      runifnecessary "inputd" trimui_inputd
      rm /tmp/trimui_inputd_restart
    fi

    if [ -f /tmp/.cmdenc ] ; then
       /root/gameloader
    elif [ -f /tmp/cmd_to_run.sh ] ; then
      chmod a+x /tmp/cmd_to_run.sh
      udpbcast -f /tmp/host_msg &
         /tmp/cmd_to_run.sh
      rm /tmp/cmd_to_run.sh
         rm /tmp/host_msg
         killall -9 udpbcast
    fi
  fi
done
```
----------------------------------------
### /usr/trimui/bin/runtrimui.sh
* This script waits for the SDCARD to be mounted and then runs the updater if present,
* otherwise it runs the original runtrimui script.

```sh
#!/bin/sh

# becomes /usr/trimui/bin/runtrimui.sh on tg5040/tg3040

#wait for SDCARD mounted
mounted=`cat /proc/mounts | grep SDCARD`
cnt=0
while [ "$mounted" == "" ] && [ $cnt -lt 6 ] ; do
  sleep 0.5
  cnt=`expr $cnt + 1`
  mounted=`cat /proc/mounts | grep SDCARD`
done

UPDATER_PATH=/mnt/SDCARD/.tmp_update/updater
if [ -f "$UPDATER_PATH" ]; then
      "$UPDATER_PATH"
else
      /usr/trimui/bin/runtrimui-original.sh
fi
```
----------------------------------------

### /usr/trimui/bin/scene.sh
* This script runs all scene scripts in the /usr/trimui/scene/ directory with a given parameter.
* Parameters:
  - $1: Parameter to pass to each scene script
```sh
#!/bin/sh

echo "run scene $1"
cd /usr/trimui/scene/

for scene in `ls *.sh`
do
   killall $scene
   echo run script $scene
   ./$scene $1 &
done
```
----------------------------------------

### /usr/trimui/bin/usb_device.sh
* This script configures the system to operate in USB device mode.

```sh
#PH07 smartpro bottom
# echo 231 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio231/direction
# echo 0 > /sys/class/gpio/gpio231/value

#PL08 back USB host
echo 360 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio360/direction
echo 1 > /sys/class/gpio/gpio360/value

# will disable inside OHCI0/EHCI0
# sleep 0.1
# echo 0 > /sys/devices/platform/soc/usbc0/axp_5v
cat /sys/devices/platform/soc/usbc0/usb_device
```
----------------------------------------
### /usr/trimui/bin/usb_dock.sh
* This script configures the system to operate in USB dock mode.

```sh
cat /sys/devices/platform/soc/usbc0/usb_host
sleep 0.2
echo 0 > /sys/devices/platform/soc/usbc0/axp_5v

#PH07  smartpro bottom
# echo 231 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio231/direction
# echo 0 > /sys/class/gpio/gpio231/value

#PL08 back USB host
echo 360 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio360/direction
echo 1 > /sys/class/gpio/gpio360/value
```
----------------------------------------

### /usr/trimui/bin/usb_host.sh
* This script configures the system to operate in USB host mode.
```sh
cat /sys/devices/platform/soc/usbc0/usb_host
# sleep 0.1
echo 1 > /sys/devices/platform/soc/usbc0/axp_5v

#PH07 smartpro bottom
# echo 231 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio231/direction
# echo 1 > /sys/class/gpio/gpio231/value

#PL08 back USB host
echo 360 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio360/direction
echo 1 > /sys/class/gpio/gpio360/value
```

Similarly, `/bin/wifi_connect_ap_test "<wifi_name>" <wifi_pass>` gave this: 

```

==================================
Connecting to the network(<wifi_name>)......
Connected to the AP(<wifi_name>)
Getting ip address(<wifi_name>)......
sh: odhcp6c: not found
Wifi connect ap : Success!
==================================
```

After testing those, pings are going through to normal websites (i.e, we're online baby)


After fiddling around with some wifi stuff: 

`/bin/wifi_list_networks_test` will give you all the saved networks including their passwords, passing in `d0-d5` will give you increading levels of 
```
---------------------------------------------------------------------------------
NAME:
  wifi_list_networks_test
DESCRIPTION:
  list all the networks in Wpa_supplicant.conf

USAGE:
  wifi_list_networks_test <level>
  level: print level(d0~d5).larger value,more info.para is not required,default d2.
--------------------------------------MORE---------------------------------------
The way to get help information:
  wifi_list_networks_test --help
  wifi_list_networks_test -h
  wifi_list_networks_test -H
---------------------------------------------------------------------------------
```

we actually need to manually connect to our saved wifi router with `/bin/wifi_connect_ap_test "<wifi_name>" <wifi_pass>` to get the wifi to connect.


1-17-25
## Wifi info 

So all wifi related actions/configs are located in `etc/wifi`.

Here you'll find `/etc/wifi/wpa_supplicant.conf` which contains wifi configuration params including the plaintext of user stored ssid/password pairs formatted like below 

```sh
ctrl_interface=/etc/wifi/sockets
disable_scan_offload=1
update_config=1
wowlan_triggers=any

network={
      ssid="PT Wifi"
      psk="5625875343"
}
```

 This is used by the [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant) service in `etc/init.d/wpa_supplicant`. It's an off-the-shelf linux tool to that handles WPA authentication. 

Simply starting the service with `/etc/init.d/wpa_supplicant start` or `/etc/init.d/wpa_supplicant reload` (use `stop` to...stop) is enough to connect to a wireless network that's saved in `/etc/wifi/wpa_supplicant.conf`. `wpa_supplicant` can also handle network discovery/login but it requires an interterface to be built for it. In the meantime, we should just setup the network on the stock OS. 

While this will connect to a network, it won't finish DNS resolution or ip routing. I'm not entirely sure how this works just yet but checking the ip resolution config in `/etc/resolv.conf` gave

```nameserver 192.168.0.1` which is just the default local ip address setting AFAIk.
Checking `ip route` showed `192.168.0.0/24 dev wlan0 scope link  src 192.168.0.95`

then running `ip route add default via 192.168.0.1 dev wlan0` enabled pings.

TLDR: 
to start wifi, run `/etc/init.d/wpa_supplicant stop start && ip route add default via 192.168.0.1 dev wlan0`
For further deep diving, check out these MinUI paks from [Jose Gonzolez](https://github.com/josegonzalez): [wifi toggle](https://github.com/josegonzalez/trimui-brick-toggle-wifi-pak) [File Browser](https://github.com/josegonzalez/trimui-brick-filebrowser-pak), [Terminal](https://github.com/josegonzalez/trimui-brick-terminal-pak), [Remote Terminal](https://github.com/josegonzalez/trimui-brick-remote-terminal-pak), [SSH Server](https://github.com/josegonzalez/trimui-brick-dropbear-server-pak), [HTTP Server](https://github.com/josegonzalez/trimui-brick-dufs-server-pak), [FTP Server](https://github.com/josegonzalez/trimui-brick-sftpgo-server-pak), [Stay Awake Daemon](https://github.com/josegonzalez/minui-developer-pak)


