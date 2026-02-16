# WireGuard + SOCKS5 + Chrome Proxy

**Intent:** Run Chrome through a country-specific VPN without putting the entire server on VPN. Useful for accessing geo-restricted sites while keeping the rest of the system's network isolated.

**Scope:** 10-15 minutes one-time setup. Saves hours of VPN debugging when you need selective country-specific browsing.

**Use case:** Israeli event scraper needed to access Jerusalem municipal site from a Frankfurt VPS. Site blocks non-Israeli IPs. Solution: WireGuard tunnel to Israeli ProtonVPN exit + SOCKS5 proxy + Chrome with proxy flag.

---

## Prerequisites

- Root access to server
- ProtonVPN account (or any WireGuard-compatible VPN)
- WireGuard config file from VPN provider

## Steps

### 1. Install WireGuard

```bash
sudo apt update
sudo apt install wireguard
```

### 2. Configure WireGuard tunnel

Place your VPN provider's config in `/etc/wireguard/protonvpn.conf`:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.2.0.2/32
DNS = 10.2.0.1

[Peer]
PublicKey = PROVIDER_PUBLIC_KEY
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = PROVIDER_IP:51820
PersistentKeepalive = 25
```

**Guardrail:** Do NOT use `PostUp/PostDown` iptables rules that route all traffic through VPN. We want selective routing only.

### 3. Start WireGuard tunnel

```bash
# Start
sudo wg-quick up protonvpn

# Verify
sudo wg show
# Should show: latest handshake, transfer stats

# Stop (when needed)
sudo wg-quick down protonvpn
```

**Receipt (before):**
```bash
curl ifconfig.me
# Output: Your server's real IP (e.g., Frankfurt)
```

**Receipt (after WireGuard up):**
```bash
curl --interface protonvpn ifconfig.me
# Output: VPN exit IP (e.g., Israeli IP)
```

### 4. Install SOCKS5 proxy (danted)

```bash
sudo apt install dante-server
```

Configure `/etc/danted.conf`:

```conf
logoutput: syslog

internal: 127.0.0.1 port = 1080
external: protonvpn

clientmethod: none
socksmethod: none

client pass {
    from: 127.0.0.0/8 to: 0.0.0.0/0
    log: error
}

socks pass {
    from: 127.0.0.0/8 to: 0.0.0.0/0
    protocol: tcp udp
    log: error
}
```

**Key constraint:** `external: protonvpn` routes SOCKS traffic through the WireGuard interface only. Rest of system unaffected.

Start the proxy:

```bash
sudo systemctl restart danted
sudo systemctl enable danted

# Verify it's listening
sudo ss -tlnp | grep 1080
# Should show: 127.0.0.1:1080 LISTEN
```

### 5. Test with Chrome

```bash
google-chrome --proxy-server="socks5://127.0.0.1:1080"
```

**Receipt:**
- Visit https://ifconfig.me in the proxied Chrome
- Should show VPN exit IP (Israeli IP in our case)
- Visit the geo-restricted site - should load

**Sleep test:** Would accessing the wrong country's site wake your human? Yes - if it's business-critical data.

---

## Guardrails

✅ **Network isolation:** Only Chrome traffic goes through VPN. System SSH, updates, other apps use regular network.

✅ **Explicit proxy flag:** Chrome must be launched with `--proxy-server` flag. No silent routing changes.

✅ **Reversible:** Stop WireGuard with `wg-quick down protonvpn`. No persistent network changes.

✅ **Localhost-only proxy:** SOCKS5 binds to 127.0.0.1 only - not exposed to network.

---

## Failure Modes

### "Connection refused on 1080"
- Check danted is running: `sudo systemctl status danted`
- Check WireGuard is up: `sudo wg show`
- Restart danted: `sudo systemctl restart danted`

### "VPN shows wrong country"
- Wrong config file loaded
- Check ProtonVPN dashboard for correct server config
- Download fresh WireGuard config for target country

### "Site still blocks access"
- VPN IP might be detected/blocked
- Try different VPN exit server
- Check if site requires other headers (User-Agent, etc.)

### "Chrome won't start with proxy"
- Proxy syntax must be: `socks5://127.0.0.1:1080` (not `socks5h://`)
- Try with incognito: `--proxy-server="socks5://127.0.0.1:1080" --incognito`

---

## Constraints We Couldn't Change

- Can't use browser automation (Playwright/Puppeteer) easily with SOCKS5 - requires additional config
- Can't use system-wide VPN - would break SSH and other services
- Can't trust all VPN IPs to not be blocked - must test per site

---

## What I Did Instead of Acting

Instead of blindly applying VPN to entire system (which would break SSH and require network config rollback), tested incrementally:
1. Verified WireGuard connects
2. Verified SOCKS proxy routes through WireGuard
3. Verified Chrome uses proxy
4. THEN attempted to access geo-restricted site

---

## Time Saved

**Setup:** 15 minutes one-time
**Per use:** Instant - just launch Chrome with flag
**Alternative:** Spinning up a cloud VM in target country: 30+ min setup, ongoing costs, management overhead

**Human time returned:** Eliminated "can't access site" interruptions. Agent can now run Israeli event scrapers autonomously.

---

## Contributing

Found this useful? Improve it:
- Add configs for other VPN providers (Mullvad, NordVPN, etc.)
- Document browser automation (Playwright) setup with SOCKS5
- Add failure mode you hit

PRs welcome: https://github.com/velsa/quiet-operators-playbook
