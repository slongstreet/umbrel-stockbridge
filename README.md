# Stockbridge — Umbrel Community App Store

Personal [community app store](https://github.com/getumbrel/umbrel-community-app-store)
that packages [Stockbridge](https://github.com/slongstreet/stockbridge) (private
repo) for umbrelOS. The store contains one app: `longstreet-stockbridge`.

The app runs two containers — the FastAPI backend and the Next.js frontend —
fronted by Umbrel's `app_proxy`, which adds the Umbrel password gate. All
durable state (SQLite DB, `.env`, snapshot backups) lives in
`~/umbrel/app-data/longstreet-stockbridge/data/`, so it survives app updates
and is covered by Umbrel's own backup tooling.

## Images

umbreld cannot pull private registry images (its Engine-API pull sends no
auth — [getumbrel/umbrel#2127](https://github.com/getumbrel/umbrel/pull/2127)),
so images are served from a **loopback-only registry on the Umbrel itself**
(`stockbridge-registry` container, `127.0.0.1:5000`, named volume
`stockbridge-registry`, `restart=always`). The compose references
`localhost:5000/stockbridge-{backend,frontend}:<version>`; the app code
never leaves the box.

- `stockbridge-backend` — built from `backend/Dockerfile`
- `stockbridge-frontend` — built from `frontend/Dockerfile`
  with `--build-arg BACKEND_URL=http://longstreet-stockbridge_backend_1:8000`
  (the `/api` rewrite proxy is baked into the Next build, so the frontend
  image is specific to this app id)

Tags match the Stockbridge app version (`frontend/package.json`). Images are
built `linux/amd64` on the dev Mac and shipped over SSH:

```bash
docker save IMAGE... | gzip | ssh umbrel@umbrel-2.local 'gunzip | docker load'
# then on the Umbrel: docker tag ... localhost:5000/... && docker push
```

(One-time registry setup:
`docker run -d --name stockbridge-registry --restart=always -p 127.0.0.1:5000:5000 -v stockbridge-registry:/var/lib/registry registry:3`)

## Install (once)

1. **Add this store**: Umbrel dashboard → App Store → ⋯ → Community App
   Stores → add `https://github.com/slongstreet/umbrel-stockbridge`.

3. **Install Stockbridge** from the store, then stop it and fill in config:

   ```bash
   cd ~/umbrel/app-data/longstreet-stockbridge/data
   sudo cp .env.example .env
   sudo nano .env      # eBay keys, FRONTEND_URL, callback URL, LLM keys
   ```

   Restart the app from the dashboard.

4. **Migrate data** (moving from another machine): use Stockbridge's
   Settings → Backup & restore. Export the archive on the old machine, then
   Import it in the freshly installed app. eBay tokens ride along in the DB;
   `.env` secrets deliberately do not — that's step 3.

## HTTPS / eBay OAuth (Tailscale)

eBay's OAuth callback requires a real-TLD HTTPS URL. With the Tailscale app
installed on the Umbrel:

```bash
docker exec tailscale_web_1 tailscale serve --bg --https=443 http://localhost:8471
```

gives `https://umbrel-2.<tailnet>.ts.net/` a real Let's Encrypt cert (enable
MagicDNS and HTTPS certificates in the Tailscale admin console first; the
container is host-networked, so `localhost:8471` is the host's app_proxy
port). Use that origin as `FRONTEND_URL` and `<origin>/api/ebay/callback` as
`EBAY_OAUTH_CALLBACK_URL`, and register the callback against the RuName in
the eBay developer console.

## Releasing an update

1. In the stockbridge repo: bump `frontend/package.json` version, build both
   images for `linux/amd64` with the new tag (frontend with the `BACKEND_URL`
   build arg above), and ship them into the Umbrel's registry (see Images).
2. Here: update the image tags in
   `longstreet-stockbridge/docker-compose.yml` and `version` in
   `umbrel-app.yml`, commit, push.
3. Umbrel dashboard → the app shows an update — click it.
