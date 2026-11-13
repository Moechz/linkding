# linkding-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Linkding](https://github.com/sissbruecker/linkding).

## What this is

A four-file TOS Docker application archive (`linkdingdocker.tar.gz`) that installs
Linkding on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `linkdingdocker`)
- `linkdingdocker.lang` — 23-language superset (TOS requires 14)
- `linkdingdocker.svg` — application icon (1176 bytes, 7 SVG elements)
- `docker-compose.yml` — one service, image `sissbruecker/linkding:1.47.0`

## How it works

| | |
|---|---|
| Container image | `sissbruecker/linkding:1.47.0` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8006/` (container port 9090) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/etc/linkding/data` → `/Volume1/DockerAppData/linkdingdocker/data` |
| Logs | `docker logs linkdingdocker` (stdout/stderr) |

`/etc/linkding/data` (mounted from `DockerAppData/linkdingdocker/data`) holds `db.sqlite3`, the generated `secretkey.txt` and the `archive/` folder with saved page snapshots.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 9090.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **No external database.** Linkding stores bookmarks in SQLite; the container runs as uid 1000 and only the data folder is persistent.
- **First account = administrator.** Registration is open until the admin disables it, so no password is baked into the package.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `/Volume1/DockerAppData/linkdingdocker/data` | Linkding application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/linkdingdocker.tar.gz + .sha256
scripts/build.sh 1.47.0-2   # packaging iteration bump
scripts/build.sh 1.47.0-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/linkdingdocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v1.47.0-1`**, assets `linkdingdocker.tar.gz` +
   `linkdingdocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `linkdingdocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `1.47.0-1` → automated validation → review.

## License / branding

Linkding is distributed by its upstream authors under **MIT**. The container image is the official image from Docker Hub (`sissbruecker/linkding:1.47.0`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

- The web UI is plain HTTP; use a TLS reverse proxy before exposing it to the internet.
- The first registered account becomes the administrator - create it right after installing.
- Browser extensions need the instance URL; the bookmarklet works from any browser without extra setup.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
