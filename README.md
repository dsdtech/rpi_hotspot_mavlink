# RPi Zero W Hotspot + MAVLink Router – Repeatable Setup Guide

A clean, end‑to‑end playbook to turn a **Raspberry Pi Zero W** into a Wi‑Fi hotspot that routes MAVLink between a Pixhawk (UART) and Ground Control Stations (UDP/TCP). This mirrors the workflow you just validated with Mission Planner (UDPCI).

---

## 0) Hardware & Network Topology

* **Pi**: Raspberry Pi Zero W (micro‑USB OTG for power + data)
* **Flight controller**: Pixhawk or ArduPilot‑compatible (USB/TELEM → Pi)
* **Power**: Stable 5V supply (≥2A recommended)
* **Clients**: Laptop/Tablet/Phone (connect to the Pi’s SSID)
* **Network**: Pi acts as AP + DHCP server. Clients receive IPs from the Pi and send MAVLink to the Pi’s UDP port. MAVLink Router fans traffic to all endpoints.

> Tip: On the client (Windows), `ipconfig` → **Default Gateway** reveals the Pi’s hotspot IP (e.g., `10.10.10.10`). Use that as the target in Mission Planner UDPCI.

---

## 1) OS Prep (first boot)

1. Flash Raspberry Pi OS Lite (32‑bit is fine) to SD card.
2. Enable SSH and set Wi‑Fi country (optional) via `raspi-config` after first boot.
3. Update packages:

   ```bash
   sudo apt update && sudo apt -y upgrade
   ```

---

## 2) Get the scripts & tools

```bash
sudo su     # become root because the installer expects root
cd /root
git clone https://github.com/lordcast/rpi_hotspot_mavlink
cd rpi_hotspot_mavlink
chmod +x Rpihotspot_buster
```

Run the installer:

```bash
./Rpihotspot_buster
```

This installs **hostapd**, **dnsmasq**, **mavlink-router**, and drops example configs.

Reboot when prompted:

```bash
reboot
```

---

## 3) Hotspot settings (SSID / password)

Edit **hostapd** configuration to set the broadcast name and key:

```bash
sudo nano /etc/hostapd/hostapd.conf
```

Change:

```
ssid=<YOUR_SSID>
wpa_passphrase=<YOUR_STRONG_PASSWORD>
# country_code=<YOUR_COUNTRY_CODE>
```

Restart services:

```bash
sudo systemctl restart hostapd dnsmasq
```

> Verify: From a phone/laptop, scan and confirm the new SSID appears. Connect and make sure you receive an IP and see a **Default Gateway** (the Pi’s IP).

---

## 4) Find the Pixhawk serial device (one‑time)

Plug the Pixhawk into the Pi (USB) or wire TELEM → UART‑USB adapter.

Quick auto‑detect:

```bash
python3 /root/rpi_hotspot_mavlink/find_port.py
```

Note the detected device path (typically `/dev/serial/by-id/usb-...`).

---

## 5) Configure MAVLink Router

Open the main config:

```bash
sudo nano /etc/mavlink-router/main.conf
```

Minimal working template:

```ini
[General]
TcpServerPort=9003
ReportStats=false
MavlinkDialect=auto

[UdpEndpoint default]
Mode = Eavesdropping
Address = 0.0.0.0
Port = 14550

# optional extra listener
[UdpEndpoint secondary]
Mode = Eavesdropping
Address = 0.0.0.0
Port = 9005

[UartEndpoint bravo]
Device = /dev/serial/by-id/<YOUR-PIXHAWK-ID>
Baud = 57600
```

> Notes
>
> * **Eavesdropping** means the Pi **listens** on these UDP ports. GCS must use **UDPCI** (client) to send to the Pi’s IP:PORT.
> * In practice you validated telemetry on **14551**. That happens when a helper starts an extra listener. If you see 14551 in `netstat`, use that in the GCS.

Restart router (or reboot):

```bash
sudo systemctl restart mavlink-router || sudo pkill mavlink-routerd; sleep 1; mavlink-routerd -c /etc/mavlink-router/main.conf &
```

Check listeners:

```bash
sudo netstat -ulnp | grep -E '1455[0-9]|9003'
```

---

## 6) Make MAVLink Router persistent (systemd)

Create a service so it survives reboots:

```bash
sudo tee /etc/systemd/system/mavlink-router.service >/dev/null <<'EOF'
[Unit]
Description=MAVLink Router
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/mavlink-routerd -c /etc/mavlink-router/main.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable mavlink-router
sudo systemctl start mavlink-router
```

Verify:

```bash
systemctl status mavlink-router --no-pager
sudo netstat -ulnp | grep -E '1455[0-9]|9003'
```

---

## 7) Mission Planner connection (UDPCI)

On the laptop connected to the Pi SSID:

1. **Connection type**: `UDP` → choose **UDPCI** when prompted
2. **Host/IP**: Pi hotspot IP (your **Default Gateway**, e.g., `10.10.10.10`)
3. **Port**: `14551` (use `14550` if that’s the listener you expose)
4. **Baud** field in MP is **ignored** for UDP; UART baud lives in `main.conf`.

You should see heartbeats and parameters begin to load.

---

## 8) Quick verification & diagnostics

* Is the Pi listening?

  ```bash
  sudo netstat -ulnp | grep -i mav
  ```
* Is the Pixhawk talking?

  ```bash
  ls -l /dev/serial/by-id/
  sudo cat /dev/serial/by-id/<YOUR-PIXHAWK-ID>  # should show binary 'gibberish'
  ```
* Live logs:

  ```bash
  sudo journalctl -u mavlink-router -f
  ```
* Confirm your Pi hotspot IP on the client:

  * **Windows**: `ipconfig` → look for **Default Gateway**

---

## 9) Optional: add a second GCS or forward stream

To also stream MAVLink to a tablet on the same Wi‑Fi:

```ini
[UdpEndpoint tablet]
Mode = Normal
Address = <TABLET_IP>
Port = 14550
```

Restart MAVLink Router and connect QGC on the tablet to `0.0.0.0:14550` (it will receive forwarded packets).

---

## 10) Optional hardening & polish

* Change SSID/password regularly and set a country code in `hostapd.conf`.
* Assign a **static hotspot subnet** you prefer and keep a note of the Pi’s IP.
* Enable logging in `main.conf` (`Log`, `LogMode`) to keep tlogs on the Pi.

---

## 11) One‑shot bring‑up checklist

* [ ] Pi boots; hotspot SSID is visible
* [ ] Client connects; **Default Gateway** recorded (Pi IP)
* [ ] `netstat` shows MAVLink listener (14550/14551)
* [ ] Pixhawk detected under `/dev/serial/by-id/`
* [ ] Mission Planner UDPCI → `Pi_IP:Port` shows heartbeat

---

## Appendix – Commands Reference

```bash
# Run hotspot installer again (if needed)
sudo su -
cd /root/rpi_hotspot_mavlink && ./Rpihotspot_buster

# Edit SSID/password
sudo nano /etc/hostapd/hostapd.conf
sudo systemctl restart hostapd dnsmasq

# Edit MAVLink Router
sudo nano /etc/mavlink-router/main.conf
sudo systemctl restart mavlink-router

# Discover serial port
python3 /root/rpi_hotspot_mavlink/find_port.py

# Inspect UDP listeners
sudo netstat -ulnp | grep -E '1455[0-9]|9003'

# Tail router logs
sudo journalctl -u mavlink-router -f
```
