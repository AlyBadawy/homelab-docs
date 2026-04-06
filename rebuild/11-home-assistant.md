# 11 — Home Assistant

Smart home automation hub with access through NPM. Uses host networking for device discovery. Users are managed with Home Assistant's built-in local accounts.

## Prerequisites

- Guide 06 complete — NPM running with wildcard cert
- Docker network: `proxy` present (not used directly — HA runs on host network, but NPM proxies via `127.0.0.1`)

> **Why host network mode?** Home Assistant needs direct access to mDNS and other network protocols to auto-discover smart home devices. Docker bridge networking breaks these features. Because of this, HA is accessed via `127.0.0.1:8123` from NPM, not via a Docker network hostname.

> **How to deploy a stack in Portainer:** Go to **Stacks → + Add stack**, enter the stack name shown, paste the compose content into the **Web editor**, then click **Deploy the stack**.

---

## Section 1: Deploy Home Assistant

Stack name: `homeassistant`

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    network_mode: host
    volumes:
      - homeassistant_config:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=America/New_York

volumes:
  homeassistant_config:
```

No environment variables needed beyond TZ (already in the compose).

Click **Deploy the stack**. Wait 2–3 minutes. Check **Containers → homeassistant → Logs**. Watch for: `Home Assistant initialized in`.

---

## Section 2: Complete Initial Onboarding

⚠️ Because HA uses host networking, access it **directly by IP** for the initial setup — not through NPM:

Open `http://172.20.20.5:8123` in your browser.

1. Walk through the onboarding wizard
2. Create your admin account
3. Finish onboarding and confirm you can reach the HA dashboard

Additional users can be added later under **Settings → People → Add Person**.

---

## Section 3: Configure NPM Proxy

In NPM → **Proxy Hosts → Add Proxy Host**:

- **Domain Names:** `ha.in.alybadawy.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `127.0.0.1`
- **Forward Port:** `8123`
- **SSL Certificate:** `wildcard-in-alybadawy-com`
- **Force SSL:** On
- **HTTP/2 Support:** On
- **Websockets Support:** On ← required for the HA frontend to work

---

## Section 4: Configure Trusted Proxies in Home Assistant

Because traffic now arrives through NPM, Home Assistant must be told to trust NPM as a proxy — otherwise it will reject forwarded headers and may log security warnings.

Find the Home Assistant config volume path:

```bash
docker inspect homeassistant | grep -A5 Mounts
```

The config volume is typically at `/var/lib/docker/volumes/homeassistant_homeassistant_config/_data`. Edit `configuration.yaml` inside it:

```bash
sudo nano /var/lib/docker/volumes/homeassistant_homeassistant_config/_data/configuration.yaml
```

Add the `http` block (or merge with an existing one if present):

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - 172.16.0.0/12
```

Restart Home Assistant from Portainer (**Containers → homeassistant → Restart**), then verify `https://ha.in.alybadawy.com` loads correctly.

---

## Verification Checklist

- [ ] Home Assistant running: `docker ps | grep homeassistant` shows `Up`
- [ ] Direct access works: `http://172.20.20.5:8123` loads
- [ ] Proxied access works: `https://ha.in.alybadawy.com` loads, SSL valid
- [ ] HA frontend loads fully (no WebSocket errors in browser console)
- [ ] Smart devices discoverable in HA Settings → Devices & Services

---

## Troubleshooting

**`ha.in.alybadawy.com` loads but the UI is broken / stuck loading:**

WebSockets are required. Confirm **Websockets Support** is enabled in the NPM proxy host. Without it, the HA frontend can't maintain its real-time connection.

**"400 Bad Request" from Home Assistant after adding NPM proxy:**

The `trusted_proxies` block in `configuration.yaml` is missing or the CIDR range doesn't cover NPM's address. Verify the file was saved and HA was restarted.

**Smart home devices not auto-discovering:**

This is expected in Docker. mDNS (Bonjour/Zeroconf) works with `network_mode: host` on Linux, but some devices on other VLANs (like IoT VLAN 100) will need manual IP configuration or a mDNS repeater at the router level.

**Can't edit `configuration.yaml`:**

```bash
# Find the actual path of the named volume
docker volume inspect homeassistant_homeassistant_config
# Look for "Mountpoint" in the output
```
