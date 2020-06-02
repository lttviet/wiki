# Homepass

## StreetPass

3DS has a built-in feature [StreetPass][1] which lets nearby users to exchange data from games and applications.
This is great if you live in a big city where there are a lot of 3DS devices.
However, if you live in a small town, you would be lucky to just get a handful of streetpasses monthly.

## StreetPass Relay

Due to that, Nintendo introduced another feature called [StreetPass Relay][2].
Every 3DS device has a whitelist of public wifi AP which it can automatically connect to.
If connected, some data will be sent and stored on Nintendo server.
When another 3DS user connects to the same AP, Nintendo server will send the stored data to the latter device.
Consequently, 3DS devices don't have to be in close proximity to exchange data.

The whitelist contains the SSID and MAC of public wifi AP.
Thus, some nice people figured out ways to create homemade StreetPass relays by spoofing MAC and SSID[^1].
The basic idea to create a public wifi AP with the same SSID and MAC as in the whitelist.

For Raspberry Pi, there is [SpillPass-Pi2][3] which work automatically when you plug in.
You only need a wifi dongle `wlan0` which support mac spoof and a wired internet connection `eth0`.

My setup is slightly different because I don't have a wired connection to the home router.
What I have is 2 wifi dongles, `wlan0` and `wlan1`.
The idea is still the same, but instead of `eth0` bridge to `wlan0`, I have `wlan1` connected to the internet and `wlan0` as wifi AP.
All the traffic are forwarded to/from `wlan0` to `wlan1`.

## Setup

I used [Raspbian Jessie Lite][4] image.
I don't have external keyboard or monitor so desktop on Raspberry Pi is unnecessary.
Installed the image on the SD card by either dd or [Etcher][5], then added your wifi network to `/etc/wpa_supplicant/wpa_supplicant.conf`.[^2]

```bash
#/etc/wpa_supplicant/wpa_supplicant.conf
network={
    ssid="testing"
    psk="testingPassword"
}
```

Updated and installed softwares:

```bash
sudo apt update && sudo apt upgrade
sudo apt -y install hostapd dnsmasq haveged iptable-persistent sqlite3
```

I used this wonderful [guide][6] to setup forwarding.

### Interfaces

Set `wlan0` to a static ip address `10.0.0.1`.


```bash
#/etc/network/interfaces
auto lo
iface lo inet loopback

iface eth0 inet manual

# AP
allow-hotplug wlan0
iface wlan0 inet static
    address 10.0.0.1
    netmask 255.255.255.0

# wireless
allow-hotplug wlan1
iface wlan1 inet manual
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

Tell DHCPCD to ignore wlan0.

```bash
#/ect/dhcpcd.conf
denyinterfaces wlan0
```

### Hostapd

Hostapd is an user daemon to start up AP.
In `/etc/default/hostapd`, uncomment or add the line `DAEMON_CONF=/etc/hostapd/hostapd.conf`.

```bash
#/etc/hostapd/hostapd.conf
#sample hostapd.conf
#overwrite by homepass.sh

ssid=attwifi
bssid=34:bd:c8:db:79:00

interface=wlan0
driver=nl80211

hw_mode=g
channel=6
auth_algs=3
wpa=0
rsn_pairwise=CCMP
beacon_int=100

macaddr_acl=0

wmm_enabled=0
eap_reauth_period=360000000
```

You can test if AP is working by running `sudo hostapd /etct/hostapd/hostapd`.
You should be able to see `attwifi`.

I use `dnsmasq` to map IP addresses to MAC addresses.
When 3DS connect to my hotspot, it will be assigned a dynamic IP address.
It is also possible to bind 3DS MAC address to a static IP address.

```bash
#/etc/dnsmasq.conf
interface=wlan0
except-interface=wlan1
dhcp-range=10.0.0.2,10.0.0.255,12h
```

### Port forwarding

In `/etc/sysctl.conf`, uncomment `net.ipv4.ip_forward=1`.

```bash
sudo iptables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o wlan1 -j ACCEPT
sudo sh -c 'iptables-save > /etc/iptables/rules.v4'
```

Start services `sudo service hostapd start && sudo service dnsmasq start`.
Reboot Raspberry Pi and test if `attwifi` can be connected to the internet.

### systemd service and timer

I use systemd to run my `homepass.sh` at boot time and the script will rerun every 5 minutes to change `wlan0` to new SSID and MAC.

```bash
#/etc/systemd/system/homepass.service
[Unit]
Description=Homepass

