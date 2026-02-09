# Home Lab Kubernetes Cluster

GitOps-managed K3s cluster running on a single node in Vienna, Austria. All applications are deployed and synced via **ArgoCD** from this repository.

## Architecture

```mermaid
graph TB
    subgraph Internet
        CF[Cloudflare Tunnel]
        GH[GitHub Repository]
    end

    subgraph "K3s Cluster (pj)"
        direction TB
        ARGO[ArgoCD] -->|sync| GH

        subgraph "Ingress Layer"
            TRAEFIK[Traefik Ingress Controller]
        end

        subgraph "Media Stack"
            JF[Jellyfin<br/>Media Server]
            IPTV[IPTV-Proxy<br/>VPN Stream Proxy]
            ARR[Arr-Stack<br/>Content Automation]
        end

        subgraph "Photo & Files"
            IMMICH[Immich<br/>Photo Management]
        end

        subgraph "DevOps"
            WP[Woodpecker CI/CD]
            MON[Prometheus + Grafana]
        end

        subgraph "Tools"
            VIK[Vikunja<br/>Task Management]
            IT[IT-Tools]
        end

        subgraph "Storage"
            NAS[(Synology NAS<br/>192.168.1.162)]
            LOCAL[(Local Path<br/>Provisioner)]
        end
    end

    CF --> TRAEFIK
    TRAEFIK --> JF & ARR & IMMICH & WP & VIK & IT
    JF -->|M3U Tuner 1| IPTV
    JF -->|M3U Tuner 2| GH
    IPTV -->|WireGuard VPN| SRB[ProtonVPN Serbia]
    ARR -->|WireGuard VPN| VPN2[Gluetun VPN]
    JF & ARR --> NAS
    IMMICH --> NAS
```

## Network Overview

```mermaid
graph LR
    subgraph "Local Access  *.local"
        A1[jellyfin.local]
        A2[sonarr.local]
        A3[radarr.local]
        A4[prowlarr.local]
        A5[jellyseerr.local]
        A6[qbittorrent.local]
        A7[bazarr.local]
        A8[vikunja.local]
        A9[it-tools.local]
        A10[immich.local]
    end

    subgraph "External Access  *.pj-home-lab.com"
        B1[jellyfin.pj-home-lab.com]
        B2[sonarr.pj-home-lab.com]
        B3[radarr.pj-home-lab.com]
        B4[requests.pj-home-lab.com]
        B5[vikunja.pj-home-lab.com]
        B6[tools.pj-home-lab.com]
        B7[wood.pj-home-lab.com]
    end

    subgraph Cloudflare Tunnel
        CF[cloudflared]
    end

    CF --> B1 & B2 & B3 & B4 & B5 & B6 & B7
```

## Applications

| App | Namespace | Description | Local URL | External URL |
|-----|-----------|-------------|-----------|-------------|
| [Jellyfin](apps/jellyfin/) | `jellyfin` | Media server with Live TV (IPTV) | jellyfin.local | jellyfin.pj-home-lab.com |
| [Arr-Stack](apps/arr-stack/) | `arr-stack` | Content automation (Sonarr, Radarr, etc.) | sonarr.local | sonarr.pj-home-lab.com |
| [Immich](apps/immich/) | `immich` | Photo management | immich.local | - |
| [Monitoring](apps/monitoring/) | `monitoring` | Prometheus + Grafana | /grafana | - |
| [Vikunja](apps/vikunja/) | `vikunja` | Task management | vikunja.local | vikunja.pj-home-lab.com |
| [Woodpecker](apps/woodpecker/) | `woodpecker` | CI/CD pipeline | - | wood.pj-home-lab.com |
| [IT-Tools](apps/it-tools/) | `it-tools` | IT utilities | it-tools.local | tools.pj-home-lab.com |
| [Cloudflare](apps/cloudflare/) | `cloudflare` | Tunnel for external access | - | - |

### External Repos

| App | Repo | Description |
|-----|------|-------------|
| [IPTV-Proxy](https://github.com/Petar-Jorgic/iptv-playlists) | `iptv-proxy` (local build) | WireGuard VPN proxy for Serbian IPTV streams |

## Storage

```mermaid
graph LR
    subgraph "Synology NAS  192.168.1.162"
        NFS1["/volume1/docker/immich/library"]
        NFS2["/volume1/docker<br/>Movies, TV, Anime, Downloads"]
    end

    subgraph "Local Storage"
        LP[local-path provisioner<br/>Configs, Databases, Caches]
    end

    IMMICH[Immich] --> NFS1
    JF[Jellyfin] --> NFS2
    ARR[Arr-Stack] --> NFS2
    JF & ARR & IMMICH --> LP
```

| Storage Class | Type | Path | Used By |
|---------------|------|------|---------|
| `nfs-immich` | NFS | `/volume1/docker/immich/library` | Immich (200Gi) |
| `nfs-jellyfin` | NFS | `/volume1/docker` | Jellyfin (500Gi) |
| `nfs-arr` | NFS | `/volume1/docker` | Arr-Stack (32Ti) |
| `local-path` | Local | Node storage | All configs/DBs |

## GitOps with ArgoCD

This cluster uses the **App of Apps** pattern:

```
root-app.yaml          -> Points to argocd/
argocd/
  kustomization.yaml   -> Lists all ArgoCD Application manifests
  base.yaml            -> Namespaces, limits, storage
  secrets.yaml         -> Sealed secrets
  jellyfin.yaml        -> apps/jellyfin/
  arr-stack.yaml       -> apps/arr-stack/
  immich.yaml          -> apps/immich/
  monitoring.yaml      -> apps/monitoring/
  vikunja.yaml         -> apps/vikunja/
  woodpecker.yaml      -> apps/woodpecker/
  it-tools.yaml        -> apps/it-tools/
  cloudflare.yaml      -> apps/cloudflare/
```

All applications have **automated sync**, **prune**, and **self-heal** enabled.

## Secrets Management

Secrets are managed with **Sealed Secrets**. Sealed secrets are safe to commit to git - only the cluster can decrypt them.

## Default Resource Limits

Applied cluster-wide via LimitRange:

| | Requests | Limits |
|---|----------|--------|
| CPU | 100m | 500m |
| Memory | 256Mi | 1Gi |
