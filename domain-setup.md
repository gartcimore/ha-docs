# kornoglab.fr Domain Setup

## Architecture

```
Internet → Freebox (router, fixed IP 82.67.87.15) → DMZ → UniFi UCG Ultra (192.168.0.1)

LAN clients → dnsmasq (192.168.0.54:53)
  ├── *.kornoglab.fr → 192.168.0.54 (Synology)
  └── everything else → UniFi gateway (192.168.0.1)
        ├── *.becquerel, *.iot → resolved by UniFi
        └── public domains → forwarded upstream

HTTPS flow:
  Browser → ha.kornoglab.fr:443 → Synology built-in reverse proxy (port 443) → 192.168.0.30:8123 (Home Assistant)
```

## Components

| Component | Location | Purpose |
|-----------|----------|---------|
| OVH DNS | OVH control panel | `*.kornoglab.fr` A record → 82.67.87.15 |
| acme.sh | Synology Docker (`kornogHosted/acme`) | Wildcard Let's Encrypt cert via DNS-01 + OVH API, auto-renews + auto-deploys to DSM |
| Reverse proxy | Synology built-in (DSM Login Portal → Advanced) | TLS termination, subdomain routing |
| dnsmasq | Synology Docker (`kornogHosted/dnsmasq`) | Local wildcard DNS for `*.kornoglab.fr` → Synology |
| HA config | `haConfig/configuration.yaml` | `trusted_proxies` for reverse proxy |

## What's done

- [x] OVH DNS: `*.kornoglab.fr` → 82.67.87.15 (Freebox fixed IP Full Stack)
- [x] OVH API token created (unlimited validity, IP-restricted, scoped to `/domain/zone/*`)
- [x] `kornogHosted/acme`: container running, wildcard cert issued (kornoglab.fr + *.kornoglab.fr), auto-renewal via daemon
- [x] Synology built-in reverse proxy configured: `ha.kornoglab.fr:443` → `192.168.0.30:8123` with WebSocket headers
- [x] Let's Encrypt wildcard cert imported into DSM and assigned to reverse proxy
- [x] `haConfig/configuration.yaml`: `http.trusted_proxies` added for reverse proxy
- [x] Reverse proxy verified working: `curl --resolve ha.kornoglab.fr:443:192.168.0.54 https://ha.kornoglab.fr` returns HA with valid LE cert
- [x] `kornogHosted/dnsmasq`: config ready (but container not yet deployed)

## What's left to do

- [ ] Deploy dnsmasq container on Synology via Container Manager
- [ ] In UniFi: change DHCP DNS server for "wu tang lan" network to 192.168.0.54 (Synology)
- [ ] Test `https://ha.kornoglab.fr` from a LAN device (without `--resolve` hack)
- [ ] Restart Home Assistant to pick up the trusted_proxies config
- [ ] Add Synology DSM credentials to `acme.secrets.env` for auto cert deploy on renewal (see below)
- [ ] Rebuild acme container with updated entrypoint (adds `--deploy-hook synology_dsm`)
- [ ] Later: add more subdomains (parr services, etc.) as additional reverse proxy rules in DSM
- [ ] Clean up: remove `kornogHosted/nginx-proxy` project (no longer needed)

## Auto cert renewal + deploy to DSM

acme.sh has a built-in `synology_dsm` deploy hook that automatically uploads the renewed cert to DSM via its web API. This is configured in the acme container.

### Setup

Add these to `kornogHosted/acme/acme.secrets.env`:

```env
SYNO_Username=your_synology_admin_username
SYNO_Password=your_synology_admin_password
SYNO_Scheme=https
SYNO_Hostname=192.168.0.54
SYNO_Port=5001
SYNO_Certificate=kornoglab.fr
SYNO_Create=1
```

- `SYNO_Username` / `SYNO_Password`: a DSM admin user (consider creating a dedicated user for this)
- `SYNO_Certificate`: the certificate description in DSM (must match the name shown in Control Panel → Security → Certificate)
- `SYNO_Create=1`: creates the cert in DSM if it doesn't exist yet (first time only)

After adding the credentials, rebuild/restart the acme container. On next renewal, it will automatically push the new cert to DSM.

### Gotcha: curl error 60 (SSL verification)

The deploy hook connects to DSM at `https://192.168.0.54:5001`, but DSM serves a self-signed cert there, so curl inside the container fails with error 60 (`SSL peer certificate not OK`) and the deploy loops.

Fix: skip SSL verification for the DSM connection.

- Add `SYNO_Insecure=1` to `acme.secrets.env`
- The entrypoint also passes `--insecure` to the `acme.sh --deploy` command

The `dns_ovh` cert issuance and Let's Encrypt validation are unaffected — this only relaxes verification for the local DSM API call.

## Port 443 situation

Synology's built-in nginx owns port 443 (DSM core, not a package). The `kornogHosted/nginx-proxy` container cannot bind to it. Solution: use DSM's built-in reverse proxy instead (Control Panel → Login Portal → Advanced → Reverse Proxy). This works for all subdomains — just add more rules.

## Key IPs

| Host | IP |
|------|-----|
| Freebox (public) | 82.67.87.15 |
| UniFi UCG Ultra | 192.168.0.1 |
| Home Assistant | 192.168.0.30 |
| Synology | 192.168.0.54 |

## OVH API token

Stored in `kornogHosted/acme/acme.secrets.env` (gitignored). Backup the three keys (OVH_AK, OVH_AS, OVH_CK) in your password manager. Token is unlimited validity, restricted to Freebox IP, scoped to `/domain/zone/*`.

## UniFi notes

- UCG Ultra running Network 10.1.89
- No "Gateway Management" or static DNS in the UI — that's why we use dnsmasq
- UniFi local DNS records (per client) don't support wildcards
- VLANs: wu tang lan (192.168.0.0/24), iot (192.168.6.0/24), guest vlan (192.168.5.0/24), trusted (192.168.20.0/24)
