# Jellyfin

Media server with Live TV support via IPTV proxy.

## Architecture

```mermaid
graph LR
    CLIENT[Browser / App] --> TRAEFIK[Traefik]
    TRAEFIK --> JF[Jellyfin :8096]

    JF -->|"Tuner 1: VPN Proxy"| IPTV[iptv-proxy.default:8080<br/>/playlist.m3u]
    JF -->|"Tuner 2: Direct"| GH[GitHub Raw<br/>/playlist-deat.m3u]

    IPTV -->|WireGuard| VPN[ProtonVPN Serbia]
    VPN --> RS[Serbian IPTV Streams]
    GH --> DE[DE/AT + Football Streams]

    JF --> NAS[(NAS: Movies, TV, Anime)]
```

## Live TV Tuners

| Tuner | URL | Content | Via VPN |
|-------|-----|---------|---------|
| 1 | `http://iptv-proxy.default:8080/playlist.m3u` | 16 Serbian channels | Yes (Serbia) |
| 2 | `https://raw.githubusercontent.com/.../playlist-deat.m3u` | 27 DE/AT + 3 Football | No (direct) |

## Resources

| | Requests | Limits |
|---|----------|--------|
| CPU | 500m | 4000m |
| Memory | 1Gi | 4Gi |

## Storage

- **Config**: 10Gi (local-path)
- **Media**: 500Gi (NFS from Synology NAS)
- **Cache**: emptyDir

## Ingress

- `jellyfin.local`
- `jellyfin.pj-home-lab.com`
