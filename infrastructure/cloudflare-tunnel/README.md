# Cloudflare Tunnel

Public ingress for the cluster. Terminates TLS at Cloudflare's edge and forwards plain HTTP to the in-cluster nginx Ingress controller via `localhost:80`. No inbound port forwarding on the home router; cloudflared makes outbound connections only.

See `decisions/0001` for the rationale.

## What is currently running

`cloudflared` is **not** in the cluster. It runs as a `systemctl` unit on the cluster host (the same Rocky Linux 9 box that runs RKE2). The tunnel is named `homelab`.

```
$ sudo systemctl status cloudflared --no-pager | head -3
● cloudflared.service - cloudflared
     Loaded: loaded (/etc/systemd/system/cloudflared.service; enabled)
     Active: active (running)
```

## Configuration

`/etc/cloudflared/config.yml` (not committed — contains the tunnel ID and credential file path):

```yaml
tunnel: <tunnel-id>
credentials-file: /home/alex/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: getdfx.uk
    service: http://localhost:30092
  - hostname: alexmchugh.dev
    service: http://localhost:80
  - hostname: fixmycampus.alexmchugh.dev
    service: http://localhost:80
  - hostname: fixmycampus-api.alexmchugh.dev
    service: http://localhost:80
  - service: http_status:404
```

Two service patterns visible:

- `http://localhost:80` → the nginx Ingress controller. The controller routes by `Host:` to `alexmchugh.dev`, `fixmycampus.alexmchugh.dev`, and `fixmycampus-api.alexmchugh.dev`.
- `http://localhost:30092` → a NodePort exposing the dfx-tag-generator Service directly, bypassing the Ingress. Legacy pattern; see `decisions/0003`.

The catch-all `service: http_status:404` must be the last entry. Cloudflare's tunnel rejects configs where the catch-all is mid-list.

## Adding a new public hostname

1. Edit `/etc/cloudflared/config.yml`, add the new hostname rule **above** `service: http_status:404`.
2. `cloudflared tunnel route dns homelab <hostname>` to create the Cloudflare DNS record pointing at the tunnel.
3. `sudo systemctl restart cloudflared`.
4. `sudo systemctl status cloudflared --no-pager | head -8` to confirm the new config validated.

## Failure modes

- **Tunnel daemon crashes**: all four hostnames go offline. cloudflared has no replicas. Mitigation deferred until traffic justifies a second host.
- **Misordered ingress (catch-all not last)**: cloudflared refuses to start. Symptom is `systemctl restart cloudflared` failing and the previous config remaining active until corrected.
- **Stale Cloudflare DNS**: a new record can take ~30s to propagate; client-side DNS caches (especially Safari and macOS mDNSResponder) sometimes hold the previous NXDOMAIN longer.
