# Configuration and deploying
## ⚙️ Configuration

| Variable | Default | What it does |
| --- | --- | --- |
| `NUXT_BLIZZARD_CLIENT_ID` | — *(required)* | Blizzard API client id |
| `NUXT_BLIZZARD_CLIENT_SECRET` | — *(required)* | Blizzard API client secret |
| `NUXT_SESSION_PASSWORD` | — *(required for accounts)* | Seals the sign-in cookie and encrypts the Battle.net tokens kept on disk (32+ characters) |
| `NUXT_ADMIN_BATTLETAGS` | `neuromask#2809` | BattleTags that may open the admin panel, comma-separated. They are *added* to the ones the site ships with, so a host that names nobody cannot lock the panel's owner out |
| `NUXT_PUBLIC_SITE_URL` | `https://heroofazeroth.com` | Canonical host for SEO and the sitemap |
| `NUXT_PUBLIC_GTM_ID` | `GTM-WXMVB755` | Tag manager container |
| `PORT` / `NITRO_PORT` | `3100` | Port the built server binds (`server/plugins/port.ts`) |
| `HOST` | — | Address the built server binds |
| `FTP_*` | — | Deploy target — see below |

The built server listens on `PORT`, else `NITRO_PORT`, else **3100**: Nitro's own fallback is 3000, which
the second application on that host already owns. `nuxt dev` uses the same number.

---

## 📦 Deploying

```bash
npm run deploy
```

1. `nuxt build` writes `.output`, the Nitro `node-server` preset.
2. The sources are committed and pushed to the current branch, `main`. GitHub holds the project only —
   `.output` is ignored by git and never committed.
3. The contents of `.output` are uploaded over FTP to `FTP_PATH`, together with a `deploy.json` that
   records which source commit they were built from - and with the `.env` beside it, which is how the host
   is configured: the app reads that file out of its own directory when it starts
   (`server/utils/runtimeEnv`), so nothing has to be put into the process's environment.

The build output of a Nuxt server app cannot be served by GitHub Pages: the pages are server-rendered,
and `/api/card`, `/api/character` and `/api/realms` are real endpoints that call the Blizzard API with
your client secret. A Node process has to run the build — hence the FTP upload.

### FTP settings

The upload reads these from the environment, falling back to the gitignored `.env`:

```bash
FTP_SERVER=www.example.com
FTP_USERNAME=...
FTP_PASSWORD=...
FTP_PATH=/data02/virt32423/domeenid/www.example.com/heroofazeroth/
# FTP_PORT=21        # optional, 21 by default
FTP_SECURE=explicit  # FTPS on port 21 (false, explicit or implicit)
FTP_INSECURE=true    # waive the certificate check when the host's certificate
                     # is issued for another name (shared hosting, usually)
# FTP_TLS_MAX=1.2    # highest TLS version for FTPS (1.2, because 1.3 data
                     # connections tend to be aborted by these hosts)
```

`FTP_PATH` is the directory the server runs from, and it receives the *contents* of `.output`:
`server/`, `public/` and `nitro.json` land directly inside it.

The `.env` travels beside them as a separate file: the local one, with its `FTP_*` settings left out - those
are the credentials of the transfer itself, and the directory they would land in is the one the site is
served from. It is only sent when it says something different from the copy the host already has - a
configuration that rarely changes should not be re-uploaded, and should not be reported as a change (the
restart note beside it is only true when something moved). When it does change, the copy that was there is
kept as `.env.previous`, so a bad configuration can be put back. `--env-all` sends the file verbatim,
`--no-env` sends none of it, and `--env-force` sends it even when the host already says the same thing.

The transfers are plain `curl` calls
(Windows 10+ ships curl, and it speaks FTP and FTPS); the password travels in a throwaway netrc file, so
it never appears on a command line or in the process list.

### Running what was uploaded

The uploaded directory is the application — no `npm install`, no build tools. `.output` carries its own
`node_modules`, including both the linux-x64 and the win32-x64 resvg binaries:

```bash
cd /data02/virt32423/domeenid/www.example.com/heroofazeroth

node server/index.mjs
```

The `.env` the deploy leaves in that directory is what configures the site: `NUXT_BLIZZARD_CLIENT_ID`,
`NUXT_BLIZZARD_CLIENT_SECRET`, `NUXT_SESSION_PASSWORD` - and `NUXT_TRUST_PROXY` when the host's own web
server overwrites `X-Forwarded-For`. The file is read out of that directory when the process starts
(`server/utils/runtimeEnv`), so a variable set in the process's own environment still wins over it.

