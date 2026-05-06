# Headscale + GL.iNet MT3000 Mesh Network Setup

A guide to setting up a self-hosted Tailscale coordination server (headscale) with a GL.iNet MT3000 travel router configured to join the mesh via a physical toggle switch.

---

## Architecture Overview

```
Physical switch
    → Kernel GPIO event
        → Hotplug daemon
            → /etc/rc.button/switch
                → reads UCI config (func=tailscale)
                    → /etc/gl-switch.d/tailscale.sh on/off
                        → tailscale up / tailscale down
```

```
Internet
    → headscale.example.com (DNS A record → server public IP)
        → nginx (port 80/443, TLS termination)
            → headscale (localhost:8080, coordination server)
                → SQLite (node registry)

Mesh nodes (WireGuard peer to peer):
    GL-MT3000 ←→ linux-node ←→ android-node ←→ any other node
```

---

## Prerequisites

- A Linux server with a public static IP (or a machine behind a static IP with port forwarding)
- A domain name with DNS management access
- Ports forwarded to the server:
  - TCP 80 (Let's Encrypt ACME challenge)
  - TCP 443 (HTTPS)
  - UDP 41641 (WireGuard direct connections)

---

## Part 1 — Server Setup (headscale host)

### 1. Add DNS A Record

In your DNS provider, create an A record:

```
headscale.example.com → your server's public IP
```

Verify propagation:

```bash
dig headscale.example.com
```

### 2. Install nginx

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl enable --now nginx
```

### 3. Configure nginx

Create `/etc/nginx/sites-enabled/headscale`:

```nginx
server {
    listen 80;
    server_name headscale.example.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Test and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 4. Install certbot and get TLS certificate

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d headscale.example.com
```

certbot will automatically modify your nginx config to add HTTPS and set up auto-renewal.

### 5. Install headscale

```bash
VERSION=$(curl --silent "https://api.github.com/repos/juanfont/headscale/releases/latest" \
  | grep '"tag_name"' \
  | sed -E 's/.*"([^"]+)".*/\1/' \
  | sed 's/v//')

wget "https://github.com/juanfont/headscale/releases/download/v${VERSION}/headscale_${VERSION}_linux_amd64.deb"
sudo apt install -f ./headscale_${VERSION}_linux_amd64.deb
```

### 6. Configure headscale

Edit `/etc/headscale/config.yaml`. Key values to change:

```yaml
server_url: https://headscale.example.com
listen_addr: 127.0.0.1:8080
metrics_listen_addr: 127.0.0.1:9090
```

### 7. Start headscale

```bash
sudo systemctl enable --now headscale
sudo systemctl status headscale
```

Verify the health endpoint:

```
https://headscale.example.com/health
```

Should return `{"status":"pass"}`.

### 8. Create a user

```bash
sudo headscale users create <username>
sudo headscale users list  # note the user ID
```

---

## Part 2 — Registering Nodes

### Generate a preauthkey

```bash
sudo headscale preauthkeys create --reusable --expiration 24h --user <user-id>
```

### Register a Linux node

```bash
sudo tailscale up --login-server https://headscale.example.com --authkey <KEY>
```

### Register an Android device

In the Tailscale app:
1. Settings → Accounts → kebab menu (three dots) → Use an alternate server
2. Enter `https://headscale.example.com`
3. Tap Reauthenticate
4. Use a preauthkey if the browser flow does not open

### Register a Windows node

In PowerShell:

```powershell
tailscale up --login-server https://headscale.example.com --authkey <KEY>
```

### Verify nodes

```bash
sudo headscale nodes list
```

### Rename a node

```bash
sudo headscale nodes rename --identifier <ID> <new-name>
```

---

## Part 3 — GL.iNet MT3000 Setup

### Files to create

| File | Purpose |
|------|---------|
| `/etc/gl-switch.d/tailscale.sh` | Script that runs tailscale up/down when switch is toggled |

### Files to modify

| File | Purpose | Change |
|------|---------|--------|
| `/etc/config/switch-button` | UCI config mapping switch to a function | Set `func=tailscale` |
| `/etc/rc.local` | Runs on every boot | Add UCI commands to survive UI overwrites |

---

### 1. SSH into the router

```bash
ssh root@192.168.8.1
```

### 2. Register the router as a headscale node

Generate a preauthkey on the headscale server, then on the router:

```bash
tailscale up --login-server https://headscale.example.com --authkey <KEY> --reset
```

Verify on the headscale server:

```bash
sudo headscale nodes list
```

### 3. Create the switch script

Create `/etc/gl-switch.d/tailscale.sh`:

```sh
#!/bin/sh
. /lib/functions/gl_util.sh

action=$1

if [ "$action" = "on" ]; then
    tailscale up --login-server https://headscale.example.com --reset --advertise-routes=192.168.8.0/24
    tries=0
    while [ $tries -lt 10 ]; do
        sleep 3
        if tailscale status | grep -q "100\.64\."; then
            logger -t "gl-tailscale" "connected successfully"
            exit 0
        fi
        tries=$((tries + 1))
    done
    logger -t "gl-tailscale" "failed to connect after 30 seconds"
fi

if [ "$action" = "off" ]; then
    tailscale down
    logger -t "gl-tailscale" "disconnected"
fi

sleep 5
```

Make it executable:

```bash
chmod +x /etc/gl-switch.d/tailscale.sh
```

### 4. Set UCI config

```bash
uci set switch-button.@main[0].func=tailscale
uci commit switch-button
```

Verify:

```bash
uci get switch-button.@main[0].func
# should return: tailscale
```

### 5. Persist UCI config across reboots

Edit `/etc/rc.local` and add before `exit 0`:

```sh
uci set switch-button.@main[0].func=tailscale
uci commit switch-button
```

Final `/etc/rc.local` should look like:

```sh
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.
. /lib/functions/gl_util.sh
remount_ubifs
uci set switch-button.@main[0].func=tailscale
uci commit switch-button
exit 0
```

---

## Part 4 — Subnet Routing

Subnet routing makes all devices behind the GL.iNet router reachable from anywhere on the mesh without installing Tailscale on those devices.

The `--advertise-routes` flag is already included in the switch script above. After toggling the switch on, approve the route on the headscale server:

```bash
sudo headscale nodes list-routes
sudo headscale nodes approve-routes --identifier <node-id> --routes 192.168.8.0/24
```

Verify:

```bash
sudo headscale nodes list-routes
# Approved and Serving columns should show 192.168.8.0/24
```

---

## Testing

### Test the switch script manually

```bash
# Test on
sh /etc/gl-switch.d/tailscale.sh on

# Check logs
logread | grep gl-tailscale

# Test off
sh /etc/gl-switch.d/tailscale.sh off
```

### Test subnet routing

From a device on LTE (not connected to the GL.iNet router):

```bash
ping 192.168.8.1        # router itself
ping 192.168.8.x        # any device behind the router
```

### Check node status

```bash
sudo headscale nodes list
tailscale status
```

---

## How the Switch Chain Works

When the physical switch is toggled:

1. **Hardware** — GPIO pin change detected by the kernel
2. **Kernel** — fires a hotplug event: `BUTTON=switch ACTION=pressed/released`
3. **Hotplug daemon** — matches `BUTTON=switch`, executes `/etc/rc.button/switch`
4. **`/etc/rc.button/switch`** — reads UCI config to find `func=tailscale`
5. **`/etc/gl-switch.d/tailscale.sh`** — called with `on` or `off` argument
6. **Tailscale** — connects to or disconnects from the headscale mesh

The UCI config is the only piece connecting the generic switch mechanism to the Tailscale script. The GL.iNet web UI can overwrite this value, which is why the UCI commands are added to `/etc/rc.local` — they restore the correct value on every boot.

---

## Important Notes

- **Subnet collision** — if multiple routers are on the mesh, each must use a different LAN subnet (e.g. `192.168.8.0/24`, `192.168.9.0/24`) to avoid routing conflicts
- **Preauthkey expiry** — keys expire after 24h by default but registered nodes stay connected indefinitely
- **Headscale going down** — existing WireGuard connections between nodes stay up; nodes cache their peer list locally and can reconnect to known peers on reboot even if headscale is temporarily unreachable
- **UI overwrites** — changing the switch function in the GL.iNet web UI will overwrite the UCI config; the rc.local fix restores it on next reboot
- **DERP relays** — headscale uses Tailscale's DERP relay servers as fallback when direct WireGuard connections cannot be established; traffic through DERP is still end-to-end encrypted and Tailscale cannot read it

---

## Useful Commands

```bash
# List all nodes
sudo headscale nodes list

# List routes
sudo headscale nodes list-routes

# Approve a route
sudo headscale nodes approve-routes --identifier <ID> --routes <CIDR>

# Rename a node
sudo headscale nodes rename --identifier <ID> <new-name>

# Create preauthkey
sudo headscale preauthkeys create --reusable --expiration 24h --user <user-id>

# Check tailscale status on any node
tailscale status

# Check router switch logs
logread | grep gl-tailscale

# Check UCI switch config
uci get switch-button.@main[0].func

# Manually test switch script
sh /etc/gl-switch.d/tailscale.sh on
sh /etc/gl-switch.d/tailscale.sh off
```
