# NGINX to Traefik Edge Cutover Prep - 2026-05-22

## Current state

- NGINX is still the public listener on Sauvage `:80` and `:443`.
- Traefik Edge is running in k8s on Sauvage with `hostNetwork`, currently on `:7080` and `:7443`.
- The desired final state is Traefik Edge on `:80` and `:443`, with NGINX stopped and retained for rollback.

## Backup

Readable NGINX config and host state were saved on Sauvage:

`/home/ubuntu/backups/k8s-legacy-decom/20260522-nginx-edge-cutover/`

Known backup limitation: the SSH user cannot read these root-only files without interactive sudo:

- `/etc/nginx/htpasswd-openclaw-images`
- `/etc/nginx/snippets/openclaw-gateway-authorization.conf`

## Routes Added To Traefik Edge

`networking/traefik-edge/legacy-public-routes.yaml` adds:

- standalone k8s public routes for `bundles.e-dani.com` and `sii.e-dani.com`.
- `edge-brain-public` for `brain.e-dani.com`, with external-dns annotations so the public record stays Cloudflare-proxied while routing to `skirmshop-brain` in `skirmshop-brain-prod`.
- k8s route for `skirmshop.e-dani.com/labels-cex` to `labels-correos-express-adapter`.
- temporary legacy host routes through `sauvage-localhost` for affiliate, SkirmBooks, Synapse webhooks/admin, OpenClaw webhooks, and selected SkirmShop legacy paths.
- `ExternalName: localhost` is intentional because Traefik Edge is pinned to Sauvage with `hostNetwork`.

The live DaemonSet was patched with:

`--providers.kubernetescrd.allowExternalNameServices=true`

The Helm release metadata is currently unhealthy/corrupt: `helm status traefik-edge -n traefik-edge` reports `superseded`, and `helm upgrade` fails with release storage errors. The running DaemonSet is healthy, but the Helm release must be repaired before relying on Helm for the final port cutover.

## Brain Public DNS

`brain.e-dani.com` previously existed in Cloudflare as a DNS-only CNAME to `sauvage.e-dani.com`, so LAN clients could hit the raw OVH IP and fail before reaching NGINX/Traefik.

On 2026-05-22 the record was changed to Cloudflare-proxied while preserving the CNAME target. The GitOps source is now `edge-brain-public`, which declares:

- `external-dns.alpha.kubernetes.io/hostname: brain.e-dani.com`
- `external-dns.alpha.kubernetes.io/target: sauvage.e-dani.com`
- `external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"`

NGINX on Sauvage is still the public listener for now. Its `brain.e-dani.com` vhost was backed up to `/etc/nginx/backups/brain.e-dani.com.20260522180330.pre-traefik-edge` and updated to proxy to Traefik Edge on `https://127.0.0.1:7443`.

## Direct Traefik Checks

Checked from Sauvage against `https://127.0.0.1:7443` with Host headers:

- `firecrawl.e-dani.com /` without Authorization -> `401`
- `firecrawl.e-dani.com /` with the legacy Bearer -> `200`
- `firecrawl.e-dani.com /health` -> `200`
- `whatsapp.e-dani.com /` -> `302`
- `harbor.e-dani.com /` -> `200`
- `litellm.e-dani.com /health/readiness` -> `200`
- `skirmshop.e-dani.com /bundles/` -> `302`
- `skirmshop.e-dani.com /labels/health` -> `200`
- `skirmshop.e-dani.com /labels-ups/health` -> `200`
- `skirmshop.e-dani.com /picker/health` -> `200`
- `skirmshop.e-dani.com /sii/` -> `302`
- `skirmshop.e-dani.com /translations/` -> `302`
- `affiliate.skirmshop.es /health` -> `200`
- `go.skirmshop.es /health` -> `200`
- `openclaw.e-dani.com /gmail-pubsub` -> `405` on GET, expected for a push endpoint.
- `synapse.e-dani.com /webhooks/health` -> `404`, backend reachable but route does not expose a health endpoint.
- `skirmbooks.e-dani.com /` -> `302`
- `rag.e-dani.com /` -> `200`
- `cv.e-dani.com /` -> `200`
- `cv-manu.e-dani.com /` -> `200`
- `skirmshop.e-dani.com /files/productos_venta_sin_stock_CATEGORIZADO.xlsx` -> `200`
- `brain.e-dani.com /` -> `200`
- `brain.e-dani.com /health` -> `200`

## Cutover Blockers

Do not stop NGINX until these are resolved or explicitly accepted:

