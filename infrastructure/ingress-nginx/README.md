# Ingress nginx (RKE2-bundled)

L7 routing inside the cluster. Receives plain HTTP from cloudflared (loopback) and routes by `Host:` header to the per-app Service.

## What is currently running

```
$ kubectl -n kube-system get pods -l app.kubernetes.io/name=rke2-ingress-nginx-controller
rke2-ingress-nginx-controller-f76mn   1/1 Running   2 (40d ago)   42d
```

The controller runs in `kube-system`, not `ingress-nginx`. The latter namespace is empty because RKE2 deploys the controller via its own helm-controller, scoped to `kube-system`.

## Install method

Bundled with RKE2 itself. RKE2's helm-controller installs the chart at cluster bootstrap:

```
$ helm list -A | grep ingress
rke2-ingress-nginx   kube-system   1   deployed   rke2-ingress-nginx-4.14.303   1.14.3
```

No manifests for the controller live in this repo. Tuning is done via `/etc/rancher/rke2/config.yaml` HelmChartConfig (not committed; document any custom values here when added).

## How apps use it

Each app's Ingress object sets:

```yaml
ingressClassName: nginx
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

The `ssl-redirect: false` annotation is mandatory under the Cloudflare Tunnel pattern (see `decisions/0001`); without it the controller would 308 plain HTTP, which would loop with cloudflared.

## Current Ingress resources

```
$ kubectl get ingress -A
NAMESPACE        NAME                 CLASS   HOSTS                            ADDRESS
alexmchugh-dev   alexmchugh-dev       nginx   alexmchugh.dev                   <host-ip>
fixmycampus      fixmycampus-client   nginx   fixmycampus.alexmchugh.dev       <host-ip>
fixmycampus      fixmycampus-server   nginx   fixmycampus-api.alexmchugh.dev   <host-ip>
```

(`getdfx.uk` reaches the dfx app via a NodePort + cloudflared `http://localhost:30092` route, not via Ingress. Inconsistent — see `decisions/0003`.)
