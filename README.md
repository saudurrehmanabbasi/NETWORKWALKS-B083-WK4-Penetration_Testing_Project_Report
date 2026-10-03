<div align="center">
  
# Penetration Testing Report — Mediroza General Hospital

**External Black-Box Web Application Assessment**

**Target:** https://medirozahospital.com

**Batch B083 | Week 4**

Submitted as part of: Cybersecurity and Ethical Hacking Internship — Network Walks <br> Prepared by: Saud Ur Rehman Abbasi, Cybersecurity Intern <br>

</div>

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope and Methodology](#2-scope-and-methodology)
3. [Findings and Proof of Exploitation](#3-findings-and-proof-of-exploitation)
4. [Risk Rating](#4-risk-rating)
5. [Recommendations and Remediation](#5-recommendations-and-remediation)

---

## 1. Executive Summary

This report was prepared as part of a hands-on cybersecurity internship project, carried out as a guided penetration testing training exercise. The task simulated an external, black-box security assessment of a public-facing patient portal, https://medirozahospital.com (a controlled training target used for the exercise), with the objective of applying real-world penetration testing methodology to identify exploitable vulnerabilities that could allow an unauthenticated attacker to compromise the confidentiality, integrity, or availability of patient and organizational data, and to produce practical remediation guidance in a professional report format.

Testing was carried out over three milestones — initial access, encryption analysis, and deep reconnaissance — using manual testing techniques supported by standard open-source tools (Gobuster, Hashcat) and online utilities. The assessment uncovered a chain of critical weaknesses that, together, allowed full compromise of the patient portal's access controls and exposure of highly sensitive data without any valid credentials.

**Key findings include:**

- A username enumeration flaw in the login form that disclosed valid account names through inconsistent error messages.
- A SQL injection vulnerability in the login mechanism that allowed complete authentication bypass, granting unauthenticated access to the "admin" account and the patient report portal.
- Three confidential patient reports protected only by weak, easily guessable passwords, all of which were cracked using password-cracking tools and publicly available wordlists.
- An unprotected backup directory (`/old/`) discovered through directory brute-forcing, containing a full SQL database backup that exposed staff personally identifiable information (national ID numbers, salaries, contact details) and confidential shareholder/financial records.

Collectively, these issues represent a severe and immediate risk to Mediroza General Hospital. An attacker with no prior access and no credentials was able to fully bypass authentication, exfiltrate protected patient documents, and recover a complete backup of sensitive staff and financial data. **The overall risk to the organization is rated CRITICAL.** Immediate remediation of the SQL injection vulnerability and removal of the exposed backup file should be treated as top priorities, followed by the broader hardening measures detailed in Section 5.

---

## 2. Scope and Methodology

### 2.1 Target

| | |
|---|---|
| **In-scope asset** | https://medirozahospital.com (patient portal and associated public web root) |
| **Assessment type** | External, unauthenticated (black-box) web application penetration test |
| **Engagement reference** | Batch B083 — Week 4 |

### 2.2 Approach

Testing followed a staged approach, mirroring the three milestones of the engagement:

- **Milestone 1 – Initial Access:** manual testing of the patient login form for logic flaws, username enumeration, and injection vulnerabilities, leading to authentication bypass and retrieval of protected patient reports.
- **Milestone 2 – Encryption Analysis:** extraction of password hashes from the retrieved PDF reports and offline/online cracking attempts using common-password and large breach-derived wordlists.
- **Milestone 3 – Deep Reconnaissance:** automated content discovery against the web root to identify hidden files and directories not linked from the application, followed by manual review of anything exposed.

### 2.3 Tools Used

- Manual SQL injection testing against the login form
- Networkwalks Hash Calculator (online) — PDF hash extraction
- Networkwalks Password Cracker (online) — common-password wordlist attack
- Hashcat (Kali Linux), mode `-m 10500` (PDF 1.7 Level 3), with the `rockyou.txt` wordlist
- Gobuster v3.8.2, directory/file brute-forcing with the `dirb` `common.txt` wordlist
- Kali Linux as the testing platform

### 2.4 Limitations

- Testing was performed as an unauthenticated external assessment; no source code, infrastructure diagrams, or credentials were provided by the client.
- Testing was limited to the public web application and its exposed web root; no testing was performed against internal networks, mobile apps, or third-party integrations.
- Password cracking relied on publicly available wordlists (built-in top-100 list and `rockyou.txt`); passwords not present in these lists would require additional time or custom wordlists to recover.
- Content discovery used a standard wordlist (`dirb` `common.txt`); a larger or custom wordlist may reveal additional hidden resources beyond those reported here.
- This assessment represents a point-in-time review; it does not guarantee the absence of other vulnerabilities not covered by the scope or methodology above.

---

## 3. Findings and Proof of Exploitation

Findings are grouped by milestone, in the order they were discovered and chained together during the engagement.

### 3.1 Milestone 1 — Initial Access: Username Enumeration & SQL Injection Authentication Bypass

#### Finding 1.1 — Username Enumeration on Login Form

The patient login page at `https://medirozahospital.com/patient/login.php` returned different error messages depending on whether the submitted username existed, allowing an attacker to enumerate valid accounts without any credentials.

**Steps and evidence:**

Submitted a non-existent username with an arbitrary password:

```
username: bob
password: test123

RESPONSE: "Username not found"
```

Submitted the common administrative username "admin" with an arbitrary password:

```
username: admin
password: test123

RESPONSE: "Incorrect password"
```

The two distinct responses confirm that "admin" is a valid account on the system. A secure login implementation should return an identical, generic message (e.g. "Invalid username or password") regardless of which field is incorrect.

> 🖼️ *[Insert screenshot here: both login attempts and their respective error messages (bob / admin)]*

#### Finding 1.2 — SQL Injection in Login Form

The username field of the login form does not sanitize or parameterize user input before placing it into a SQL query, making it vulnerable to SQL injection.

**Steps and evidence:**

Submitted a single quote in the username field to test for injectable syntax:

```
username: admin'
password: test123
```

The application returned a raw database error, confirming the input is passed directly into a SQL statement:

```
Warning: mysqli_query(): You have an error in your SQL syntax; check the manual
that corresponds to your MySQL server version for the right syntax to use near
''' at line 1
```

> 🖼️ *[Insert screenshot here: the MySQL syntax error returned to the browser]*

#### Finding 1.3 — Authentication Bypass via SQL Injection

Building on Finding 1.2, the SQL injection was weaponized into a full authentication bypass. The payload `admin' --` breaks out of the intended query string with a single quote, then uses the SQL comment sequence `--` to discard the remainder of the query, including the password check. The resulting query matches the admin account with no password condition at all.

```
username: admin' --
password: anything
```

**Result:** the attacker was logged in as the admin user with no valid credentials. The patient portal loaded showing 3 confidential patient reports, which were downloaded in full:

- `patient_report_1.pdf`
- `patient_report_2.pdf`
- `patient_report_3.pdf`

> 🖼️ *[Insert screenshot here: the successful login bypass and the portal page listing the 3 downloadable reports]*

**Impact:** An unauthenticated attacker can gain full administrative access to the patient portal and exfiltrate confidential patient records, with no prior knowledge of any password. This is the single most severe finding in this engagement and is the root cause that enabled every subsequent finding in Milestone 2.

### 3.2 Milestone 2 — Weak Encryption on Confidential Patient Reports

The three PDF reports obtained in Milestone 1 were password-protected using PDF standard encryption. Each PDF's password was recovered using hash extraction followed by a dictionary (wordlist) attack.

#### Finding 2.1 — Hash Extraction

Each PDF was uploaded to an online PDF hash calculator, which extracted a crackable hash (format `$pdf$...`) representing the document's encryption password.

> 🖼️ *[Insert screenshot here: each PDF hash as generated by the hash calculator]*

#### Finding 2.2 — Cracking Reports 1 and 2 (Common-Password Wordlist)

Both hashes were submitted to an online password cracker using its built-in wordlist of the 100 most common passwords. Both passwords were recovered within seconds:

```
patient_report_1.pdf  ->  123456
patient_report_2.pdf  ->  password
```

> 🖼️ *[Insert screenshot here: the cracking results for reports 1 and 2]*

#### Finding 2.3 — Cracking Report 3 (rockyou.txt / Hashcat)

Report 3 was not cracked by the built-in top-100 wordlist, returning "Exhausted wordlist – ACCESS DENIED." The attack was escalated to Hashcat on Kali Linux using the much larger `rockyou.txt` wordlist (over 14 million real-world breached passwords):

```bash
hashcat -m 10500 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

This recovered the password for report 3:

```
patient_report_3.pdf  ->  !@#$%^&
```

> 🖼️ *[Insert screenshot here: the Hashcat session and cracked password for report 3]*

**Impact:** All three confidential patient reports relied on weak, dictionary-crackable passwords rather than strong, unique encryption keys. Combined with the authentication bypass in Milestone 1, an attacker can obtain and fully decrypt all protected patient documents in minutes using freely available tools, with no advanced skill required.

### 3.3 Milestone 3 — Deep Reconnaissance: Exposed Database Backup

To look beyond the areas of the application discovered through normal browsing, a content-discovery scan was run against the web root to identify hidden files and directories not linked from any visible page.

#### Finding 3.1 — Directory and File Brute-Forcing

Gobuster was run against the target root using the `dirb` `common.txt` wordlist with a set of common backup and configuration extensions, to search for forgotten or unlinked resources:

```bash
gobuster dir -u https://medirozahospital.com/ -w /usr/share/wordlists/dirb/common.txt \
  -x bak,zip,sql,txt,old,conf,env -t 50
```

The scan returned numerous paths, including administrative and configuration-style paths that were access-restricted (HTTP 403), but also returned an accessible, unlinked directory: `/old/`.

![Gobuster scan starting against medirozahospital.com, showing restricted (403) results for common sensitive paths.](images/image1.png)
*Figure 1 — Gobuster scan starting against medirozahospital.com, showing restricted (403) results for common sensitive paths.*

![Gobuster scan results continued, completing 36,904 requests at 100%.](images/image2.png)
*Figure 2 — Gobuster scan results continued, completing 36,904 requests at 100%.*

#### Finding 3.2 — Unprotected Legacy Backup Directory

Browsing to the discovered `/old/` path returned a directory listing — confirming it is publicly accessible with no authentication and no access restriction — containing a single file: `mediroza_db_backup_2019.sql` (7 KB).

![Directory listing of https://medirozahospital.com/old/ exposing mediroza_db_backup_2019.sql.](images/image3.png)
*Figure 3 — Directory listing of https://medirozahospital.com/old/ exposing mediroza_db_backup_2019.sql.*

#### Finding 3.3 — Sensitive Data Exposure in the Backup File

The exposed SQL file was downloaded directly, with no authentication, and found to contain a full export of the hospital's staff table, including names, job titles, departments, email addresses, phone numbers, national identification numbers, monthly salaries, and dates joined for all 30 staff records.

![Excerpt of the exposed staff table: full names, national ID numbers, salaries, and contact details for all staff.](images/image4.png)
*Figure 4 — Excerpt of the exposed `staff` table: full names, national ID numbers, salaries, and contact details for all staff.*

The same backup also contained a shareholders table, disclosing the hospital's private ownership structure, including individual shareholder names, shareholding percentages, and share class — financial information with no business reason to be exposed on a public web server.

![Excerpt of the exposed shareholders table: ownership percentages and shareholdings.](images/image5.png)
*Figure 5 — Excerpt of the exposed `shareholders` table: ownership percentages and shareholdings.*

**Impact:** This finding alone, independent of Milestones 1 and 2, constitutes a severe data breach. An unauthenticated attacker who simply ran a directory brute-force scan was able to download a complete backup of highly sensitive personal data (including national ID numbers, which enable identity theft) for every member of staff, together with confidential corporate ownership and financial records. This exposure is likely reportable as a data breach under applicable data-protection regulation and could expose the hospital to regulatory, financial, and reputational harm independent of the other findings.

---

## 4. Risk Rating

Each finding is rated Critical, High, Medium, or Low based on its ease of exploitation (an unauthenticated attacker with no special access was able to exploit every finding below) and the sensitivity of the data or access it exposes.

| Finding | Rating | Justification |
|---|---|---|
| 1.1 — Username enumeration on login form | 🟢 **Low** | Does not directly expose data, but allows an attacker to confirm valid accounts (e.g. "admin") as a precursor to credential attacks. Low effort to exploit, low direct impact on its own. |
| 1.2 / 1.3 — SQL injection & full authentication bypass | 🔴 **Critical** | Trivial to exploit (single payload, no tools required), requires no credentials, and grants full administrative access to the patient portal and confidential patient reports. Root cause enabling the rest of the data exposure chain. |
| 2.1–2.3 — Weak PDF passwords (patient reports 1–3) | 🟠 **High** | All three encryption passwords were recoverable with free, widely available tools and wordlists within minutes to hours. Directly exposes confidential patient report contents once the files are obtained. |
| 3.1 / 3.2 — Unprotected legacy backup directory (`/old/`) | 🔴 **Critical** | No authentication or access control of any kind; discoverable with a basic, default-wordlist directory scan. Directly serves a full database backup file to any unauthenticated visitor. |
| 3.3 — Exposure of staff PII and shareholder/financial records | 🔴 **Critical** | Exposes highly sensitive personal data (national ID numbers, salaries, contact details) for all staff and confidential ownership/financial data, with no exploitation required beyond downloading a linked file. High likelihood of regulatory and reputational impact. |

**Overall engagement risk rating: CRITICAL.** The combination of an unauthenticated authentication bypass and an unauthenticated data exposure means two independent paths exist for a complete, unauthenticated compromise of confidential patient and corporate data.

---

## 5. Recommendations and Remediation

### 5.1 Username Enumeration

- Return a single, generic error message (e.g. "Invalid username or password") for all failed login attempts, regardless of whether the username or password was incorrect.
- Ensure login response times are consistent regardless of whether the username exists, to prevent timing-based enumeration.
- Apply rate limiting and account lockout/backoff on the login endpoint to slow down enumeration and brute-force attempts.

### 5.2 SQL Injection / Authentication Bypass

- Rewrite all database queries, especially the login query, to use parameterized queries / prepared statements rather than concatenating user input into SQL strings. This eliminates the injection vector entirely rather than attempting to filter malicious input.
- Adopt an ORM or a vetted database access layer that enforces parameterization by default across the codebase, not just the login form.
- Disable detailed database error messages in production; log them server-side only and return a generic error page to the user.
- Conduct a full source-code review of all other input fields and endpoints for the same unparameterized-query pattern, as this is rarely an isolated issue.
- Enforce multi-factor authentication for administrative accounts such as "admin", so that a bypassed password check alone cannot grant full access.

### 5.3 Weak Document Encryption / Passwords

- Replace manually chosen PDF passwords with strong, randomly generated passwords (minimum 16 characters) or, preferably, move away from password-based PDF encryption toward key-based access control managed by the application (e.g. reports served only to authenticated, authorized sessions, with no standalone encrypted file ever distributed).
- If password-protected PDFs must be used, generate passwords programmatically and never reuse common or dictionary words ("123456", "password", or simple symbol patterns).
- Enforce a minimum password policy (length, complexity, and a check against known-breached password lists such as `rockyou.txt` or Have I Been Pwned) for any password protecting sensitive documents.

### 5.4 Exposed Backup Directory and Data Exposure

- Remove the `/old/` directory and the `mediroza_db_backup_2019.sql` file from the public web server immediately; this is an emergency action independent of the rest of this report.
- Audit the entire public web root for other forgotten, legacy, or backup files (`.bak`, `.zip`, `.sql`, `.old`, `.env`, `.conf`) and remove or relocate anything that is not intended to be served publicly.
- Store database backups outside the web-accessible document root entirely, in a dedicated, access-controlled backup location (e.g. encrypted object storage with strict IAM policies), never inside a folder reachable by a web request.
- Encrypt all database backups at rest, so that even an accidental exposure does not yield plaintext sensitive data.
- Implement a web application firewall (WAF) and disable directory listing on the web server to reduce the impact of any future misconfiguration.
- Run periodic automated content-discovery and configuration-review scans (e.g. Gobuster, Nikto) against the production environment as part of ongoing security operations, not only during point-in-time penetration tests.

### 5.5 Data Protection / Governance

- Treat the exposure of staff national ID numbers, salaries, and shareholder financial data as a potential data breach requiring assessment against applicable data-protection and breach-notification obligations.
- Review and minimize what personal and financial data is retained in systems connected to, or backed up alongside, the public-facing patient portal.
- Conduct staff security awareness and secure-development training, with emphasis on secure query construction, secrets/backup handling, and least-privilege data storage.

### 5.6 Suggested Remediation Priority

| Priority | Action | Rating | Justification |
|---|---|---|---|
| 1 | Remove exposed backup file / disable directory listing | 🔴 **Critical** | Immediate action — stops an active, unauthenticated data exposure with zero exploitation effort required. |
| 2 | Fix SQL injection in login (parameterized queries) | 🔴 **Critical** | Immediate action — closes the root-cause authentication bypass enabling unauthorized access to patient data. |
| 3 | Rotate/strengthen document encryption passwords | 🟠 **High** | Short-term — prevents recovery of protected report contents even if a file is obtained. |
| 4 | Generic login error messages / rate limiting | 🟢 **Low** | Short-term hardening — reduces reconnaissance value for future attackers. |
| 5 | Backup storage, encryption-at-rest, and governance review | 🟡 **Medium** | Medium-term — addresses the systemic process gap that allowed sensitive data to end up in a public location. |

---

*End of Report*