The variables are read per request (`useRuntimeConfig()`), so changing them needs a restart, not a
rebuild — and shipping a new version is another `npm run deploy`.

### Running it in Docker

A trial copy can be built and run from a checkout with Docker Desktop (WSL 2 backend) — the image runs
the same `npm run build` the FTP upload does, and serves the same Nitro `node-server` entry:

```bash
docker compose up --build -d     # http://localhost:3100
docker compose logs -f           # [env], [data] and the requests
docker compose down              # stop it; the `hoa-data` volume keeps the database
```

`Dockerfile` builds on Node 24 / Debian: the pinned `linux-x64` builds of `sharp` and
`@resvg/resvg-js` in `package.json` are glibc ones (Alpine would need the musl builds), and
`node:sqlite` needs Node 22.5 or newer. `docker-compose.yml` reads the checkout's `.env` at run time,
so the credentials never enter the image, publishes port `3100` (override with `HOST_PORT=3200`), and
mounts the named volume `hoa-data` at `/data` — that is `NUXT_DATA_DIR`, the directory
`server/utils/appPaths` otherwise puts beside the built server.

A fresh volume is an empty site: the schema is created on first use (`server/utils/db.ts`), and looking
a character up fills the table. To start from the data this checkout already has, stop the container
first — the process must not be writing while the files are copied — and copy them into the volume:

```powershell
docker compose down
docker run --rm -v hero-of-azeroth_hoa-data:/data -v "${PWD}/server/data:/seed:ro" `
  node:24-bookworm-slim sh -c "cp -a /seed/. /data/ && chown -R node:node /data"
docker compose up -d
```

The trial pages still load the production tag manager (`NUXT_PUBLIC_GTM_ID` defaults to it); set that
variable to a container of your own if the trial should not be counted.

The same files run on an Apple Silicon Mac under OrbStack. The base image is multi-arch, and the
`--force` in the `Dockerfile`'s `npm ci` is what lets an arm64 machine install the x64 binaries this
project pins for the FTP host, while `sharp` and `@resvg/resvg-js` supply their own arm64 binaries
through their optional dependencies — so the build on an M1 is native `linux/arm64`, nothing is
emulated. OrbStack publishes forwarded ports to the LAN by default and gives the container the domain
`hero-of-azeroth.orb.local`, so on the Mac itself the site opens without a port number. To make the
Mac a small server rather than a session:

```bash
orb config set app.start_at_login true   # OrbStack comes back after a reboot
```

`restart: unless-stopped` in the compose file then brings the container back with it. The build is
the one thing that needs more than OrbStack's default 4 GB: Nitro's server bundle peaks just above
2 GB and the engine aborts it with SIGABRT, so the builder stage raises Node's heap to 3 GB
(`NODE_OPTIONS` in the `Dockerfile`) and the VM needs room for it — `orb config set memory_mib 6144`,
then `orb stop` and start the app again, because the setting is applied on the next start. The
running site itself stays around 250 MB. Signing in with
Battle.net needs the address Battle.net knows: a reader on the LAN reaches the site by the Mac's name,
but Blizzard accepts a plain `http` callback only on `localhost`, so the way to sign in from another
machine is a tunnel — `ssh -L 3100:localhost:3100 <mac>` and then `http://localhost:3100`.

### Driving the Mac from this checkout

`scripts/docker-mac.ps1` builds and runs the site on a Mac on the same network, from Windows:

```powershell
./scripts/docker-mac.ps1 up -Remote 192.168.8.97 -User <mac-account>
./scripts/docker-mac.ps1 logs
```

It creates the `macmini` Docker context over SSH once, then sends this checkout's sources to the
Mac's own engine on every run — the Mac needs no clone, no git and no `node_modules`, and the `.env`
is read here, so the credentials never sit on it. Remote Login has to be on, and `docker` has to be
reachable in a non-interactive SSH session (add `export PATH="/usr/local/bin:$HOME/.orbstack/bin:$PATH"`
to `~/.zshenv` on the Mac if it is not). Windows has no mDNS resolver, so the Mac's Bonjour name does
not resolve from it: pass the address, `-Remote 192.168.8.97`. The context is created once, so every
later run can leave `-Remote` out. `rebuild`, `down`, `ps` and `check` are the other actions.

### Living inside the host's memory

