# 0001 — Cloudflare Tunnel for edge TLS, no cert-manager in cluster

Date: 2026-04-27
Status: Accepted

## Context

Three apps run in a self-hosted RKE2 cluster on a home network with no static public IP and no inbound port forwarding. Public users reach them at:

- alexmchugh.dev
- getdfx.uk
- fixmycampus.alexmchugh.dev (and fixmycampus-api.alexmchugh.dev)

DNS for both apex domains is on Cloudflare. The platform needed:

1. A way to expose cluster services without opening ingress ports on the home router.
2. Public TLS for every hostname.
3. Low operational cost and minimal moving parts for a one-person homelab.

cert-manager + Let's Encrypt + a managed Ingress NodePort + dynamic DNS would have worked but means the cluster owns certificate issuance, ACME challenges, renewals, and rate-limit handling. Each is a small failure mode that compounds.

## Decision

Cloudflare Tunnel (cloudflared) runs as a systemd unit on the cluster host. The tunnel terminates TLS at Cloudflare's edge and forwards plain HTTP to the in-cluster nginx Ingress controller on `localhost:80`. Ingress rules route by `Host:` header.

cert-manager is **not** installed. Cluster Ingress objects deliberately disable redirects with `nginx.ingress.kubernetes.io/ssl-redirect: "false"` so the tunnel-to-ingress hop stays HTTP.

## Consequences

**Positive**

- Zero certificate management inside the cluster. Cloudflare handles issuance, renewal, OCSP.
- No inbound NAT or port forwarding. The tunnel makes outbound connections only.
- DDoS, bot, and rate-limit protection ride along free of charge.
- Adding a new public hostname is a config-file edit + `cloudflared tunnel route dns` + `systemctl restart cloudflared`.

**Negative**

- The platform is tied to Cloudflare for ingress. Migrating away means re-introducing cert-manager and an externally-reachable Ingress.
- TLS terminates outside the cluster. Anything inside the cluster cannot make assertions about end-user TLS (e.g. mutual TLS at the edge would need extra work).
- The tunnel daemon is a single process on a single host. If the host dies, all three apps go offline. cloudflared replicas across nodes would mitigate but are not yet configured.
- Internal cluster traffic between cloudflared and the Ingress is plaintext HTTP on loopback. Acceptable on a single-host cluster; would need attention if the tunnel ran off-cluster or was replicated across hosts.

## Roadmap link

If a future app needs an ingress path that bypasses Cloudflare (e.g. a customer requiring direct DNS-to-cluster TLS), cert-manager + a ClusterIssuer is the planned addition. Not implemented today.