[Service]
ExecStart=/home/pi/homepass.sh

[Install]
WantedBy=multi-user.target
```

```bash
#/etc/systemd/system/homepass.timer
[Unit]
Description=Run homepass.service every 5 minutes

[Timer]
OnStartupSec=10
OnUnitActiveSec=5min
Unit=homepass.service

[Install]
WantedBy=multi-user.target
```

Enable the timer by executing this command

```bash
systemctl enable homepass.timer
systemctl start homepass.timer
```

A nice feature of using systemd to run the script is we can access the logging easily for debugging

```bash
journalctl -u homepass
```

### homepass.sh

This script was modified from orginal script by Semperverus.
You can check out his [guide][7] for more details about HomePass.
I took the idea of using sqlite3 to store MAC and SSID in a database from danielhoherd's [script][8].

The script is rerun every 5 minutes by systemd.
It stops current running hostapd service.
A new `/etc/hostapd/hostapd.conf` is generated from querying MAC and SSID from a database.
The hostapd service is then restarted with the new conf file.

```bash
#!/bin/bash
service hostapd stop

SLEEP_TIME=300
DB=/home/pi/homepass.db
CONFIG_FILE=/etc/hostapd/hostapd.conf

if [[ ! -e "$CONFIG_FILE" ]] ; then
  touch "$CONFIG_FILE"
fi

read -d '' query << EOF
SELECT mac, ssid
FROM aps
WHERE last_used < datetime('now', '-2 days') OR last_used IS NULL
ORDER BY random()
LIMIT 1;
EOF

result=$(sqlite3 "$DB" "$query")
resultArr=(${result//|/ })

MAC=${resultArr[0]}
SSID=${resultArr[1]}

if [[ -z "$MAC" ]] || [[ -z "$SSID" ]] ; then
  echo "missing MAC or SSID"
  exit 1
fi

sqlite3 "$DB" "UPDATE aps SET last_used = datetime('now') WHERE mac = '$MAC'"

cat > $CONFIG_FILE << EOF
ssid=$SSID
bssid=$MAC

interface=wlan0
driver=nl80211

hw_mode=g
channel=6
auth_algs=3
wpa=0
rsn_pairwise=CCMP
beacon_int=100

macaddr_acl=0

wmm_enabled=0
eap_reauth_period=360000000
EOF

echo "==========================================="
echo "SSID:" $SSID "- BSSID:" $MAC
echo "Time before next change:" $SLEEP_TIME "seconds"
echo "Current time:" $(date)
echo "==========================================="

service hostapd start
```

## All files

You can find all my files at my github [repo][9].

[1]: http://www.nintendo.com/3ds/built-in-software/streetpass
[2]: http://en-americas-support.nintendo.com/app/answers/detail/a_id/709/~/streetpass-relay-overview
[3]: http://www.spillmonkey.com/?page_id=169
[4]: https://www.raspberrypi.org/downloads/raspbian/
[5]: https://etcher.io/
[6]: https://help.ubuntu.com/community/WifiDocs/WirelessAccessPoint#Configuring_networking
[7]: https://docs.google.com/document/d/1EvmIwTIjPva5MHSFEIN0qsHtRmdRRiX3WNHu_ThxnOs/edit?pref=2&pli=1#
[8]: https://github.com/danielhoherd/homepass/blob/master/RaspberryPi/homepass.sh
[9]: https://github.com/lttviet/homepass

[^1]: <https://gbatemp.net/threads/how-to-have-a-homemade-streetpass-relay.352645/>

[^2]: <https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md>
