# 🧦 Simple Dante SOCKS5 Proxy Installer

A **paste-and-run** Bash installer that sets up a **Dante SOCKS5 proxy** with username/password authentication.  
It automatically opens the firewall, enables the service on boot, and works out of the box on **Ubuntu 22.04 / 24.04**.

---

## 🚀 Quick Install

Copy and paste this command into your **Ubuntu VPS**:

```bash
cat <<'EOF' > ~/install-dante.sh
#!/usr/bin/env bash
set -euo pipefail

echo "=== Dante SOCKS5 installer for Ubuntu 22.04/24.04 IPv4 ==="

if [[ $EUID -ne 0 ]]; then
  echo "Run with sudo:"
  echo "sudo bash ~/install-dante.sh"
  exit 1
fi

read -rp "Proxy username [proxyuser]: " PROXY_USER
PROXY_USER=${PROXY_USER:-proxyuser}

while true; do
  read -rsp "Proxy password: " PROXY_PASS; echo
  read -rsp "Confirm password: " PROXY_PASS2; echo

  if [[ -n "$PROXY_PASS" && "$PROXY_PASS" == "$PROXY_PASS2" ]]; then
    break
  fi

  echo "Passwords did not match or were empty. Try again."
done

read -rp "SOCKS port [1080]: " PROXY_PORT
PROXY_PORT=${PROXY_PORT:-1080}

if ! [[ "$PROXY_PORT" =~ ^[0-9]+$ ]] || (( PROXY_PORT < 1 || PROXY_PORT > 65535 )); then
  echo "Invalid port: $PROXY_PORT"
  exit 1
fi

IFACE="$(ip -4 route ls default 2>/dev/null | awk '{print $5; exit}')"

if [[ -z "${IFACE:-}" ]]; then
  echo "Could not detect IPv4 default network interface."
  echo "Your server may not have IPv4 configured."
  exit 1
fi

SERVER_IPV4="$(curl -4 -fsS --max-time 5 https://ifconfig.me 2>/dev/null || true)"

if [[ -z "$SERVER_IPV4" ]]; then
  echo "Could not detect public IPv4."
  echo "Check if your server has IPv4:"
  echo "ip -4 addr"
  exit 1
fi

echo "Using IPv4 interface: $IFACE"
echo "Detected public IPv4: $SERVER_IPV4"

export DEBIAN_FRONTEND=noninteractive

apt-get update -y
apt-get install -y dante-server ufw curl ca-certificates

if id -u "$PROXY_USER" >/dev/null 2>&1; then
  echo "User '$PROXY_USER' already exists; updating password."
else
  useradd --system --no-create-home --shell /usr/sbin/nologin "$PROXY_USER"
fi

echo "${PROXY_USER}:${PROXY_PASS}" | chpasswd
unset PROXY_PASS PROXY_PASS2

if ! id -u dantedrun >/dev/null 2>&1; then
  useradd --system --no-create-home --shell /usr/sbin/nologin dantedrun
fi

if [[ -f /etc/danted.conf ]]; then
  cp -a /etc/danted.conf "/etc/danted.conf.bak.$(date +%Y%m%d%H%M%S)"
fi

cat >/etc/danted.conf <<CFG
logoutput: syslog

internal: 0.0.0.0 port = ${PROXY_PORT}
external: ${IFACE}

socksmethod: username

user.privileged: root
user.unprivileged: dantedrun
user.libwrap: dantedrun

client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect disconnect error
}

client block {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect error
}

socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    command: connect
    protocol: tcp
    proxyprotocol: socks_v5
    socksmethod: username
    user: ${PROXY_USER}
    log: connect disconnect error
}

socks block {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect error
}
CFG

danted -V -f /etc/danted.conf

ufw allow OpenSSH >/dev/null 2>&1 || true
ufw allow "${PROXY_PORT}/tcp" comment "Dante SOCKS5" >/dev/null

if ufw status | grep -qi inactive; then
  ufw --force enable
else
  ufw reload
fi

systemctl daemon-reload
systemctl enable danted >/dev/null
systemctl restart danted

sleep 1

if ! systemctl is-active --quiet danted; then
  echo "Dante failed to start."
  systemctl status danted --no-pager --full
  exit 1
fi

echo
echo "=== Done. SOCKS5 proxy is ready on IPv4. ==="
echo "SOCKS5 host: $SERVER_IPV4"
echo "SOCKS5 port: $PROXY_PORT"
echo "Username:    $PROXY_USER"
echo "Password:    the password you entered"
echo
echo "Test from your computer:"
echo "curl -x socks5h://${PROXY_USER}:YOUR_PASSWORD@${SERVER_IPV4}:${PROXY_PORT} https://ipinfo.io/ip"
EOF

sudo bash ~/install-dante.sh
```

---

## 🧩 Features

- 🔒 Username/password authentication (no open proxy risk)  
- 🔥 UFW firewall configuration  
- 🔁 Autostart on boot (`systemd`)  
- 🧠 Automatic network interface detection  
- 💨 Works out of the box with browsers, `curl`, or any SOCKS5-compatible client  

---

## 🧪 Example Test Command

```bash
curl -x socks5h://proxyuser:yourpassword@<SERVER_IP>:1080 https://ipinfo.io
```

---

## 🛠 Managing the Service

| Command | Description |
|----------|-------------|
| `sudo systemctl restart danted` | Restart the proxy |
| `sudo systemctl status danted` | Check proxy status |
| `sudo systemctl disable danted` | Disable on boot |
| `sudo systemctl enable danted` | Enable on boot |

---

## ⚠️ Security Notes

- Do **not** expose port 1080 publicly without authentication.  
- Use strong passwords and restrict access by IP if possible.  
- Regularly update your packages:
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
- For extra stealth, run on a non-standard port (e.g. 51820).

---

## 🧾 License

**MIT License © 2025**  
Feel free to fork, modify, and share.
