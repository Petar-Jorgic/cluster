# Jellyfin

Media server with Live TV support via IPTV proxy and VAAPI hardware transcoding.

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
    JF --> GPU[/dev/dri - AMD Radeon iGPU]
```

## Hardware Transcoding

- **Type**: VAAPI (Video Acceleration API)
- **Device**: `/dev/dri/renderD128` (AMD Radeon, Ryzen 7 5825U iGPU)
- **Supported codecs**: H.264, HEVC, HEVC 10bit, MPEG2, VC1, VP9
- **Not supported**: VP8, AV1, VP9 10bit, HEVC RExt
- **Pinned to node `pj`** (only node with iGPU)
- **Privileged container** required for GPU device access

## Network / Streaming

- **LAN networks**: `192.168.1.0/24`
- **Known proxies**: `10.42.0.1` (Traefik/K8s gateway)
- **Remote bitrate limit**: 20 Mbit/s (~9 GB/h)
- Local clients get Direct Play (no transcoding), remote clients get VAAPI-accelerated transcoding

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

- `jellyfin.local` (lokal, Direct Play)
- `jellyfin.pj-home-lab.com` (extern via Pangolin, 20 Mbit/s Limit)
