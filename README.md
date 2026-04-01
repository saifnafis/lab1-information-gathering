# Lab 1 — Information Gathering, Packet Analysis & Web Exploitation

**Subject:** 48730/32548 Cybersecurity — University of Technology Sydney  
**Tools Used:** Wireshark, Zenmap, Netcraft, John the Ripper, SQL Injection (manual)  
**Skills Demonstrated:** Passive reconnaissance, packet sniffing, credential extraction, password cracking, web application attacks

---

## Overview

This lab covers the full early-stage attack lifecycle: gathering intelligence on a target, intercepting unencrypted traffic to extract credentials, cracking hashed passwords, and exploiting a vulnerable web application through SQL injection. Each task mirrors real techniques used in penetration testing and red team engagements.

---

## Task 1 — Sniffing Credentials from Unencrypted HTTP Traffic

### What I did
Used Wireshark to capture live network traffic on the lab network, then filtered for HTTP POST requests to identify a login form submission sent in plaintext.

### What I found
The captured packet contained a visible HTML form submission with the username and password in cleartext inside the HTTP body. No encryption, no protection — exactly what an attacker on the same network segment would see.

### Why this matters
HTTP does not encrypt data in transit. Any attacker with access to the same network (coffee shop Wi-Fi, a compromised switch, a rogue access point) can capture login credentials using freely available tools like Wireshark. This is one of the most common real-world attack vectors against legacy internal systems and poorly configured web apps.

**Mitigation:** Enforce HTTPS with TLS everywhere. HTTP should be redirected and never used for authentication.

---

## Task 2 — Extracting an Embedded Image from HTTP Traffic

### What I did
Used Wireshark's built-in HTTP object export feature to reconstruct a GIF image (`logo.gif`) directly from captured network packets.

### Why this matters
This demonstrates that an attacker with network access can passively reconstruct files, images, documents, or any unencrypted data being transmitted. This is a real concern for organisations still using unencrypted file transfer protocols or HTTP-based internal portals.

---

## Task 3 — Passive Reconnaissance with Zenmap and Netcraft

### What I did
Ran an active scan against the UTS web server using Zenmap (Nmap GUI) and also gathered passive intelligence using Netcraft's site report tool.

### Findings

| Field | Value |
|---|---|
| Target | www.uts.edu.au |
| IPv4 Address | 172.64.144.213 |
| IP Owner | Cloudflare, Inc. |
| Hosting Location | United States |
| Domain Registrar | audns.net.au |
| Nameserver | ns.uts.edu.au |
| DNS Security | DNSSEC Enabled |

### What I learned
Typing the IP address directly into the browser returned a Cloudflare Error 1003 (direct IP access not allowed). This is because Cloudflare acts as a reverse proxy — the origin server requires a valid `Host` header to route the request correctly. This is a common CDN hardening technique.

Netcraft also exposed the DNS admin email (`dnsadmin@uts.edu.au`), which in a real attack scenario would be a target for phishing or social engineering — a reminder that WHOIS and DNS records can leak sensitive contact information.

### Why this matters
Reconnaissance is the first phase of every real attack. Understanding what information is publicly available about a target allows an attacker to map out attack surface before touching a single system. Defenders need to understand this to know what they are exposing.

---

## Task 4 — Password Cracking with John the Ripper

### What I did
Used John the Ripper (JtR) to crack hashed passwords from a sample `/etc/passwd`-style file on the lab system.

### Result
Successfully cracked the password for user `Eric`: **Student!**

### How JtR works
John the Ripper operates by taking a file of hashed passwords and running candidate passwords through the same hashing algorithm, comparing outputs until a match is found. It supports wordlist attacks (testing common passwords), incremental (brute-force), and rule-based mangling (adding numbers, symbols, capitalising first letters).

The password `Student!` was cracked quickly because it follows a predictable pattern: common word + punctuation. Despite having a special character, it is a weak password because it is short and uses an obvious base word.

### What makes a strong password
From this exercise I learned that length matters far more than complexity. A 7-character password with symbols (`Student!`) is vastly weaker than a 16-character passphrase with no symbols. Modern password guidance recommends at minimum 14 characters, unique per account, ideally managed through a password manager.

---

## Task 5 — SQL Injection Attack

### What I did
Exploited a vulnerable login form on a local lab web application using manual SQL injection payloads, progressively escalating from authentication bypass to full database extraction.

### Attack stages

**Stage 1 — Authentication bypass**  
Entered `'or'1'='1` as both the username and password. The application constructed a SQL query like:

```sql
SELECT * FROM users WHERE username = ''or'1'='1' AND password = ''or'1'='1'
```

Since `'1'='1'` is always true, the query returned all rows and granted access without valid credentials.

**Stage 2 — Dumping the database to disk**  
Used the payload `'or'1'='1' into outfile '/tmp/pass.txt'#` to write all username/password combinations from the database directly to the server filesystem. Running `cat /tmp/pass.txt` on the terminal confirmed the file was written with all user records visible.

**Stage 3 — Targeted account login**  
Used credentials extracted from the dumped file to log in as a specific user, confirming full account takeover.

### Why SQL injection still matters
SQL injection has been in the OWASP Top 10 for over a decade. It works whenever user input is concatenated directly into a SQL query without sanitisation or parameterisation. The consequences range from authentication bypass all the way to full database compromise and server-side file writes as demonstrated here.

**Mitigation:** Use parameterised queries (prepared statements). Never concatenate raw user input into SQL strings. Implement input validation and least-privilege database accounts.

---

## Task 6 — Port Scanning with Nmap

### What I did
Used Nmap with aggressive scan flags (`-A -p-`) to enumerate all open ports and running services on the lab server (`10.0.2.6`).

### Results

| Port | Protocol | Service | Version |
|---|---|---|---|
| 21 | TCP | FTP | vsftpd 3.0.2 |
| 22 | TCP | SSH | OpenSSH (Protocol 2.0) |
| 23 | TCP | Telnet | Linux telnetd |
| 53 | TCP | DNS | BIND 9.9.5 |
| 80 | TCP | HTTP | Apache httpd 2.4 |

### Why this matters
Port scanning gives an attacker a complete map of the attack surface. In this case, the server is running Telnet on port 23, which transmits all data including credentials in plaintext — a serious finding in any real environment. FTP on port 21 has the same problem. A penetration tester would flag both as critical recommendations to disable or replace with secure alternatives (SSH instead of Telnet, SFTP instead of FTP).

---

## Key Takeaways

This lab reinforced that many of the most effective attacks rely not on sophisticated exploits but on fundamental misconfigurations: unencrypted protocols, unsanitised user input, and weak passwords. As a defender, understanding exactly how these attacks work at a technical level is essential to building effective controls against them.
