# Getting Started

VPN Management Studio is a self-hosted platform for managing WireGuard and AmneziaWG VPN servers. It provides a web admin panel, a client self-service portal, Telegram bots, and a remote agent system — all in a single deployable package.

---

## What Is VPN Management Studio

You install it on a Linux server. It takes over the WireGuard configuration on that server and any remote servers you add. You manage everything from a single web interface: add clients, issue configs, monitor traffic, apply bandwidth limits, and automate subscription billing.

It is designed for:
- VPN resellers and operators running fleets of WireGuard servers
- Teams that need controlled, audited access to VPN config management
- Service providers offering VPN subscriptions with a self-service client portal

---

## Key Features

| Feature | Description |
|---------|-------------|
| Multi-server management | Manage unlimited remote WireGuard / AmneziaWG servers from one panel |
| Remote agent | Lightweight HTTP agent replaces SSH for day-to-day operations |
| Client portal | End-user web UI for self-registration, payment, and config download |
| Telegram bots | Admin bot for operations, client bot for self-service |
| Traffic & bandwidth | Per-client traffic counters, limits, and `tc`-based bandwidth shaping |
| Subscriptions | Plan-based billing with CryptoPay integration |
| Automatic updates | Signed update packages with rollback support |
| Backup and restore | Scheduled database + config backups with one-command restore |
| Drift detection | Automatic comparison of DB state vs live WireGuard interface with auto-reconcile |
| AmneziaWG support | Full support for obfuscated WireGuard with obfuscation parameters |
| White-label | Custom branding: name, logo, colors, domain |

---

## WireGuard vs AmneziaWG

VPN Management Studio supports both protocols transparently.

**WireGuard** is the standard protocol. It is fast, widely supported, and works out of the box on most clients. Use it by default.

**AmneziaWG** is an obfuscated variant of WireGuard that hides the VPN traffic signature. It is useful in environments where standard WireGuard is blocked by DPI firewalls. The server requires the `amneziawg-dkms` kernel module. Clients must use the AmneziaVPN app (available for Android, iOS, Windows, macOS).

When you add a server, you select the type: `wireguard` or `amneziawg`. All subsequent operations — peer management, config generation, health checks — handle the protocol differences automatically.

---

## Quick Start

**1. Install on a fresh server:**

```bash
tar xzf vpn-manager-v1.1.0.tar.gz
cd vpn-manager-v1.1.0
sudo bash install.sh
```

**2. Open the admin panel:**

```
http://YOUR_SERVER_IP:10086
```

Create your admin account on first visit.

**3. Activate your license:**

Go to **Settings → License**, paste your activation code, click **Activate**.

Trial mode allows 10 clients on 1 server for 7 days.

**4. Add your first client:**

Go to **Clients → Add Client**, enter a name, select the server, click **Create**.

Download the `.conf` file or scan the QR code with the WireGuard mobile app.

---

## What Happens Next

- [Installation details →](installation.md)
- [Adding remote servers →](add-server.md)
- [Managing clients →](client-management.md)
- [Update mechanism →](updates.md)
- [Full architecture overview →](architecture.md)