- `firecrawl.e-dani.com`: resolved with `firecrawl-edge-auth`, a Traefik ForwardAuth shim that preserves the old Bearer check. The live Secret is intentionally not stored in Git and should be moved to Vault/ExternalSecret.
- `openclaw.e-dani.com /openclaw-mem/`: backend port `5002` is not currently listening. Root `/` is routed to `18789`.
- `images.openclaw.e-dani.com`: OpenClaw generated-image static store. NGINX serves `/var/lib/openclaw-images` with BasicAuth realm `openclaw-images`; allowed paths are `/ephemeral/<32hex>.(png|jpg|jpeg|webp)` and `/permanent/<32hex>.(png|jpg|jpeg|webp)`. This must be migrated before NGINX shutdown, preserving the hostPath and `/etc/nginx/htpasswd-openclaw-images` auth.
- `obsidian.e-dani.com`: deprecated. It is intentionally not migrated.
- `synapse.e-dani.com/attachments/`: routed to `edge-static-server` with the same LAN/VPN/Tailscale allowlist.
- some NGINX routes point to legacy ports that are not currently listening, so Traefik correctly returns `502`:
  - `skirmshop.e-dani.com/sn`
  - `skirmshop.e-dani.com/collections-tree`
  - `skirmshop.e-dani.com/product-ai`
  - `skirmshop.e-dani.com/`
  - `sauvage.e-dani.com/healthz`
  - `qdrant.e-dani.com`
  - `webui.e-dani.com`

## Final Cutover Procedure

1. Repair the `traefik-edge` Helm release or replace it with a clean GitOps-managed release.
2. Provide equivalents for auth/static blockers above.
3. Stop NGINX on Sauvage.
4. Move Traefik Edge host ports from `7080/7443` to `80/443`.
5. Validate all public Host/path checks.
6. Keep `/etc/nginx` and the backup directory for 30 days.

Rollback is to restore Traefik Edge to `7080/7443` and start NGINX again.

## Cutover Completed

Completed on 2026-05-22:

- NGINX on Sauvage was stopped and left installed/configured for rollback.
- NGINX service was disabled in systemd so a host reboot does not reclaim `:80/:443`.
- Traefik Edge Helm release was repaired from corrupt/superseded state and upgraded to a clean deployed release.
- Traefik Edge now binds hostNetwork `:80` and `:443`.
- Helm release after cutover: `traefik-edge` revision `10`, status `deployed`.
- Traefik required `NET_BIND_SERVICE` plus `runAsUser: 0` to bind low ports on the Sauvage host network.
- `images.openclaw.e-dani.com` is DNS-only and uses a dedicated cert-manager certificate `images-openclaw-tls`; Cloudflare proxied mode is not valid for this nested hostname with the standard Universal SSL certificate.

Final backup path on Sauvage:

`/home/ubuntu/backups/k8s-legacy-decom/20260522-nginx-traefik-cutover-final/`

It contains tarballs for:

- `/etc/nginx`
- `/etc/letsencrypt`
- `/var/lib/openclaw-images`

Post-cutover validation highlights:

- `brain.e-dani.com/health` -> `200`
- `firecrawl.e-dani.com/` without token -> `401`
- `n8n.e-dani.com/` -> `200`
- `whatsapp.e-dani.com/` -> `302`
- `images.openclaw.e-dani.com/ephemeral/<real>.png` -> `401`, certificate valid from Let's Encrypt
- `openclaw.e-dani.com/health` -> `200`
- `openclaw-webhooks.e-dani.com/health` -> `200`
- `synapse.e-dani.com/` -> `200`
- `skirmshop.e-dani.com/picker` -> `302`
- `skirmshop.e-dani.com/labels-ups` -> `301`
- `skirmshop.e-dani.com/collections-tree` -> `301`; `/collections-tree/` returns app-owned `410`
- `harbor.e-dani.com/v2/` -> `401`

Known preexisting issue:

- `skirmshop.e-dani.com/` returns `502` both before and after the cutover because the legacy root backend is not healthy/listening. This is not a Traefik regression.

Rollback:

```bash
helm upgrade traefik-edge traefik/traefik \
  --namespace traefik-edge \
  --version 40.2.0 \
  -f /tmp/traefik-edge-values-rollback-7080-7443.yaml \
  --wait --timeout 5m

kubectl exec -n traefik-edge sauvage-hostctl -- chroot /host /bin/systemctl start nginx
```

If the temporary `sauvage-hostctl` pod is no longer present, recreate any privileged hostPath `/` pod on Sauvage or start NGINX through SSH with interactive sudo.
