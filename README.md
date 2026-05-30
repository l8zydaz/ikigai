# 🔒 Hardened Self-Hosted RSS Intelligence Pipeline

I've been asked a lot of times by friends, colleagues, and family about how I keep up to date with emerging technologies and events occurring in cybersecurity. Here are the answers they've — and now you, yes you reading this — been seeking.

Before I formalized my RSS feeds, I ran mobile feeds on Pixels running GrapheneOS by utilizing Feeder. Recently I decided to take it a step further with self-hosting. Inside of FreshRSS, there's an option to add your email which you can set up SMTP to pull these feeds to your phone directly — or go the other route of running them through your terminal.

I'll go a little in-depth with how I not only set up a self-hosted RSS aggregator but how I hardened it to ensure I'm receiving my news in a secure fashion.

Thanks for taking a moment to read. Enjoy!

---

---

## What You're Building

A local web server that:
- Aggregates 100+ curated RSS feeds across security, academia, threat intel, and culture
- Runs over HTTPS on your own machine — never exposed to the internet
- Is accessible from your browser and your terminal
- Stores nothing off-device

When you're done you'll have a personal intelligence platform that updates itself and stays completely under your control.

---

## What You Need

- A Debian-based Linux machine (this was built and tested on Debian 13 Trixie)
- Basic comfort with a terminal — you don't need to be an expert, just willing
- About 30-45 minutes

That's it. Everything else gets installed along the way.

---

## Stack

| Component | What it does |
|-----------|-------------|
| FreshRSS | The RSS reader — browser interface |
| Apache 2.4 | Local web server |
| OpenSSL | Handles the HTTPS certificate |
| UFW | Firewall — keeps it localhost only |
| Newsboat | Optional terminal RSS reader |
| Debian 13 (Trixie) | OS — Linux 6.12 kernel |

---

## Step 1 — Install Dependencies

```bash
sudo apt install apache2 php php-curl php-gd php-mbstring php-xml php-zip php-sqlite3 git ufw
```

This installs Apache (your web server), PHP (FreshRSS needs it to run), and a few utilities. Nothing unusual here.

---

## Step 2 — Install FreshRSS

```bash
cd /var/www/html
sudo git clone https://github.com/FreshRSS/FreshRSS.git
sudo chown -R www-data:www-data FreshRSS/
sudo chmod -R 755 FreshRSS/
```

This pulls FreshRSS into your web server directory and gives Apache permission to run it. The `chown` and `chmod` commands set the right file ownership and permissions — without them Apache can't read the files.

---

## Step 3 — Enable Required Apache Modules

```bash
sudo a2enmod ssl headers rewrite php8.4
sudo systemctl restart apache2
```

Apache is modular — features are off by default and you turn on what you need. Here you're enabling SSL (for HTTPS), headers (for security headers), rewrite (for the HTTP to HTTPS redirect), and PHP support. Restart Apache after so the changes take effect.

---

## Step 4 — Generate Your Certificate

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/freshrss.key \
-out /etc/ssl/certs/freshrss.crt \
-subj "/CN=localhost"
```

This creates a self-signed SSL certificate so your connection is encrypted. Since this is localhost only, you don't need a certificate from a real CA — you're the only one connecting. Fill in whatever you want for the prompts, just make sure Common Name is `localhost`.

Your browser will warn you about the self-signed cert — that's expected. Add an exception and move on.

---

## Step 5 — Configure Apache

Create your VirtualHost config at `/etc/apache2/sites-available/freshrss-ssl.conf`. The key things it needs to do:

- Serve FreshRSS over HTTPS on port 443, bound to `127.0.0.1` only
- Point to your SSL certificate and key
- Redirect any HTTP traffic on port 80 to HTTPS automatically

Refer to the Security Hardening section below for the exact directives to include.

Enable your new config and disable the default:
```bash
sudo a2ensite freshrss-ssl.conf
sudo a2dissite 000-default.conf
sudo systemctl restart apache2
```

Update `/etc/apache2/ports.conf` so Apache only listens on localhost — not your network:
```apache
Listen 127.0.0.1:80
Listen 127.0.0.1:443
```

---

## Step 6 — Apply Security Headers

In `/etc/apache2/conf-available/security.conf`, apply the hardening directives from the Security Hardening section below. Then enable it:

```bash
sudo a2enconf security
sudo systemctl restart apache2
```

---

## Step 7 — Set Up Your Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw deny 80
sudo ufw deny 443
sudo ufw allow from 127.0.0.1 to any port 80
sudo ufw allow from 127.0.0.1 to any port 443
sudo ufw enable
```

