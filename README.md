[internet_radio_project.md](https://github.com/user-attachments/files/34339332/internet_radio_project.md)
# Building a 24/7 Internet Radio Station with Ubuntu, SHOUTcast, Nginx, HTTPS and Mixxx

*A practical project report from wirrtuel.ch — November 2026, Switzerland*

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Architecture](#2-system-architecture)
3. [Installing Ubuntu Server](#3-installing-ubuntu-server)
4. [Installing SHOUTcast DNAS](#4-installing-shoutcast-dnas)
5. [Running SHOUTcast as a systemd Service](#5-running-shoutcast-as-a-systemd-service)
6. [Introducing Nginx as a Reverse Proxy](#6-introducing-nginx-as-a-reverse-proxy)
7. [Enabling HTTPS with Let's Encrypt](#7-enabling-https-with-lets-encrypt)
8. [Configuring Mixxx](#8-configuring-mixxx)
9. [Testing and Verification](#9-testing-and-verification)
10. [Backup Strategy](#10-backup-strategy)
11. [Lessons Learned](#11-lessons-learned)
12. [Acknowledgements](#12-acknowledgements)

---

## 1. Introduction

The goal was straightforward: build a reliable, privately operated Internet radio station capable of broadcasting music around the clock.

Our existing setup used Mixxx on a Windows computer and a SHOUTcast server running inside an Ubuntu 20 virtual machine. We wanted to modernize the infrastructure, introduce a new Ubuntu server, enable secure HTTPS streaming and migrate without unnecessarily interrupting the existing broadcast.

The result is a working system with two SHOUTcast servers, a secure public streaming endpoint and a Windows-based Mixxx AutoDJ installation.

This article documents the architecture, installation, configuration, verification procedures and lessons learned.

## 2. System Architecture

The hardware consists of a Dell Windows 11 computer and Oracle VirtualBox.

| Component | Configuration |
|---|---|
| Host | Dell computer |
| Host operating system | Windows 11 |
| Virtualization | Oracle VirtualBox |
| Guest operating system | Ubuntu 26 |
| Network | Bridged adapter |
| Streaming server | SHOUTcast DNAS 2.6.1.777 |
| Web server | Nginx |
| TLS certificates | Let's Encrypt / Certbot |
| Broadcast software | Mixxx 2.5.4 |
| Public hostname | radio.wirrtuel.ch |

### Signal flow

```text
QNAP NAS (music collection)
          |
          v
Windows 11 / Mixxx / AutoDJ
          |
          +----> Old Ubuntu 20 VM
          |      SHOUTcast :8000
          |
          +----> New Ubuntu 26 VM
                 SHOUTcast :8002
                       |
                       v
                 Nginx reverse proxy
                       |
                       v
                  HTTPS :443
                       |
                       v
               Internet radio listeners
```

The new Ubuntu VM uses the local address `192.168.178.53`. The public streaming endpoint is:

`https://radio.wirrtuel.ch/index.html?sid=1`

## 3. Installing Ubuntu Server

We created a new VirtualBox virtual machine with bridged networking so the Ubuntu server could be reached directly from other devices on the local network.

After installation, we verified the operating system and network configuration:

```bash
lsb_release -a
hostnamectl
ip -br addr
```

We verified SSH access:

```bash
ssh ******@192.168.178.53
```

For routine maintenance:

```bash
sudo apt update
sudo apt upgrade
```

These are standard administrative commands, not necessarily a verbatim historical transcript.

## 4. Installing SHOUTcast DNAS

We migrated SHOUTcast DNAS version 2.6.1.777 from the existing environment.

An illustrative server layout is:

```text
/home/radio/server/
    sc_serv
    sc_serv.conf
```

Verify executable dependencies, adjusting the path to match the installation:

```bash
ldd /home/radio/server/sc_serv
```

The new SHOUTcast service listens on port **8002**. Keep administrator and source passwords out of public documentation.

## 5. Running SHOUTcast as a systemd Service

Instead of manually launching SHOUTcast after every restart, we configured a systemd service.

An **illustrative** `/etc/systemd/system/shoutcast.service` (adjust paths and user to the real deployment):

```ini
[Unit]
Description=SHOUTcast DNAS Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=radio
WorkingDirectory=/home/radio/server
ExecStart=/home/radio/server/sc_serv /home/radio/server/sc_serv.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable shoutcast.service
sudo systemctl start shoutcast.service
```

Verify its status and inspect recent logs:

```bash
sudo systemctl status shoutcast.service
sudo journalctl -u shoutcast.service -n 50
```

## 6. Introducing Nginx as a Reverse Proxy

SHOUTcast was initially available over HTTP on port 8002. Nginx provides the public HTTPS entry point:

```text
HTTPS client -> Nginx :443 -> SHOUTcast 127.0.0.1:8002
```

The following is a **simplified example**, not a verbatim production configuration:

```nginx
server {
    listen 80;
    server_name radio.wirrtuel.ch;

    location / {
        proxy_pass http://127.0.0.1:8002;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 3600s;
    }
}
```

Check and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Disabling proxy buffering is useful for continuous audio streaming.

## 7. Enabling HTTPS with Let's Encrypt

We configured a TLS certificate for `radio.wirrtuel.ch` using Certbot.

Check the installed certificate, test renewal and inspect the timer:

```bash
sudo certbot certificates
sudo certbot renew --dry-run
systemctl status certbot.timer
```

The renewal dry run completed successfully, and the renewal timer was enabled. Our FRITZ!Box forwards public TCP port **443** to the new Ubuntu VM.

## 8. Configuring Mixxx

Mixxx 2.5.4 runs on Windows and manages the music library and AutoDJ playback. The music collection is stored on a QNAP NAS.

During migration, two SHOUTcast broadcast connections were configured:

- **Existing server:** Ubuntu 20, SHOUTcast port 8000.
- **New server:** Ubuntu 26, SHOUTcast port 8002.

Both servers could receive the same program from Mixxx. An initially incorrect source password prevented the new connection from authenticating; correcting the credentials fixed it.

Mixxx restricts editing broadcast connection settings while broadcasting is active.

## 9. Testing and Verification

Check network configuration:

```bash
ip -br addr
```

Check SHOUTcast service and listening ports:

```bash
sudo systemctl status shoutcast.service
sudo ss -ltnp
```

Check Nginx and HTTPS:

```bash
sudo nginx -t
curl -I https://radio.wirrtuel.ch/
```

Check certificate renewal:

```bash
sudo certbot renew --dry-run
```

Finally, play the stream from another computer. A successful HTTP response alone does not guarantee that audio is being delivered correctly.

## 10. Backup Strategy

Before further changes, we shut down both Ubuntu virtual machines cleanly and copied their complete VirtualBox directories to a separate backup drive.

We also backed up the Mixxx configuration and SQLite database. The database contains music library entries, playlists, crates, track metadata, cuepoints, file locations and playback statistics.

For a consistent SQLite database backup, close Mixxx before copying its database files.

## 11. Lessons Learned

- **Keep the old system running during migration.** Parallel servers allow safer testing.
- **Separate streaming from TLS termination.** Nginx provides a clean HTTPS endpoint.
- **Use systemd for unattended operation.** It helps services restart after failures and boot automatically.
- **Verify the full signal chain.** Test encoder, streaming server, proxy, certificate and actual playback.
- **Back up before changing working infrastructure.** Complete VM and Mixxx backups offer recovery options.
- **Never publish secrets.** Remove passwords, tokens and sensitive configuration values.

## 12. Acknowledgements

This project was developed through many interactive troubleshooting sessions with **Cleverina**, our affectionate nickname for OpenAI's ChatGPT.

Cleverina helped with Linux administration, SHOUTcast configuration, systemd services, Nginx, HTTPS, diagnostic commands and troubleshooting.

The collaboration combined practical system administration experience with AI-assisted problem solving. There were occasional surprises, a few stubborn configuration problems and plenty of laughter.

Our unofficial mascot became a small penguin, tirelessly carrying configuration files between Windows, Ubuntu and the Internet.

The important engineering decisions, testing and operation remained under human control.

**The result: a working Internet radio station, music around the clock, and a few very happy penguins.**

*Project: wirrtuel.ch — Switzerland — November 2026*

---

*Publication note: Example paths and configurations must be checked against the actual deployed files before this guide is used as a reproducible installation procedure.*
