**Category:** OSINT
**Difficulty:** Easy

---
## Investigation Summary
```
Image Metadata → Username → Email & Real Name → Crypto Wallet → WiFi BSSID → Flight Path → Home City
```

### Reconnaissance (Image Metadata)

#### Challenge Question
"Images can contain a treasure trove of information, both on the surface as well as embedded within the file itself. In order to answer the following question, you will need to thoroughly analyze the image found by the OSINT Dojo administrators in order to obtain basic information on the attacker. What username does the attacker go by?"

#### Approach

Downloaded the SVG file and examined its raw contents:
```bash
wget https://raw.githubusercontent.com/OsintDojo/public/3f178408909bc1aae7ea2f51126984a8813b0901/sakurapwnedletter.svg
cat sakurapwnedletter.svg
```

Found the following metadata embedded in the SVG file:
```
inkscape:export-filename="/home/SakuraSnowAngelAiko/Desktop/pwnedletter.png"
```

The file path revealed the attacker's Linux home directory username.

#### Answer
`SakuraSnowAngelAiko`

#### Key Takeaway
SVG files are XML-based and contain readable metadata including file paths, software used, and author information. Always examine raw file contents. The attacker made a critical mistake by leaving their username embedded in the file export path.

---

### Unveil (Username Enumeration)

#### Challenge Question
"It appears that our attacker made a fatal mistake in their operational security. They seem to have reused their username across other social media platforms as well. Use the attacker's username found in Task 2 to expand the OSINT investigation onto other platforms. What is the full email address used by the attacker? What is the attacker's full real name?"

#### Approach

**Step 1 — Run Sherlock to find accounts across platforms:**
```bash
sherlock SakuraSnowAngelAiko
```

Sherlock found 40+ accounts including GitHub, Reddit, Twitter, LinkedIn and more.

**Step 2 — Check GitHub profile:**
Visited `https://github.com/SakuraSnowAngelAiko` and found nine repositories including a **PGP** repo containing their public PGP key.

**Step 3 — Extract email from PGP key using CyberChef:**
Downloaded the PGP key and decoded it using CyberChef (From Base64). The decoded output contained the email address in plain text:
```
SakuraSnowAngel83@protonmail.com
```

**Step 4 — Find real name via Twitter:**
Found Twitter account `@SakuraLoverAiko`. Scrolling through tweets found:

> "Silly me, I forgot to introduce myself! Hi there! I'm @AikoAbe3!"

#### Answers
- **Email:** `SakuraSnowAngel83@protonmail.com`
- **Full name:** `Aiko Abe`

#### Key Takeaway
Username reuse across platforms is one of the most common OSINT vulnerabilities. PGP public keys contain the owner's name and email in readable form. Tools like Sherlock automate platform discovery (always manually verify results not all hits are valid).

---

### Taunt (Cryptocurrency Investigation)

#### Challenge Question
"It seems the cybercriminal is aware that we are on to them. As we were investigating their Github account we observed indicators that the account owner had already begun editing and deleting information. Perform a deeper dive into the attacker's Github account for any additional information that may have been altered or removed. What cryptocurrency does the attacker own a wallet for? What is the attacker's cryptocurrency wallet address? What mining pool did the attacker receive payments from on January 23, 2021 UTC? What other cryptocurrency did the attacker exchange with?"

#### Approach

**Step 1 — Search GitHub repos for deleted content:**
Found a repo named `ETH` containing a mining script:
```
stratum://0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef.Aiko:pswd@eu1.ethermine.org:4444
```

This revealed the cryptocurrency type, wallet address, and mining pool.

**Step 2 — Trace wallet transactions on Etherscan:**
Visited:
```
https://etherscan.io/address/0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef
```

Scrolling through the transaction history found multiple OUT transactions to **Tether: USDT Stablecoin**.

#### Answers
- **Cryptocurrency:** `Ethereum (ETH)`
- **Wallet address:** `0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef`
- **Mining pool:** `Ethermine`
- **Other cryptocurrency:** `Tether (USDT)`

#### Key Takeaway
Git repositories store full commit history — even "deleted" content can often be recovered by examining previous commits. Cryptocurrency transactions are permanently recorded on the blockchain and fully public, making them a powerful OSINT resource.

---

### Homebound (WiFi BSSID Lookup)

#### Challenge Question
"The cybercriminal taunted us using a different Twitter username. Their Twitter account appears to have photos which should allow us to track their movements. What is the attacker's current Twitter handle? What is the BSSID for the attacker's Home WiFi?"

#### Approach

