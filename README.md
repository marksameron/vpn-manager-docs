# VPN Manager — Documentation

Self-hosted WireGuard VPN management platform. Admin panel, client portal, Telegram bots.

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
tar xzf vpn-manager-v1.4.48.tar.gz
cd vpn-manager-v1.4.48
sudo bash install.sh
```

Open `http://YOUR_SERVER_IP:10086` after install.

---

## Support

Contact: [support@flirexa.biz](mailto:support@flirexa.biz)
