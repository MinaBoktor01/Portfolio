# Portable Probe-Request Tail Detector

A portable, battery-powered Raspberry Pi sensor that detects physical surveillance by passively capturing 802.11 probe requests and flagging devices that reappear across multiple time-separated encounters that was inspired by the ["Chasing Your Tail"](https://github.com/ArgeliusLabs/Chasing-Your-Tail-NG)methodology (Matt Edmondson, Black Hat USA 2022).

---
### What I Learned About Probe Requests
- **Devices broadcast their network history without being asked.** To reconnect faster, phones and laptops send out probe requests that sometimes naming specific remembered SSIDs by name before they've joined anything. You don't need to be on a network to see this you just need a radio in monitor mode within range.
	- This is similar to someone shouting out your name to try to find you, you dont need to me that person to hear the shout. 
- **MAC randomization changes everything.** Modern iOS/Android devices use randomized, rotating MAC addresses for background/idle scanning by default. This also means a single device can appear as several different MACs over a day, which limits how reliable MAC-based correlation can be. It's the single biggest reason this type of tool has real limitations.
- **A single sighting means almost nothing.** Given how much ambient Wi-Fi traffic exists in most places, seeing one nearby device once is not a meaningful signal on its own since we interact with people everyday, which is what pushed the design toward correlation across sessions instead of a proximity threshold.
- **Passive capture is legally different from active interception**, but scope still matters. Not transmitting anything and only listening to broadcast frames is a meaningfully different posture than actively probing or connecting to something but what you retain and do with captured data is a separate question from whether you can technically receive it. That distinction drove most of this project's data minimization choices (see Design Notes below).
---
### Design Notes (Privacy-by-Design)
This tool necessarily captures ambient MACs beyond the operator's own, since detecting a tail requires observing whether a stranger's device persists nearby. What keeps it in the same legitimate category as its inspiration rather than sliding into a tracking tool:
- Alerts require **correlation across sessions**, never a single proximity hit
- Owned devices are **excluded at capture time**, not just filtered in reports
- **No SSID/network-history logging by default**
- Designed to be **carried by the person it protects**, not deployed against a chosen target or left logging a fixed location

---
### Prerequisites
Raspberry Pi 4
Alfa AWUS036ACS USB WiFi adapter (or any monitor-mode-capable adapter)
SD card with Raspberry Pi OS Lite (64-bit)
USB power bank (for portable/carried operation)
Internet connection for initial setup
### System Diagram
```
Probe requests (all nearby devices)
        ↓
wlan1 (monitor mode, USB adapter)
        ↓
sniffer.py  →  session segmentation (time-gap based)
        ↓
sightings.db (SQLite: MAC, RSSI, timestamp, session_id)
        ↓
correlation check  →  alert if MAC seen in 3+ distinct sessions
        ↓
console log + optional ntfy.sh push notification
```

---
### 1. System Setup
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y linux-headers-rpi-v8 build-essential dkms git python3-pip python3-scapy python3-requests
sudo reboot
```
---
### 2. USB WiFi Adapter Driver (Alfa AWUS036ACS)
Driver install can changed based on model if not using the Alfa AWUS036ACS (Realtek 8812AU/8811AU):
```
git clone https://github.com/morrownr/8812au-20210820.git
cd 8812au-20210820
sudo ./dkms-install.sh
sudo reboot
```
---
### 3. Isolate the Adapter from NetworkManager
Prevent NetworkManager from grabbing the monitor-mode interface, so it
doesn't fight your capture setup:
```
sudo tee /etc/NetworkManager/conf.d/unmanage-wlan1.conf << 'EOF'
[keyfile]
unmanaged-devices=interface-name:wlan1
EOF

sudo systemctl restart NetworkManager
```
---
### 4. Enable Monitor Mode
```
sudo rfkill unblock wifi
sudo ip link set wlan1 down
sudo iwconfig wlan1 mode monitor
sudo ip link set wlan1 up
iwconfig wlan1
```
Confirm the output shows `Mode:Monitor` before moving on.

---
### 5. Create the Project Directory
```
mkdir -p ~/tail-detector
cd ~/tail-detector
```
Two files live here: 
`sniffer.py` (capture + session segmentation + live alerting)
`correlate.py` (standalone post-hoc correlation report)

>**Note:** `sightings.db` is created automatically on first run.

Create `sniffer.py`:
```
nano ~/tail-detector/sniffer.py
```

Paste in the following, then save and exit (Ctrl+O, Enter, Ctrl+X):
```python
#!/usr/bin/env python3

import argparse
import datetime
import os
import sqlite3
import sys
import time
import subprocess
import threading

import requests
from scapy.all import Dot11, Dot11Elt, Dot11ProbeReq, sniff

# ==========================================
# CONFIGURATION
# ==========================================
INTERFACE = "wlan1"
DB_PATH = os.path.expanduser("~/tail-detector/sightings.db")

# --- Your own devices ------------------------------------------------
# MACs listed here are dropped at capture time — never written to the
# database. You already know they're you; excluding them here keeps
# the correlation logic from ever flagging your own phone.
OWN_MACS = {
    # "aa:bb:cc:dd:ee:ff",   # <- your phone
}

# --- Session segmentation (no GPS required) ---------------------------
SESSION_GAP_MINUTES = 20

# --- Correlation / alerting --------------------------------------------
# A MAC is flagged only after it appears in this many different sessions
SESSION_THRESHOLD = 3
LOOKBACK_HOURS = 24

# Whether to also log the SSID being probed for. OFF by default
# tool doesn't need a stranger's network history to detect a tail, only whether their MAC persists across sessions
LOG_SSID = False

# Push notification topic (ntfy.sh) — set to your own private topic name.
NTFY_TOPIC = None  # e.g. "my-private-tail-alerts-8f2k1"

# Multi-band channel list
CHANNELS = [1, 6, 11, 36, 40, 44, 48, 149, 153, 157, 161]
CHANNEL_HOP_INTERVAL = 0.5

# Track MACs already alerted on this run, so we don't spam repeat alerts
ALREADY_ALERTED = set()

# CHANNEL HOPPER
def channel_hopper(interface, stop_event):
    while not stop_event.is_set():
        for ch in CHANNELS:
            if stop_event.is_set():
                return
            try:
                subprocess.run(
                    ["iwconfig", interface, "channel", str(ch)],
                    stdout=subprocess.DEVNULL,
                    stderr=subprocess.DEVNULL,
                )
            except Exception:
                pass
            time.sleep(CHANNEL_HOP_INTERVAL)

# DATABASE
def init_db():
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS sightings (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT,
            mac_address TEXT,
            ssid TEXT,
            rssi INTEGER,
            session_id INTEGER
        )
        """
    )
    conn.commit()
    conn.close()


def get_current_session_id(conn):
    """Returns the active session id, starting a new one if the gap since
    the last sighting (of ANY device) exceeds SESSION_GAP_MINUTES."""
    cursor = conn.cursor()
    cursor.execute("SELECT timestamp, session_id FROM sightings ORDER BY id DESC LIMIT 1")
    row = cursor.fetchone()
    now = datetime.datetime.now()

    if row is None:
        return 1

    last_ts = datetime.datetime.strptime(row[0], "%Y-%m-%d %H:%M:%S")
    last_session = row[1]
    gap = (now - last_ts).total_seconds() / 60.0

    if gap > SESSION_GAP_MINUTES:
        return last_session + 1
    return last_session


# CORRELATION CHECK
def check_correlation(conn, mac_addr):
    """Returns (distinct_session_count, first_seen, last_seen) for this MAC
    within the lookback window."""
    cursor = conn.cursor()
    cutoff = (datetime.datetime.now() - datetime.timedelta(hours=LOOKBACK_HOURS)).strftime("%Y-%m-%d %H:%M:%S")
    cursor.execute(
        """
        SELECT COUNT(DISTINCT session_id), MIN(timestamp), MAX(timestamp)
        FROM sightings
        WHERE mac_address = ? AND timestamp >= ?
        """,
        (mac_addr, cutoff),
    )
    count, first_seen, last_seen = cursor.fetchone()
    return count or 0, first_seen, last_seen


def send_tail_alert(mac_addr, session_count, first_seen, last_seen):
    print(f"\n[!] POSSIBLE TAIL: {mac_addr} seen in {session_count} distinct sessions")
    print(f"    First seen: {first_seen}")
    print(f"    Last seen:  {last_seen}\n")

    if not NTFY_TOPIC:
        return

    payload = (
        f"Device {mac_addr} has appeared in {session_count} separate sessions "
        f"in the last {LOOKBACK_HOURS}h.\nFirst: {first_seen}\nLast: {last_seen}"
    )
    headers = {
        "Title": "Possible tail detected",
        "Priority": "high",
        "Tags": "eyes",
    }
    try:
        requests.post(f"https://ntfy.sh/{NTFY_TOPIC}", data=payload.encode("utf-8"), headers=headers, timeout=5)
    except Exception as e:
        print(f"[-] Alert send failed: {e}")

# FRAME PARSER
def make_packet_handler(session_stats):
    def packet_handler(packet):
        if not packet.haslayer(Dot11ProbeReq):
            return

        mac_addr = packet.getlayer(Dot11).addr2
        if mac_addr is None:
            return

        session_stats["seen_total"] += 1

        if mac_addr.lower() in OWN_MACS:
            return  # never logged — it's you

        rssi = getattr(packet, "dBm_AntSignal", None)

        ssid = ""
        if LOG_SSID:
            elt = packet.getlayer(Dot11Elt)
            while isinstance(elt, Dot11Elt):
                if elt.ID == 0:
                    try:
                        ssid = elt.info.decode("utf-8", errors="ignore")
                    except Exception:
                        ssid = ""
                    break
                elt = elt.payload

        now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        conn = sqlite3.connect(DB_PATH)
        try:
            session_id = get_current_session_id(conn)
            cursor = conn.cursor()
            cursor.execute(
                "INSERT INTO sightings (timestamp, mac_address, ssid, rssi, session_id) VALUES (?, ?, ?, ?, ?)",
                (now, mac_addr, ssid, rssi, session_id),
            )
            conn.commit()

            session_stats["seen_logged"] += 1
            print(f"[+] [{now}] session={session_id} MAC={mac_addr} RSSI={rssi}")

            if mac_addr not in ALREADY_ALERTED:
                count, first_seen, last_seen = check_correlation(conn, mac_addr)
                if count >= SESSION_THRESHOLD:
                    send_tail_alert(mac_addr, count, first_seen, last_seen)
                    ALREADY_ALERTED.add(mac_addr)
        except Exception as e:
            print(f"[-] DB error: {e}")
        finally:
            conn.close()

    return packet_handler

# MAIN
def main():
    parser = argparse.ArgumentParser(description="Portable correlation-based tail detector")
    parser.add_argument("--interface", default=INTERFACE)
    parser.add_argument("--no-hop", action="store_true")
    parser.add_argument("--duration", type=int, default=0, help="Auto-stop after N seconds (0 = run until Ctrl+C)")
    args = parser.parse_args()

    init_db()

    print("[*] Probe-Request Tail Detector")
    print(f"[*] Interface:         {args.interface}")
    print(f"[*] Database:          {DB_PATH}")
    print(f"[*] Session gap:       {SESSION_GAP_MINUTES} min")
    print(f"[*] Alert threshold:   {SESSION_THRESHOLD} distinct sessions within {LOOKBACK_HOURS}h")
    print(f"[*] SSID logging:      {'ON' if LOG_SSID else 'OFF (default)'}")
    print(f"[*] Own devices excl.: {len(OWN_MACS)} MAC(s)\n")

    if not NTFY_TOPIC:
        print("[!] NTFY_TOPIC is not set — alerts will only print to console, not push to your phone.")
        print("    Set NTFY_TOPIC to a private topic name to enable push notifications.\n")

    stop_event = threading.Event()
    session_stats = {"seen_total": 0, "seen_logged": 0}

    if not args.no_hop:
        threading.Thread(target=channel_hopper, args=(args.interface, stop_event), daemon=True).start()

    handler = make_packet_handler(session_stats)

    try:
        sniff(
            iface=args.interface,
            prn=handler,
            store=0,
            timeout=args.duration if args.duration > 0 else None,
        )
    except KeyboardInterrupt:
        pass
    finally:
        stop_event.set()
        print(f"\n[*] Stopped. Total probes seen: {session_stats['seen_total']}, logged: {session_stats['seen_logged']}")
        print("[*] Run `python3 correlate.py` for a full report.")


if __name__ == "__main__":
    main()
```

Create `correlate.py`:
```
nano ~/tail-detector/correlate.py
```
Paste in the following, then save and exit (Ctrl+O, Enter, Ctrl+X):
```python
#!/usr/bin/env python3
import datetime
import os
import sqlite3

DB_PATH = os.path.expanduser("~/tail-detector/sightings.db")

# Keep in sync with sniffer.py if you change these there
SESSION_THRESHOLD = 3
LOOKBACK_HOURS = 24


def main():
    if not os.path.exists(DB_PATH):
        print(f"[-] No database found at {DB_PATH}. Run sniffer.py first.")
        return

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cutoff = (datetime.datetime.now() - datetime.timedelta(hours=LOOKBACK_HOURS)).strftime("%Y-%m-%d %H:%M:%S")

    print("=" * 60)
    print("  TAIL DETECTOR — CORRELATION REPORT")
    print(f"  Lookback window: {LOOKBACK_HOURS}h   Threshold: {SESSION_THRESHOLD} sessions")
    print("=" * 60)

    cursor.execute("SELECT COUNT(*) FROM sightings WHERE timestamp >= ?", (cutoff,))
    total = cursor.fetchone()[0]
    print(f"\nTotal sightings in window: {total}")

    cursor.execute("SELECT COUNT(DISTINCT session_id) FROM sightings WHERE timestamp >= ?", (cutoff,))
    sessions = cursor.fetchone()[0]
    print(f"Distinct sessions in window: {sessions}\n")

    cursor.execute(
        """
        SELECT mac_address, COUNT(DISTINCT session_id) as session_count,
               MIN(timestamp) as first_seen, MAX(timestamp) as last_seen,
               MAX(rssi) as peak_rssi
        FROM sightings
        WHERE timestamp >= ?
        GROUP BY mac_address
        ORDER BY session_count DESC
        LIMIT 20
        """,
        (cutoff,),
    )
    rows = cursor.fetchall()

    if not rows:
        print("No sightings in this window yet.")
        conn.close()
        return

    print(f"{'MAC Address':<20}{'Sessions':<10}{'Peak RSSI':<12}{'First Seen':<22}{'Last Seen'}")
    print("-" * 90)
    flagged = []
    for mac, count, first_seen, last_seen, peak_rssi in rows:
        marker = " <-- FLAGGED" if count >= SESSION_THRESHOLD else ""
        if marker:
            flagged.append(mac)
        print(f"{mac:<20}{count:<10}{str(peak_rssi):<12}{first_seen:<22}{last_seen}{marker}")

    print()
    if flagged:
        print(f"[!] {len(flagged)} device(s) crossed the {SESSION_THRESHOLD}-session threshold:")
        for mac in flagged:
            print(f"    - {mac}")
        print("\n    A device reappearing across several separate sessions is the")
        print("    core signal this tool looks for. Consider context before drawing")
        print("    conclusions: dense areas (transit, offices) produce more false")
        print("    positives than open/residential areas.")
    else:
        print("[*] No devices crossed the correlation threshold in this window.")

    conn.close()


if __name__ == "__main__":
    main()
```

Make both scripts executable (optional, since they're run via `python3`):
```
chmod +x ~/tail-detector/sniffer.py ~/tail-detector/correlate.py
```
---
### 6. Exclude Your Own Devices
Find your phone/laptop's MAC address so it's excluded from capture
entirely (never written to the database — you already know it's you):

- **iOS:** Settings → General → About → Wi-Fi Address
- **Android:** Settings → About Phone → Status → Wi-Fi MAC Address
- **macOS:** `networksetup -getmacaddress Wi-Fi`
- **Windows:** `ipconfig /all` → Wi-Fi adapter's Physical Address
- **Linux:** `ip link show`

> **Note:** iOS 14+ and Android 10+ randomize MACs for background
> scanning by default. Disable "Private Wi-Fi Address" for a network in
> your phone's Wi-Fi settings if you want to exclude your device's real
> hardware MAC specifically.

Edit `sniffer.py`:
```python
OWN_MACS = {
    "aa:bb:cc:dd:ee:ff",   # your phone
    "11:22:33:44:55:66",   # your laptop, if also carried
}
```
---
### 7. Configure Detection Parameters
Open `sniffer.py` and adjust as needed (defaults shown):
```python
SESSION_GAP_MINUTES = 20   # gap before a new "session" starts
SESSION_THRESHOLD   = 3    # distinct sessions before a MAC is flagged
LOOKBACK_HOURS      = 24   # correlation window
LOG_SSID            = False  # keep off unless you have a specific reason
```
---
### 8. (Optional) Enable Push Notifications
Set a private [ntfy.sh](https://ntfy.sh) topic name so alerts push to
your phone instead of only printing to console:
```python
NTFY_TOPIC = "your-private-topic-name-here"
```
Subscribe to the same topic in the ntfy app to receive alerts.

---
### 9. Run It
```
sudo python3 sniffer.py
```
Carry the Pi with you — backpack, bag, powered by the power bank as
you go about your day. Let it accumulate multiple sessions before expecting meaningful results.

Stop anytime with `Ctrl+C`.

---
### 10. Review Results
```
python3 correlate.py
```
Prints every MAC seen in the lookback window, how many distinct sessions
it appeared in, and flags anything at or above the threshold.

---
### Tuning Reference

| Setting               | Default | Effect                                                                 |
| --------------------- | ------- | ---------------------------------------------------------------------- |
| `SESSION_GAP_MINUTES` | 20      | Shorter = more/smaller sessions (more sensitive, more false positives) |
| `SESSION_THRESHOLD`   | 3       | Distinct sessions required before a MAC is flagged                     |
| `LOOKBACK_HOURS`      | 24      | How far back correlation looks                                         |
| `LOG_SSID`            | `False` | Whether to also record probed SSIDs (off by default)                   |