The host gives the process **416 MiB**, and the Node heap is the smaller half of what it spends. A card is drawn by
native libraries — `@resvg/resvg-js` and `sharp`/libvips — which allocate *outside* the V8 heap, where the garbage
collector neither sees them nor is woken by them; and glibc keeps what they release in per-thread arenas rather than
handing it back to the kernel. That is why RSS climbs while cards are being drawn and then **settles instead of
falling**: the freed memory is real, but the process does not return it.

Two settings keep that high-water mark low, and both have to be in the environment the process is started with —
neither can be changed from inside it:

```bash
MALLOC_ARENA_MAX=2 NODE_OPTIONS=--max-old-space-size=192 node server/index.mjs
```

- `MALLOC_ARENA_MAX=2` — one arena per thread is one arena per rendering thread, each holding its own freed blocks.
  Two of them is all this process needs, and it is the single largest difference to RSS.
- `--max-old-space-size=192` — the heap is capped below the ceiling, so V8 collects before the *process* is the thing
  that runs out. The site's own caches are byte-bounded (`server/utils/lruStore`, `server/utils/swrCache`), so this is
  a guard rather than a diet.

What a card costs is bounded on purpose: the render is rasterised **once** (text widths are measured in a half-scale
document whose ink is read straight off the rasteriser's pixels), the model is cut out of Blizzard's 1600×1200 canvas
and only that crop is embedded, the class backdrop is downscaled to the width it is drawn at and kept per class, and
one rasteriser pass runs at a time (`server/utils/taskGate`). The finished JPEG and the heavy shelves live in SQLite
(`server/utils/sqliteStore`), not in the heap.

### Where the site keeps its data

Everything the site writes lives in one directory: the SQLite database (`server/data/hoa.db`) — which holds
the hall of fame's own table beside everything else the site knows — and the character
index the sitemap is built from (`server/data/characters.json`). It is resolved by `server/utils/appPaths` from the directory the app
*runs* from, not from the working directory of the process — so a panel or a process manager that starts
`node /somewhere/app/server/index.mjs` with a working directory of its own reads and writes exactly the same
files as `cd /somewhere/app && node server/index.mjs`. A data directory that follows the working directory
is how a site's own front page comes up empty while its data sits untouched in the directory the previous
process used. `NUXT_DATA_DIR` overrides the location outright, and the process prints the directory it
settled on while it starts (`[data] ...`, beside the `[env]` line).

Every `npm run deploy` reads that directory back into `backups/<date>/` before it uploads anything: the files
under `server-data/`, and one `server-data.zip` holding that folder plus a `backup.json` saying where it came
from, which commit it was taken beside and what is inside (`--no-backup` skips it). A store the database has
taken over — `leaderboard.json`, the hall of fame's table before it moved into `hoa.db` — is left out of the copy
once the copied database really holds its rows, and kept until then, so the deploy that first ships the move still
carries it. None of it is part of `.output` and nothing in the deploy deletes it — which is why a copy is worth
taking: it is the only thing in this repository's world that a build cannot produce again.

To put a copy back, **stop the site first**: the files are written by the running process - the database is open
for as long as it runs, and the character index is rewritten as characters are read - so they have to be copied
over while nothing is running, and the site started again afterwards to read them. Unpack the archive with
`tar -x -f backups/<date>/server-data.zip` it holds the same `server-data/` folder the rest of the copy is in.

### Deploy options

```bash
npm run deploy                     # build, GitHub, FTP
npm run deploy -- --dry-run        # report what would happen, change nothing
npm run deploy -- --no-build       # upload the `.output` that already exists
npm run deploy -- --no-github      # sources stay local
npm run deploy -- --no-ftp         # sources only
npm run deploy -- --prune          # also delete remote files this build no longer has
npm run deploy -- --verify         # read every directory back and report missing files
npm run deploy -- --no-env         # leave the remote `.env` alone
npm run deploy -- --env-all        # send that `.env` verbatim, `FTP_*` settings included
npm run deploy -- --env-force      # send it even when the host already says the same thing
npm run deploy -- --no-backup      # do not read the site's data directory back first
npm run deploy -- -m "Fix the card"
```

`npm run deploy:ftp` uploads without building and without touching GitHub; `npm run deploy:github`
pushes the sources only.

`--verify` reads the upload back: every file has to be present, at the size it was built with (one
request per file, so it is the slow part of a deploy). Transfers are retried a few times as well,
because FTP hosts throttle a burst of uploads and answer 451 for a while.

`--prune` never deletes directories, and only deletes files inside `public/` and `server/chunks/` — the
content-hashed bundles that change on every build — so a mistyped `FTP_PATH` cannot wipe anything else.

---

