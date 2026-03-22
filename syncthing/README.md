# Syncthing

Syncthing running via Docker Compose

## Setup

### 1. Create the `vars.env` file

Copy or create a `vars.env` file in the same directory as `docker-compose.yml`:

```
export SYNCTHING_USER=<placeholder>
export SYNCTHING_PASSWORD=<placeholder>
```

> `vars.env` is gitignored and never committed.

### 2. Start the container

```bash
docker compose up -d
```

### 3. Set GUI credentials

Run these once after the first boot to configure authentication for the web UI:

```bash
source vars.env
docker exec syncthing syncthing cli --home /config config gui user set $SYNCTHING_USER
docker exec syncthing syncthing cli --home /config config gui password set $SYNCTHING_PASSWORD
docker restart syncthing
```

Credentials are written into Syncthing's `config.xml` and persist via the named Docker volume — you only need to do this once per fresh install, or if the volume is wiped.

### 4. Access the web UI

```
http://<homeserver-ip>:8384
```

---

## Volumes

| Volume | Purpose |
|---|---|
| `syncthing-config` | Syncthing config and state (named Docker volume) |
| `/home/<userName>/sync` | Vault storage on host filesystem |

