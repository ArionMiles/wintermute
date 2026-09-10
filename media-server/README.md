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

1. Open qBittorrent at `http://SERVER_IP:8080`.

   - Change its temporary default password.
   - Set the default save path to `/data/Downloads`.
   - Add these categories:
     - `movies` at `/data/Downloads/movies`.
     - `tv` at `/data/Downloads/tv`.

2. Open Sonarr at `http://SERVER_IP:8989`.

   - Set its root folder to `/data/TV`.
   - Under **Settings -> Download Clients**, add qBittorrent with:
     - Host: `qbittorrent`.
     - Port: `8080`.
     - Credentials: the qBittorrent credentials.
     - Category: `tv`.
   - Enable **Completed Download Handling**.

3. Open Radarr at `http://SERVER_IP:7878`.

   - Set its root folder to `/data/Movies`.
   - Add the same qBittorrent client with category `movies`.
   - Enable **Completed Download Handling**.

4. Open Prowlarr at `http://SERVER_IP:9696`.

   - Add the desired indexers.
   - Under **Settings -> Apps**, add:
     - Sonarr at `http://sonarr:8989`.
     - Radarr at `http://radarr:7878`.
     - Use the API key from each application's **Settings -> General** page.
   - Use **Full Sync** so Prowlarr manages their indexers.
   - For an indexer that requires Cloudflare handling:
     - Add a FlareSolverr indexer proxy using
       `http://flaresolverr:8191`.
     - Apply the same tag to the proxy and indexer.

5. Open Jellyfin at `http://SERVER_IP:8096`.

   - Create its administrator account.
   - Create libraries pointing at `/data/TV` and `/data/Movies`.
     Jellyfin's media mount is read-only by design.
   - To enable the locally hosted Ultrachromic theme, open
     **Dashboard -> Branding**, put one of the following lines in
     **Custom CSS**, and save. Monochromic is the suggested starting point:

     ```css
     @import url('/web/custom-themes/ultrachromic/presets/monochromic_preset.css');
     ```

     The other available entry points are `kaleidochromic_preset.css` and
     `novachromic_preset.css` in the same URL directory. The theme source and
     verification details are documented in
     [`jellyfin/themes/ultrachromic/README.md`](jellyfin/themes/ultrachromic/README.md).
   - Under **Dashboard -> Playback -> Transcoding**:
     - Select **Intel Quick Sync (QSV)**.
     - Use `/dev/dri/renderD128` as the device.
     - Enable only codecs supported by the server's Intel generation.

6. Open Tunarr at `http://SERVER_IP:8000`.

   - In its initial setup, add Jellyfin as a media source:
     - URL: `http://jellyfin:8096`.
     - API key: create one under Jellyfin's
       **Dashboard -> Advanced -> API Keys**.
   - Create a channel, add *The Twilight Zone* with **Add Series**, and
     choose chronological or shuffled scheduling.
   - Create a transcode configuration:
     - Use HLS and VA-API with `/dev/dri/renderD128`.
     - Assign the configuration to the channel.
     - Prefer VA-API over Quick Sync for Intel GPUs on Linux, as recommended
       by Tunarr.

7. In Jellyfin, open **Dashboard -> Live TV**.

   - Add an **HDHomeRun** tuner manually at `http://tunarr:8000`.
   - Add an **XMLTV** guide provider using
     `http://tunarr:8000/api/xmltv.xml`.
   - Wait for the guide refresh to complete. The custom channel should then
     appear in Jellyfin's Live TV section.
   - Prefer HDHomeRun over M3U for Jellyfin. Tunarr notes that M3U can have
     playback instability at program boundaries.

8. Open Seerr at `http://SERVER_IP:5055`.

   - In the setup wizard, choose Jellyfin.
   - Sign in with the Jellyfin administrator.
   - Set the internal URL to `http://jellyfin:8096`.
   - Set the external URL to a browser-accessible address such as
     `http://SERVER_IP:8096`.
   - Select the Jellyfin libraries to scan.
   - Note how Seerr uses these addresses:
     - The internal address is used for API calls.
     - The external address is used for **Play on Jellyfin** links.
   - Plex can also be connected later from Seerr's media-server settings.

9. In Seerr, open **Settings -> Services**.

   - Add Sonarr with:
     - Hostname: `sonarr`.
     - Port: `8989`.
     - SSL: disabled.
     - API key: the key from Sonarr.
     - External URL: `http://SERVER_IP:8989`.
   - Add Radarr with:
     - Hostname: `radarr`.
     - Port: `7878`.
     - SSL: disabled.
     - API key: the key from Radarr.
     - External URL: `http://SERVER_IP:7878`.
   - Use the external URLs only for browser-facing
     **Open in Sonarr/Radarr** links. Seerr uses the Compose service names for
     API calls.
   - Mark each application as the **Default** server.
   - Select its quality profile, `/data/TV` or `/data/Movies` root folder, and
     other required defaults such as minimum availability.
   - Enable automatic search so approved requests immediately start searching.
   - Enable scanning so Seerr recognizes existing or already-requested media.