This is the belt-and-suspenders approach. Apache is already bound to localhost — UFW is a second layer that makes sure ports 80 and 443 are unreachable from your network regardless. Only your own machine can talk to it.

---

## Step 8 — Access FreshRSS

Navigate to `https://127.0.0.1/FreshRSS/p/` in your browser. Accept the self-signed cert warning and run through the setup wizard. Create your admin account and you're in.

---

## Step 9 — Import Your Feeds

FreshRSS > Subscription Management > Import/Export > Import OPML > select `feeds.opml`

100+ sources across security, research, and culture — organized into categories and ready to go. Delete whatever doesn't fit your interests and add what does.

---

## Security Hardening Applied

This is what separates this setup from just running FreshRSS on a default Apache install.

### Transport Security
- TLS 1.2 and 1.3 only — SSLv2, SSLv3, TLS 1.0, TLS 1.1 all disabled
- AES-256 cipher suites only (ECDHE-ECDSA-AES256-GCM-SHA384, ECDHE-RSA-AES256-GCM-SHA384, ChaCha20-Poly1305)
- `SSLHonorCipherOrder on` — server picks the cipher, not the client. Prevents downgrade attacks.
- `SSLSessionTickets off` — prevents session resumption attacks
- HTTP permanently redirects to HTTPS — you can't accidentally connect unencrypted

### Network Exposure
- Apache bound to `127.0.0.1` — not your network interface, not the internet. Localhost only.
- UFW blocks ports 80 and 443 from external access as a second layer
- Zero external attack surface

### Information Disclosure
- `ServerTokens Prod` — Apache won't tell anyone what version it's running
- `ServerSignature Off` — no server info on error pages
- `TraceEnable Off` — disables HTTP TRACE, used in cross-site tracing attacks
- Git/SVN directories return 404 — no accidental source code exposure

### Security Headers
- `X-Frame-Options: SAMEORIGIN` — prevents your pages being embedded in iframes on other sites (clickjacking)
- `X-XSS-Protection: 1; mode=block` — browser-level XSS protection
- `X-Content-Type-Options: nosniff` — stops browsers from guessing MIME types
- `Referrer-Policy: strict-origin-when-cross-origin` — controls what gets sent in the Referer header
- FreshRSS ships with its own Content Security Policy — left intact rather than overriding it

---

## Verify It's Working

Run these after setup to confirm everything is locked down:

```bash
# Check TLS version and cipher actually in use
openssl s_client -connect 127.0.0.1:443 2>&1 | grep -E "Protocol|Cipher"

# Confirm Apache is only listening on localhost
sudo ss -tlnp | grep apache

# Check hardened modules are loaded
sudo apache2ctl -M | grep -E "ssl|headers|rewrite"

# Review firewall rules
sudo ufw status verbose
```

---

## Feed Categories

`feeds.opml` includes 100+ sources organized into:

- **Cybersecurity & Infosec** — The Hacker News, Bleeping Computer, Krebs on Security, Talos, CrowdStrike, SentinelOne, PortSwigger, Trail of Bits, and more
- **Vulnerability Tracking** — NVD CVE Feed, CISA Alerts, CERT/CC, Debian/Ubuntu/Red Hat security advisories
- **Threat Intelligence** — Mandiant, The DFIR Report, VirusTotal, MalwareBazaar, URLhaus, Abuse.ch
- **Privacy & OPSEC** — EFF, Privacy Guides, Access Now
- **CTF & Learning** — CTFtime, VulnHub, Hack The Box
- **Academic Research** — arXiv (cs.CR, cs.AI, cs.LG), Nature, Science, Quanta Magazine, PNAS, USENIX
- **Philosophy & Deep Thought** — Aeon, The Paris Review, NY Review of Books, Lapham's Quarterly
- **Dark Academia** — The Marginalian, JSTOR Daily, Public Domain Review, Atlas Obscura, Literary Hub
- **Cognitive Science** — Neuroscience News, Frontiers in Psychology, APA
- **Mathematics** — arXiv math.LO, Quanta Mathematics, Plus Maths

---

## Why Self-Hosted?

Commercial RSS services like Feedly or Inoreader know exactly what you're reading, when you read it, and how often. That data gets sold. It can be subpoenaed. It can be breached.

A self-hosted instance on localhost means your reading habits never leave your machine. No third party knows you're tracking CVEs at 2am or deep-diving into philosophy papers on a Tuesday. Your research is your own business.

For security researchers and academics this isn't paranoia — it's just good practice.

---

## Files

```
.
├── README.md
└── feeds.opml    # 100+ curated RSS feeds, ready to import
```

---

## License

MIT — take it, modify it, make it yours.

---

*Built on Debian 13 Trixie · Apache 2.4 · FreshRSS · OpenSSL 3.5*
