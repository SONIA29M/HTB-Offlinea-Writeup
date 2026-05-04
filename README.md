# 🧩 Hack The Box – Offlinea Challenge Write-Up

> **Course:** CSCE 5552 Cybersecurity Essentials  
> **Instructor:** Dr. Ali Zarafshani  
> **Institution:** Department of Computer Science and Engineering, University of North Texas  
> **Date:** April 17, 2026  

---

## 👥 Team Members

| Name |
|------|
| Sonia Sonia |
| Amarnath Pulluru |
| Sahila Sariyeva |
| Sowgoto Raha Sunny |

---

## 📋 Challenge Overview

| Field | Details |
|-------|---------|
| **Platform** | Hack The Box |
| **Challenge Name** | Offlinea |
| **Category** | Web |
| **Difficulty** | Easy |
| **Flag** | `HTB{3b0cbe383374cf3f02336f9e0822997d}` |
| **HTB Achievement** | [View Here](https://labs.hackthebox.com/achievement/challenge/3164896/1108) |

---

## 🛠️ Tools Used

- `ping` – Host availability check
- `whatweb` – Web technology fingerprinting
- `curl` – HTTP requests and exploitation
- `pdfinfo` – PDF metadata analysis
- `pdftotext` – PDF text extraction
- Kali Linux – Attacker machine
- Firefox – Application observation
- Python / JWT library – Token forging

---

## 🔍 Methodology

### 1. Environment Setup

We configured **Kali Linux** as our attacker machine and connected to the Hack The Box target host via **VPN**.

---

### 2. Reconnaissance

- Used `ping` to confirm the target host was active.
- Used `whatweb` to fingerprint the web technology stack.
- Received an **HTTP 200 OK** response confirming the site titled **"Offlinea"** was accessible.
- Verified the host serves a standard HTML5 page.

---

### 3. Application Observation

- Opened the target URL in **Firefox** and accessed the Offlinea web interface.
- Discovered a **web form** that accepts a URL and generates a PDF in response.
- Submitted a test URL and received a valid PDF from the server.

---

### 4. PDF Inspection

- Downloaded `no_way.pdf` from the target application.
- Used `pdfinfo` to confirm it is a valid PDF (version 1.4, 1 page, unencrypted, no embedded JavaScript).
- **Key Finding:** Metadata revealed the PDF was rendered by a **Chromium-based engine**.
- The **Title field** exposed the internal source: `127.0.0.1:8000/bartender.php` — revealing an internal service.

---

### 5. Source Code Analysis

After extracting the lab archive and reviewing the directory structure:

**Public-facing files:**
- `index.html`, `bartender.php`, `pdfs/no_way.pdf`

**Internal backend files:**
- `app.py`, `init_db.py`, `bartender.html`, `Dockerfile`

**Key findings per file:**

| File | Finding |
|------|---------|
| `bartender.php` | Communicates with internal service at `127.0.0.1:5000`; validates user-supplied URLs; checks for private/reserved IP ranges |
| `app.py` | Defines an internal Flask service; uses Selenium + headless Chrome (JavaScript disabled) to render webpages into PDFs |
| `init_db.py` | Creates `secrets` and `history` database tables; inserts the initial flag from `flag.txt` — confirming **the flag is stored in the `secrets` table** |

---

### 6. Vulnerability Identification

#### 🔴 Vulnerability 1: Server-Side Request Forgery (SSRF)

`bartender.php` validates only the **last** URL parameter, but the internal Flask service processes the **first** one. By supplying **duplicate URL parameters**, we bypassed the validation and gained unauthorized access to internal localhost resources.

#### 🔴 Vulnerability 2: Python Format-String Injection

The application formats log data using Python's `.format()` method, where log data is sourced from the `history` table in the database. This allowed us to inject **format-string expressions**, giving access to Flask global objects including `app.config`.

#### 🔴 Vulnerability 3: Flask SECRET_KEY Leakage

Access to `app.config` exposed sensitive configuration values, including the **Flask SECRET_KEY**. With this key, it is possible to forge valid **JWT tokens**.

---

### 7. Exploitation Chain

```
SSRF Bypass → Format-String Injection → SECRET_KEY Leak → JWT Forgery → Flag Retrieval
```

**Step-by-step:**

1. **SSRF Bypass** – Supplied duplicate URL parameters to bypass `bartender.php` IP validation and reach `127.0.0.1:5000`.
2. **Format-String Injection** – Injected format-string expressions via the `history` table to access Flask's `app.config` object.
3. **SECRET_KEY Extraction** – Leaked the Flask `SECRET_KEY` from `app.config` via the vulnerable log rendering function.
4. **JWT Token Forgery** – Used the leaked `SECRET_KEY` to forge a valid JWT token using the **HS256 algorithm**.
5. **Secured Endpoint Access** – Sent the forged JWT via `curl` to `bartender.php`, accessing `127.0.0.1:5000` through SSRF, which accepted the token and returned `final.pdf`.
6. **Flag Retrieval** – Used `pdftotext` to extract the contents of `final.pdf`, which revealed the secrets table and the flag.

---

### 8. Obstacles Encountered

- The application initially blocked SSRF attempts with **"Don't try to trick me!"** — we restarted the unresponsive challenge instance and were assigned a new target host to proceed.
- SSTI (Server-Side Template Injection) through the `name` parameter was also blocked with the same error message, ruling out that attack vector.

---

## 🚩 Flag

```
HTB{3b0cbe383374cf3f02336f9e0822997d}
```

---

## 📖 Key Lessons Learned

- **Chaining vulnerabilities** – Individual low-risk issues can be chained into a critical exploit. An SSRF bypass + format-string injection + JWT forgery together led to full compromise.
- **Parameter parsing differences** – Different backend components (PHP vs Flask) can parse the same request differently, creating exploitable inconsistencies.
- **Metadata leakage** – PDF metadata can inadvertently expose internal infrastructure details.
- **Secure key management** – Flask `SECRET_KEY` must never be accessible through application logic or logs.

---

## 🔗 References

- [HTB Achievement Link](https://labs.hackthebox.com/achievement/challenge/3164896/1108)
- [Hack The Box Platform](https://www.hackthebox.com)
- [OWASP – Server-Side Request Forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [OWASP – JWT Security](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
