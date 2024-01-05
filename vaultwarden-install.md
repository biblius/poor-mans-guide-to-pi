# Vaultwarden Install

This document provides scripts that can be utilised to easily set up a [Vaultwarden](https://github.com/dani-garcia/vaultwarden) instance directly on a linux machine (without docker).
Additionally, it provides a guide for building and deploying a Vaultwarden instance. This document will primarily focus on the `aarch64` architecture, but the same principles apply for any.

This guide assumes you are using linux for all steps.

It consists of 4 (potentially) simple steps:
1. [Compiling the binary](#compiling-the-binary)
2. [File infrastructure](#infrastructure)
3. [Web Vault](#infrastructure)

## Compiling the binary

We'll be compiling the binary on our main machine, for which we will need [rust. 🦀](https://www.rust-lang.org/tools/install)

Since we are focusing on `aarch64`, we have to add it to `rustup`'s target list as well as make sure we have the necessary compilation plumbing for it.

The first command adds the target, the second one installs dependencies needed to cross compile

```bash
rustup target add aarch64-unknown-linux-gnu
sudo apt install gcc-aarch64-linux-gnu
```

After this, there should be a new directory at `/usr/lib/aarch64-linux-gnu`. This directory contains the necessary libraries
for the target architecture and we have to tell cargo to use it. We do this by pasting the following in `~/.cargo/config.toml`:

```toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
rustflags = [
    "-L/usr/lib/aarch64-linux-gnu",
    "-Ctarget-feature=+crt-static",
]
```

This makes sure the compiler will use the new directory as the [sysroot](https://autotools.info/pkgconfig/cross-compiling.html) during compilation - it is the equivalent of setting `PKG_CONFIG_SYSROOT_DIR=/usr/lib/aarch64-linux-gnu`. Additionally,
we are linking everything statically, which ensures the binary does not need any dependencies when it's run on the target system.

Now we have to cross our fingers 

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
