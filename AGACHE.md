# Install the AGACHE server

This guide is designed for a Raspberry Pi 5 with an additional WiFi adapter running Debian 64bit bookworm.

## Use the built in WiFi adapter for the WiFi client connection

```bash
sudo nmcli connection modify preconfigured ifname wlan0
```

Reboot the Raspberry.

## Configure the additional WiFi adapter as Access Point

```bash
sudo nmcli connection add con-name ap ifname wlan1 type wifi ssid "Raspberry Pi AP 01"
sudo nmcli connection modify ap wifi-sec.key-mgmt wpa-psk
sudo nmcli connection modify ap wifi-sec.psk "<PASSWORD>"
sudo nmcli connection modify ap 802-11-wireless.mode ap 802-11-wireless.band bg ipv4.method shared
sudo nmcli connection modify ap wifi-sec.pmf disable
```

Reboot the Raspberry.

## References

Use a Raspberry Pi as a WiFi Hotspot: https://trainedmonkey.com/2024/05/20/using_a_raspberry_pi_as_a_wifi_hotspot
Disable SAE and force the use of WPA2 Personal/SHA-256/AES: https://superuser.com/questions/1771752/m1-macbook-pro-cant-join-linux-hotspot-when-other-devices-can
