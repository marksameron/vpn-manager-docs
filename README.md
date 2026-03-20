# VPN Manager — Documentation

Self-hosted WireGuard VPN management platform. Admin panel, client portal, Telegram bots, Android app.

---

## Contents

| Document | Description |
|----------|-------------|
| [Getting Started](getting-started.md) | Overview, quick start, feature summary |
| [Installation](installation.md) | System requirements, install script, Docker, SSL |
| [Adding Servers](add-server.md) | How to connect remote WireGuard servers |
| [Client Management](client-management.md) | Creating clients, QR codes, traffic limits |
| [Agent Mode](agent-mode.md) | Lightweight HTTP agent for remote servers |
| [Backup & Restore](backup-restore.md) | Scheduled backups, restore procedure |
| [Updates](updates.md) | Update mechanism, rollback |
| [Troubleshooting](troubleshooting.md) | Common issues and fixes |
| [Architecture](architecture.md) | System architecture overview |
| [API Reference](api.md) | REST API endpoints |

---

## Quick Start

```bash
tar xzf vpn-manager-v1.2.36.tar.gz
cd vpn-manager-v1.2.36
sudo bash install.sh
```

Open `http://YOUR_SERVER_IP:10086` after install.

---

## Support

Contact: [mazarata.kirara@gmail.com](mailto:mazarata.kirara@gmail.com)