After this setup, a request in Seerr triggers the appropriate manager,
which searches Prowlarr, sends a release to qBittorrent, and imports it after
completion. For torrents that continue seeding, Sonarr/Radarr normally hardlink
the file into the media library rather than deleting the qBittorrent copy. This
requires Downloads, Movies, and TV to be on the same underlying host filesystem;
compare `stat -c '%d'` for those directories on the Linux host. If their device
IDs differ, imports use copies instead. Downloads are removed according to the
seed limits configured in qBittorrent.

## Configure quality and language profiles

Sonarr and Radarr store profiles in their application databases, so recreate
these settings through each application's web interface after a fresh install.
The normal profile accepts 1080p when no suitable 4K release exists. The
fallback profile additionally accepts lower resolutions for selected titles.

1. Create an English-or-Hindi custom format in Radarr.

   - Open **Settings -> Custom Formats** and add a custom format named
     `Audio - English or Hindi`.
   - Add a **Language** condition for `English`.
   - Add another **Language** condition for `Hindi`.
   - Leave **Required**, **Negate**, and **Except Language** disabled for both
     conditions. Multiple non-required conditions of the same type are matched
     as alternatives, so either language satisfies the custom format.
   - Save the custom format.

2. Create the same custom format in Sonarr.

   - Open **Settings -> Custom Formats**.
   - Repeat the custom-format settings from the previous step exactly.

3. Create the normal profile in Radarr.

   - Open **Settings -> Profiles** and add a quality profile named
     `4K - 1080p EN-HI`.
   - Set **Language** to `Any`. Leaving it on `Original` rejects English and
     Hindi dubs when a movie's original language is something else.
   - Enable **Upgrades Allowed**.
   - Tick the checkbox for, and arrange, these qualities from most to least
     preferred. Ordering controls preference; only checked qualities are
     eligible for download:
     - `Bluray-2160p`.
     - `WEB 2160p`.
     - `Bluray-1080p`.
     - `WEB 1080p`.
   - Leave 720p, 480p, `BR-DISK`, and remux qualities disabled. Enable remuxes
     only if their substantially larger file sizes are acceptable.
   - Set **Upgrade Until** to `Bluray-2160p`.
   - Give `Audio - English or Hindi` a score of `10000`.
   - Set **Minimum Custom Format Score** and
     **Upgrade Until Custom Format Score** to `10000`.
   - Keep the combined positive scores of every other custom format below
     `10000`. If other scores can total `10000` or more, increase both the
     language score and minimum score above that total so the language format
     remains mandatory.
   - Save the profile.

4. Create the normal profile in Sonarr.

   - Open **Settings -> Profiles** and add a quality profile named
     `4K - 1080p EN-HI`.
   - Repeat the quality checkboxes, ordering, upgrade settings, custom-format
     score, and minimum-score settings from the Radarr profile.
   - Add the `Audio - English or Hindi` custom format to this Sonarr profile
     with the same score. Sonarr supports language conditions through custom
     formats, but it does not have Radarr's additional standalone **Language**
     dropdown in the quality-profile editor.

5. Create a lower-resolution fallback profile in both applications.

   - Add another profile named `4K - 1080p - 720p-SD EN-HI`.
   - Copy the normal profile's settings.
   - Additionally tick the checkbox for these qualities and arrange them below
     the 1080p qualities. Leaving a quality unchecked means the profile will
     reject it regardless of its position:
     - `Bluray-720p`.
     - `WEB 720p`.
     - `HDTV-720p`.
     - `WEB 480p`.
     - `Bluray-480p` and `Bluray-576p`.
     - `DVD`.
     - `SDTV`, when old television series may require it.
   - Keep the same language custom-format score and minimum score. Lowering
     resolution must not weaken the language requirement.
   - Save the profile.

6. Use the profiles for new Seerr requests.

   - Open **Seerr -> Settings -> Services**.
   - Edit the Sonarr and Radarr services.
   - Refresh each service's profiles if the new profiles are not yet listed.
   - Select `4K - 1080p EN-HI` as the default profile.
   - To let a user select the fallback profile while requesting:
     - Open **Users**, edit that user, and enable the
       **Advanced Requests** permission.
     - In the request dialog, open the advanced options and select
       `4K - 1080p - 720p-SD EN-HI`.
   - Without **Advanced Requests**, change the title's profile directly in
     Sonarr or Radarr before triggering another search.

7. Migrate requests that were created with the old `Ultra-HD` profile.

   - In Radarr, use **Movies -> Movie Editor** to select every movie assigned
     to `Ultra-HD` and assign the appropriate new profile.
   - In Sonarr, open **Series**, choose **Select Series**, select every series
     assigned to `Ultra-HD`, and change their quality profile.
   - Trigger a search for monitored titles that are missing or have not met
     their quality cutoff.
   - Review an interactive search when a title remains missing. A lower
     resolution cannot help when no matching, sufficiently seeded release is
     available from the configured indexers.

Language matching uses the language that Sonarr or Radarr derives from the
release title, indexer metadata, and the title's original-language metadata. It
requires the application to classify a release as English or Hindi, but it
cannot inspect a torrent's actual audio tracks in advance or prevent an
otherwise acceptable multilingual file from containing additional languages.
