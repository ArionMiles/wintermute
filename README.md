# Wintermute

| Service     | WebUI Port |
|--------------|------------|
| Plex         | 32400      |
| Jellyfin     | 8096       |
| qBittorrent  | 8080       |
| Sonarr       | 8989       |
| Radarr       | 7878       |
| Prowlarr     | 9696       |
| Seerr        | 5055       |
| Jackett      | 9117       |
| ZNC          | 2020       |
| Beszel       | 8090       |

FlareSolverr is available only to containers in the media-server Compose network.
Configure Prowlarr or Jackett to reach it at `http://flaresolverr:8191`.

See the [media-server setup guide](media-server/README.md) for the automated
request, download, and import workflow.
