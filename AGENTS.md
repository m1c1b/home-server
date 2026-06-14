# AGENTS.md

Home-server infrastructure repo. No application code, no build/test/lint — it's declarative config (Kubernetes manifests, Helm charts/values, a Mihomo router template). Changes are applied to a live k3s cluster and a MikroTik router by hand, not via CI.

## Layout

- `k3s/native/` — raw Kubernetes manifests applied with `kubectl apply -f`. Subdirs by resource role, not by app: `apps/`, `ingress/`, `certs/`, `volumes/`, `storageclasses/`, `claims/`, `k3s/`. `namespaces.yml` defines all namespaces.
- `k3s/helm/cfg/` — values-only overrides for upstream charts (cert-manager, ingress-nginx, prometheus, immich, plex, coredns, seerr, reloader/reflector/trust-manager). Each dir is just a `values.yaml`; the chart itself is not vendored here.
- `k3s/helm/app/appflowy/` — the one fully vendored Helm chart (subcharts in `charts/`, deps in `Chart.yaml`/`Chart.lock`).
- `mikrotik/mihomo-cfg.yml.tmpl` — Mihomo/Clash.Meta proxy config template, NOT k8s.

## Conventions (match these when adding resources)

- One app per namespace; add new namespaces to `k3s/native/namespaces.yml`.
- Storage is node-local: `hostPath` under `/SSD/app-data/<app>` (config) and `/DATA/media` (media), or PVs with `storageClassName: local-generic-storage` / `local-path`. There is no cloud/dynamic storage.
- Media apps (sonarr/radarr/lidarr/prowlarr/qbittorrent) use `lscr.io/linuxserver/*` images with `PUID/PGID=1000`, `TZ=Etc/UTC`, and a busybox initContainer to `chown` `/mnt/media`.
- Ingress: `ingressClassName: nginx`. Internal hosts use `*.m1c1b.home`; externally-exposed services use `*.ru.tuna.am` (tuna tunnel). Service ports are usually named (`webui`, `http`) — reference by name in ingress backends.
- TLS via cert-manager `ClusterIssuer` `selfsigned-cluster-issuer` (self-signed; not Let's Encrypt).
- Comments in manifests are in Russian; keep that style.

## Gotchas

- Secrets and API keys are committed in plaintext in some manifests (e.g. probe `X-Api-Key` in `k3s/native/apps/media/sonarr.yml`, passwords in appflowy values). Do not introduce new ones casually, but be aware existing ones are intentional for this private repo.
- `mihomo-cfg.yml.tmpl` uses `$VAR` placeholders (e.g. `$LOG_LEVEL`, `$PROVIDERS_BLOCK`) substituted by an external tool before deployment — it is not valid YAML as-is. Edit the template, not a rendered copy.
- `k3s/helm/cfg/*` only hold override values; to know real defaults you must consult the upstream chart, not this repo.
- k3s auto-upgrades via the system-upgrade-controller `Plan` in `k3s/native/k3s/upgrade-plan.yml` (weekdays 04:00–07:00 UTC).

## Applying changes

There is no orchestration tooling. Native manifests: `kubectl apply -f <path>`. Helm cfg: `helm upgrade --install <release> <upstream-chart> -f k3s/helm/cfg/<name>/values.yaml`. The appflowy chart: `helm dependency build k3s/helm/app/appflowy` then `helm upgrade --install`.