**Step 1 — View the taunt image:**
```
https://raw.githubusercontent.com/OsintDojo/public/main/taunt.png
```

The message was sent from Twitter handle `@SakuraLoverAiko`.

**Step 2 — Find WiFi SSID from Twitter:**
Scrolled through `@SakuraLoverAiko` tweets and found posts about storing WiFi passwords on paste sites. Located the WiFi network name `DK1F-G` from the attacker's posts.

**Step 3 — Search Wigle.net for BSSID:**
Visited `https://wigle.net` and searched for SSID `DK1F-G`. The result returned:

- **BSSID:** `84:AF:EC:34:FC:F8`
- **Estimated coordinates:** Lat 40.60551453, Long 140.4606781

#### Answers
- **Twitter handle:** `SakuraLoverAiko`
- **BSSID:** `84:AF:EC:34:FC:F8`

#### Key Takeaway
Wigle.net is a global database of WiFi networks collected by wardriving. If you know a network's SSID you can often locate it geographically. 

BSSIDs are unique hardware identifiers for WiFi access points and can pinpoint a physical location.

---

### Final Trace (Flight Path & Home City)

#### Challenge Question
"Based on their tweets, it appears our cybercriminal is heading home. Their Twitter account has photos which should allow us to piece together their route back home. What airport is closest to the location the attacker shared a photo from prior to getting on their flight? What airport did the attacker have their last layover in? What lake can be seen in the map shared by the attacker? What city does the attacker likely consider home?"

#### Approach

**Step 1 — Identify departure location:**
Found a photo of cherry blossom trees on `@SakuraLoverAiko`. Used Google Reverse Image Search to identify the specific variety of cherry trees (Okame cherry trees) which are unique to Washington D.C. The closest major airport to D.C. is **DCA (Ronald Reagan Washington National Airport)**.

**Step 2 — Identify layover airport:**
Found a photo showing a **JAL (Japan Airlines)** First Class Lounge. JAL's main hub in Japan is **HND (Tokyo Haneda Airport)**, confirming this as the layover.

**Step 3 — Identify the lake:**
Found a map photo shared during the final flight home. By cross-referencing the island shapes and geographic features visible in the map with Google Maps over Japan, identified the lake as **Lake Inawashiro** in Fukushima.

**Step 4 — Identify home city:**
Cross-referenced the BSSID coordinates from Wigle (Lat 40.60551453, Long 140.4606781) with Google Maps. The coordinates point to **Hirosaki** in Aomori Prefecture, Japan — confirming the attacker's home city.

#### Answers

- **Departure airport:** `DCA`
- **Layover airport:** `HND`
- **Lake:** `Lake Inawashiro`
- **Home city:** `Hirosaki`

#### Key Takeaway
Combining multiple data points (photo metadata, reverse image search, geographic analysis, BSSID coordinates) allows you to track a target's physical movements. No single piece of intelligence tells the whole story, OSINT is about putting together multiple sources.

---
## OSINT Methodology

```
1. Image Metadata Analysis    →  Extract username from SVG file path
2. Username Enumeration       →  Sherlock finds 40+ accounts
3. Platform Investigation     →  GitHub, Twitter, PGP keys
4. Cryptocurrency Tracing     →  Etherscan blockchain analysis
5. WiFi Geolocation           →  Wigle.net BSSID lookup
6. Geographic Analysis        →  Reverse image search + map analysis
7. Intelligence Synthesis     →  Combine all data points for home city
```

---
## Tools Used

| Tool                          | Purpose                                   |
| ----------------------------- | ----------------------------------------- |
| `sherlock`                    | Username enumeration across 40+ platforms |
| `CyberChef`                   | Base64 decode PGP key to extract email    |
| `Etherscan`                   | Trace cryptocurrency transactions         |
| `Wigle.net`                   | WiFi BSSID lookup and geolocation         |
| `Google Reverse Image Search` | Identify locations from photos            |
| `GitHub`                      | Find deleted content in commit history    |
| `Wayback Machine`             | Recover cached/archived content           |

---
## Lessons Learned

- **Metadata is everywhere.** SVG files, images, and documents all contain hidden information that can reveal usernames, paths, and software.
- **Username reuse is a goldmine.** One username can unlock dozens of accounts across platforms.
- **Nothing is deleted.** Git commit history, cached pages, and archived content can recover "deleted" information.
- **The blockchain logs.** Cryptocurrency transactions are permanent, public, and fully traceable.
- **WiFi networks can geolocate you.** BSSIDs logged in databases like Wigle can pinpoint a physical address.
- **OSINT is like a investigation board.** No single clue tells the full story — combine multiple data points to build a complete picture.
