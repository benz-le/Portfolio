
# 🔐 Pi-hole v6 HTTPS with Private CA (Green Lock, No Public Exposure)

## Overview

This project documents how I securely enabled **HTTPS for the Pi-hole v6 admin interface** using a **private Certificate Authority (CA)** instead of Let’s Encrypt.

### Why this matters
- Pi-hole is an **internal infrastructure service**
- The admin UI should **never be publicly exposed**
- HTTPS is still important to prevent credential leakage and MITM attacks
- Let’s Encrypt is **not appropriate** for VPN/LAN-only services

This setup provides:
- ✅ Green lock (trusted HTTPS)
- ✅ No public DNS or port forwarding
- ✅ Works on LAN and WireGuard
- ✅ Fully compliant with modern browser requirements

---

## Architecture

```
Browser
  ↓ trusts
Private Local CA
  ↓ signs
pihole.benz.lan (server certificate)
  ↓ served by
Pi-hole v6 (civetweb web server)
```

---

## Key Technical Lessons (TL;DR)

- Pi-hole v6 **does NOT use** `pihole-FTL.conf` for TLS
- TLS configuration is managed via **`/etc/pihole/pihole.toml`**
- Pi-hole v6 requires a **single PEM file** (key + cert)
- Pi-hole will silently fall back to its built-in `pi.hole` certificate if misconfigured
- Browsers trust **CAs**, not individual server certificates

---

## File Layout

```
/etc/pihole/
├── pihole.toml
├── tls.pem
└── tls/
    ├── pihole.key
    └── pihole.crt

~/pihole-ca/
├── ca.key
├── ca.crt
├── pihole.key
├── pihole.crt
└── pihole-leaf.cnf
```

---

## Step 1 – Create a Private Certificate Authority

```bash
mkdir ~/pihole-ca
cd ~/pihole-ca
```

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -days 3650 -out ca.crt
```

---

## Step 2 – Create the Pi-hole Server Certificate

```bash
openssl genrsa -out pihole.key 2048
openssl req -new -key pihole.key -out pihole.csr
openssl x509 -req -in pihole.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out pihole.crt -days 825
```

---

## Step 3 – Install Certificates on Pi-hole

```bash
sudo mkdir -p /etc/pihole/tls
sudo cp pihole.key pihole.crt /etc/pihole/tls/
```

Create PEM:

```bash
sudo bash -c 'cat /etc/pihole/tls/pihole.key /etc/pihole/tls/pihole.crt > /etc/pihole/tls.pem'
sudo chown pihole:pihole /etc/pihole/tls.pem
sudo chmod 600 /etc/pihole/tls.pem
```

---

## Step 4 – Configure Pi-hole v6 TLS

```toml
[webserver]
port = 443

[webserver.api.tls]
cert = "/etc/pihole/tls.pem"
```

```bash
sudo systemctl restart pihole-FTL
```

---

## Step 5 – Verify Certificate

```bash
openssl s_client -connect localhost:443 -servername pihole.benz.lan </dev/null | openssl x509 -noout -issuer -subject
```

---

## Step 6 – Trust the CA on Client Devices

Import `ca.crt` into **Trusted Root Certification Authorities**.

---

## Final Result

```
https://pihole.benz.lan/admin
```

✔ Green lock  
✔ Private CA  
✔ No public exposure  

---

## Security Notes

- Never expose Pi-hole publicly
- Never copy `ca.key` off the server
- Always verify with OpenSSL
