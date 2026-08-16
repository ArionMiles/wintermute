# Wintermute

| Service     | WebUI Port |
|--------------|------------|
| Plex         | 32400      |
| qBittorrent  | 8080       |
| Sonarr       | 8989       |
| Radarr       | 7878       |
| Jackett      | 9117       |
| ZNC          | 2020       |
| Beszel       | 8090       |

FlareSolverr is available only to containers in the media-server Compose network.
Configure Jackett's **FlareSolverr API URL** as `http://flaresolverr:8191`.
