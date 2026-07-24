# Zeam mirror Postgres on p73

A Docker container (`zeam-mirror-postgres`, `postgres:15-alpine`) that stands in for prod
Redshift so the Hetzner-deployed `zeam-personalization` app can run against real,
point-in-time data on this box. Part of the **zeam-prod-mirror** effort; full context lives in
`~/git/zeam/zeam-notes/_inbox/zeam-prod-mirror/` (`SERVING.md`).

- **Compose:** `~/git/zeam/zeam-notes/_inbox/zeam-prod-mirror/scripts/serving/docker-compose.mirror.yml`
- **Data:** Docker named volume `zeam-mirror_pgdata` on **ext4** (under `/var/lib/docker` on `/`),
  never on `/media/Arxv` (ntfs — no real fsync). Removing the *container* is safe; removing that
  *volume* destroys the loaded tables (67.6M-row log slice + the `audience_*` build).
- **Creds/db:** `zeam_user` / `zeam_password` / `zeam_db` (non-prod; matches the app's committed
  compose contract so the deploy repoint is just env vars).
- **Port:** `0.0.0.0:5432` — reachable on loopback (on-box loader), the tailnet IP
  `100.70.130.87` (the Hetzner app dials this), and the home LAN.
- **Helper:** `bin/zeam-mirror` (`up | down | status | disable | enable | logs | psql`).

## Boot behaviour

It **auto-starts on boot** by design, via two independent mechanisms:

1. **dockerd** starts at boot — `/etc/rc*.d/S01docker` (antiX/runit sysv compat).
2. The container carries **`restart: unless-stopped`**, so dockerd brings it back up on start
   unless it was *manually* stopped.

## The 2026-07-24 reboot lesson

After a p73 reboot the mirror was **down** even though `restart: unless-stopped` was already set.
Root cause was a **boot-order race on the port bind**: the compose used to publish on the
*tailnet IP* (`100.70.130.87:5432`). On boot, dockerd tried to (re)start the container before
`tailscale0` had come up with that IP, the bind failed, and the container gave up and stayed dead.
The `pgdata` volume was fine the whole time — only the container was down.

**Fix (already applied):** bind to `0.0.0.0:5432` instead of the tailnet IP. `0.0.0.0` is always
available regardless of tailscale, so there is no interface to race. Tradeoff: the port is now
also reachable on the home LAN (`wlan0`), which is acceptable — p73 sits behind home NAT and this
is derived, non-sensitive data with non-prod creds. The Hetzner app still reaches it over the
tailnet IP (a subset of `0.0.0.0`).

> If it's ever down after a reboot again, first check the container, not the data:
> `bin/zeam-mirror status`. If the container is stopped, `bin/zeam-mirror up`. If tailscale is
> DERP-only (`tailscale ping <peer>` shows "via DERP", no direct), that's a separate p73 issue —
> `sudo sv restart tailscaled` — but it no longer affects the Postgres port since the 0.0.0.0 bind.

## Stopping it from booting up (if you ever want to)

The mirror is only needed while actively validating a `zeam-personalization` change; it holds a
warm 4 GB `shared_buffers` and a big log slice, so you may want it off on a normal day.

**The one command:**

```sh
bin/zeam-mirror disable
```

That does two things so it stays down permanently:

- `docker update --restart=no zeam-mirror-postgres` — removes the auto-restart policy, so dockerd
  won't bring it up on boot even if it had been running.
- `docker stop zeam-mirror-postgres` — stops it now.

The **data volume is untouched** — this is pause, not delete. Bring it back with:

```sh
bin/zeam-mirror enable      # re-arms restart:unless-stopped and starts it
```

### Manual equivalents (no helper script)

```sh
# stop from booting
sudo docker update --restart=no zeam-mirror-postgres
sudo docker stop zeam-mirror-postgres

# re-enable
sudo docker update --restart=unless-stopped zeam-mirror-postgres
sudo docker start zeam-mirror-postgres
```

### Heavier hammers (rarely needed)

- **Just this container, forever, and forget it:** `sudo docker rm -f zeam-mirror-postgres`
  (volume kept; recreate later with `bin/zeam-mirror up`).
- **All of Docker off at boot** (affects everything Docker on p73, not just the mirror):
  `sudo rm /etc/rc*.d/S01docker` (re-add with `sudo update-rc.d docker defaults`). Overkill for
  just the mirror — prefer `zeam-mirror disable`.
