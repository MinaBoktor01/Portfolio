# Security Research — Authorized Network Forensic Investigation
**Platform:** Real-World Engagement
**Category:** Network Forensics, OSINT, Surveillance Detection

## Overview
Conducted an authorized network forensic investigation on behalf of residents who suspected an unauthorized surveillance device had been installed on their premises by a third party. Full written consent was obtained from all residents prior to beginning. The goal was to enumerate all devices on the network, identify any with characteristics consistent with IP cameras, and produce documented evidence of findings.

---

## Methodology

### Phase 1 — Environment Setup
Configured Kali Linux VM with access to the target network via bridged networking through QEMU/virt-manager on an Arch Linux host.

### Phase 2 — Host Discovery
Performed ARP and ICMP-based host discovery to enumerate all active devices:

```bash
sudo nmap -sn 192.168.1.0/24
```

Discovered 6 active hosts on the network.

### Phase 3 — Device Classification

| Device | Classification | Method |
|--------|---------------|--------|
| Router | Network infrastructure | Hostname, port 80 |
| Host machine | Known — investigator device | Hostname |
| Smart TV | Known — Roku/Philips | Hostname |
| Mobile device | Known — iOS | Hostname |
| Mobile device | Known — iOS | Hostname |
| Unknown device | Unidentified | No hostname |

### Phase 4 — Unknown Device Investigation
The unidentified device had no hostname and no open ports across all 65535 TCP ports:

```bash
sudo nmap -p- --min-rate 1000 <unknown-ip>
```

TTL fingerprinting via ping returned TTL 63, consistent with Android or Linux:

```bash
ping -c 3 <unknown-ip>
```

UDP scan across common camera ports returned no results:

```bash
sudo nmap -sU -p 37777,554,5353,123 <unknown-ip>
```

MAC OUI lookup returned no identifiable vendor (consistent with MAC address randomization enabled on a modern mobile device).

### Phase 5 — Camera-Specific Port Enumeration
Ran targeted camera port scan across the entire subnet:

```bash
sudo nmap -p 554,37777,34567,8080,8000 192.168.1.0/24
```

No devices on the network had any camera-associated ports open.

### Phase 6 — Findings
No devices consistent with an IP camera were found on the network. Key indicators checked:

| Indicator | Expected (Camera) | Found |
|-----------|------------------|-------|
| RTSP port 554 | Open | Closed on all devices |
| Dahua/Swann port 37777 | Open | Closed on all devices |
| Web UI port 8080 | Open | Closed on all devices |
| Chinese OEM MAC OUI | Present | Not found |
| Active video stream (Wireshark) | RTSP traffic | No RTSP traffic observed |

## Conclusion
The network forensic evidence does not support the presence of an unauthorized IP camera on the WiFi network. All 6 discovered devices were accounted for and classified. The unidentified device displayed characteristics consistent with a mobile device using MAC randomization rather than a surveillance device.

Findings were documented and presented to the residents as evidence.

## Tools Used
| Tool | Purpose |
|------|---------|
| Nmap | Host discovery, port scanning, OS fingerprinting |
| Ping | TTL-based OS fingerprinting |
| ARP scan | Layer 2 device discovery |
| Wireshark | Passive traffic analysis for RTSP streams |
| MAC OUI lookup | Hardware vendor identification |

## Lessons Learned
- IP cameras require at least one open port to serve a stream — no open ports means no active camera service
- TTL values identify OS family: ~64 = Linux/Android, ~128 = Windows, ~255 = iOS/network device
- MAC randomization on modern phones can make devices appear unidentified during network scans
- A full -p- scan is essential (default Nmap only covers the top 1000 ports)
- Passive Wireshark analysis can detect RTSP streams even when active scanning finds nothing
- Documenting negative findings is just as important as documenting positive ones in a real engagement
