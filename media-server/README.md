# Media server

This Compose project runs Plex and Jellyfin side by side over the same media
library. Tunarr turns library media into custom linear channels that appear in
Jellyfin's Live TV section. Requests entered in Seerr flow through Sonarr or
Radarr, which use Prowlarr's indexers and send downloads to qBittorrent. Sonarr
and Radarr then import completed downloads into the appropriate library.

```text
Seerr -> Sonarr/Radarr -> Prowlarr (search)
                   |
                   +-> qBittorrent -> /data/Downloads
                   |                       |
                   +<---- completed -------+
                   |
                   +-> /data/TV or /data/Movies -> Plex/Jellyfin

Jellyfin library -> Tunarr schedule -> Jellyfin Live TV
```

## Start the stack

Create the host directories if they do not exist:

```sh
sudo mkdir -p /media/Downloads/{movies,tv} /media/{Movies,TV}
```

Do not recursively change ownership on an existing NAS library. Verify the
account IDs with `id`, then grant the configured UID/GID write access to the
Downloads, Movies, and TV directories using the ownership, shared-group, or ACL
policy appropriate for that NAS. Plex and Jellyfin only require read access.

From the repository root, create `vars.env` only if it does not already exist,
enter the Plex values for a new installation, then start the stack. Existing
installations should retain their current `vars.env`.

```sh
cd media-server
test -e vars.env || cp vars.env.example vars.env
mkdir -p config/seerr/data
sudo chown 1000:1000 config/seerr/data
sudo mkdir -p config/tunarr/data
sudo chown 1000:1000 config/tunarr/data
docker compose pull
docker compose up -d
```

The LinuxServer containers use the configured UID and GID `1000`; change every
`PUID` and `PGID` together if the account that owns `/media` uses different IDs.
The official Seerr container runs as UID 1000 independently of those variables
and needs write access only to `config/seerr/data`.

## One-time application setup

Use Compose service names (not `localhost`) when one container connects to
another.

1. Open qBittorrent at `http://SERVER_IP:8080` and change its temporary default
   password. Set the default save path to `/data/Downloads`. Add categories:
   `movies` at `/data/Downloads/movies` and `tv` at `/data/Downloads/tv`.
2. Open Sonarr at `http://SERVER_IP:8989`. Set its root folder to `/data/TV`.
   Add qBittorrent under **Settings -> Download Clients** with host
   `qbittorrent`, port `8080`, its credentials, and category `tv`. Enable
   **Completed Download Handling**.
3. Open Radarr at `http://SERVER_IP:7878`. Set its root folder to
   `/data/Movies`. Add the same qBittorrent client with category `movies` and
   enable **Completed Download Handling**.
4. Open Prowlarr at `http://SERVER_IP:9696` and add the desired indexers. Under
   **Settings -> Apps**, add Sonarr at `http://sonarr:8989` and Radarr at
   `http://radarr:7878`, using API keys from each app's **Settings -> General**
   page. Use **Full Sync** so Prowlarr manages their indexers. For an indexer
   that requires Cloudflare handling, add a FlareSolverr indexer proxy using
   `http://flaresolverr:8191` and apply the same tag to that indexer.
5. Open Jellyfin at `http://SERVER_IP:8096`, create its administrator account,
   and create libraries pointing at `/data/TV` and `/data/Movies`. Jellyfin's
   media mount is read-only by design. Under **Dashboard -> Playback ->
   Transcoding**, select **Intel Quick Sync (QSV)** and use
   `/dev/dri/renderD128` as the device. Only enable codecs supported by the
   server's Intel generation.
6. Open Tunarr at `http://SERVER_IP:8000`. In its initial setup, add Jellyfin as
   a media source using `http://jellyfin:8096` and an API key created under
   Jellyfin's **Dashboard -> Advanced -> API Keys**. Create a channel, add *The
   Twilight Zone* with **Add Series**, and choose chronological or shuffled
   scheduling. Create a transcode configuration using HLS and VA-API with
   `/dev/dri/renderD128`, then assign it to the channel. Tunarr currently
   recommends VA-API over Quick Sync for Intel GPUs on Linux.
7. In Jellyfin, open **Dashboard -> Live TV**, add an **HDHomeRun** tuner
   manually at `http://tunarr:8000`, then add an **XMLTV** guide provider using
   `http://tunarr:8000/api/xmltv.xml`. After the guide refresh completes, the
   custom channel appears in Jellyfin's Live TV section. Tunarr's documentation
   recommends HDHomeRun rather than M3U for Jellyfin because some users see
   playback instability at program boundaries with M3U.
8. Open Seerr at `http://SERVER_IP:5055`. In the setup wizard, choose Jellyfin,
   sign in with the Jellyfin administrator, and use `http://jellyfin:8096` as
   the internal URL. Use the browser-accessible Jellyfin URL as the external
   URL and select the Jellyfin libraries to scan. Plex can also be connected
   later from Seerr's media-server settings.
9. In Seerr's **Settings -> Services**, add Sonarr with hostname `sonarr`, port
   `8989`, SSL disabled, and its API key; add Radarr with hostname `radarr`, port
   `7878`, SSL disabled, and its API key. Mark each as the **Default** server and
   select its quality profile, `/data/TV` or `/data/Movies` root folder, and all
   other required defaults such as minimum availability. Enable automatic search
   so approved requests immediately start searching, and enable scanning so
   Seerr recognizes existing or already-requested media.

After this setup, a request in Seerr triggers the appropriate manager,
which searches Prowlarr, sends a release to qBittorrent, and imports it after
completion. For torrents that continue seeding, Sonarr/Radarr normally hardlink
the file into the media library rather than deleting the qBittorrent copy. This
requires Downloads, Movies, and TV to be on the same underlying host filesystem;
compare `stat -c '%d'` for those directories on the Linux host. If their device
IDs differ, imports use copies instead. Downloads are removed according to the
seed limits configured in qBittorrent.

## Existing installations

The old `/downloads` container path has been retired from the stack. qBittorrent
and both media managers now use `/data/Downloads`; verify that no active or
seeding torrent still refers to `/downloads` before deploying this version.

The old library paths (`/tv` and `/movies`) remain mounted as compatibility
aliases for existing root folders. Migrate those roots in stages:

1. Back up the `media-server/config` directory and pause new grabs.
2. Set qBittorrent's default and category paths to the exact-case
   `/data/Downloads` paths and finish or relocate any torrent that still uses
   `/downloads`.
3. Recreate the affected containers with
   `docker compose up -d --force-recreate qbittorrent sonarr radarr jackett`.
   Verify afterward that qBittorrent retained the exact-case default and
   category paths.
4. Add `/data/TV` and `/data/Movies` as root folders. Use Sonarr's series editor
   and Radarr's movie editor to change existing entries to the new roots. Do not
   ask the apps to move files: the old and new paths point at the same host data.
5. Resume grabs and verify that a completed TV and movie download each imports.

The host directories and files remain in the same `/media` locations throughout.
The remaining `/tv` and `/movies` aliases can be removed from Compose later,
after no library root refers to them.
