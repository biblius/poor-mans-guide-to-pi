# Vaultwarden Install

This document will guide through building and deploying a Vaultwarden instance directly on a linux machine. 

```bash
# User config
sudo addgroup --system vaultwarden
sudo adduser --system --home /opt/vaultwarden --shell /usr/sbin/nologin --no-create-home --gecos 'vaultwarden' --ingroup vaultwarden --disabled-login --disabled-password vaultwarden

# Create directories
mkdir -p /opt/vaultwarden/bin
mkdir -p /opt/vaultwarden/data

# Permissions
sudo chown -R vaultwarden:vaultwarden /opt/vaultwarden/
sudo chown root:root /opt/vaultwarden/bin/vaultwarden
sudo chmod +x /opt/vaultwarden/bin/vaultwarden
sudo chown -R root:root /opt/vaultwarden/web-vault/
sudo chmod +r /opt/vaultwarden/.env

# Build binary
cargo build -F sqlite -F vendored_openssl --target=armv7-unknown-linux-gnueabihf --release

# Web vault
curl -fsSLO https://github.com/dani-garcia/bw_web_builds/releases/download/v2023.12.0/bw_web_v2023.12.0.tar.gz # check latest available version on https://github.com/dani-garcia/bw_web_builds/releases
sudo tar -zxf bw_web_v2023.12.0.tar.gz -C /opt/vaultwarden/
rm -f bw_web_v2023.12.0.tar.gz

# Create secure Admin Token
echo -n "MySecretPassword" | argon2 "$(openssl rand -base64 32)" -e -id -k 65540 -t 3 -p 4
```
