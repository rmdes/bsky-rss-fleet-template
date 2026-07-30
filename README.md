# bsky.rss fleet template

Deploy your own fleet of Bluesky RSS-poster bots — many independent bots
running out of one process, instead of one container per bot — using the
prebuilt image published from [`rmdes/bsky.rss`](https://github.com/rmdes/bsky.rss).

This repo is deploy-only: a `docker-compose.yml`, an example config tree, and
this README. There's no application source here — if you want the
architecture, the legacy-import/rollback story, or to build the image
yourself instead of pulling it, see
[`rmdes/bsky.rss`'s `documentation/fleet.md`](https://github.com/rmdes/bsky.rss/blob/main/documentation/fleet.md).

## Prerequisites

- Docker and Docker Compose
- At least one Bluesky account (an app password, not your real password —
  create one at Settings → App Passwords) per bot you want to run
- An RSS/Atom feed URL per bot

## Quickstart

```bash
git clone https://github.com/rmdes/bsky-rss-fleet-template.git
cd bsky-rss-fleet-template

cp -r config.example config
# Edit config/fleet.json if you want different stagger timing or limiter caps
# (the defaults are reasonable for a first run).

# For each bot you want to run, either edit the two example bot directories
# in place or add more under config/bots/<your-bot-id>/:
#   config/bots/<bot-id>/bot.json     - identifier, instanceUrl, feedUrl, secretKey
#   config/bots/<bot-id>/config.json  - post template, embed/language settings

# Real app passwords go in secrets/bsky-fleet.json, keyed by each bot's
# secretKey (see config.example/secrets/bsky-fleet.json for the shape):
cat > secrets/bsky-fleet.json <<'EOF'
{
  "your-bot-id": "your-real-app-password"
}
EOF
chmod 600 secrets/bsky-fleet.json

docker compose up -d
docker compose logs -f
```

Look for `Loaded N bot(s), 0 config error(s)` followed by `[AUTH] Activated`
lines, staggered a few seconds apart per bot (this is deliberate — it avoids
hammering Bluesky's login endpoint if you're running many bots at once).

## What the compose file does

```yaml
environment:
  - DRY_RUN=false                                  # set to "true" to test without publishing
  - FLEET_CONFIG_ROOT=/build/config                 # -> ./config on the host
  - FLEET_SECRETS_PATH=/build/secrets/bsky-fleet.json # -> ./secrets on the host
  - FLEET_DATA_ROOT=/build/data/fleet                # -> ./data on the host (per-bot state)
  - FLEET_LOCK_PATH=/build/data/fleet/fleet.pid      # prevents two instances running at once
```

`./data` accumulates one SQLite database per bot (dedup history, the current
post queue, the AT-Proto session) — back it up like you would any other
stateful volume; it's what lets a restart resume cleanly instead of starting
cold.

`DRY_RUN=true` is worth running first: the whole pipeline runs (feed poll,
Open Graph scrape, image fetch/resize) except the final publish step, which
just logs what it would have posted.

## Updating

New releases are published to `ghcr.io/rmdes/bsky.rss` as both a specific
version tag and `:latest`. To update:

```bash
docker compose pull
docker compose up -d
```

`stop_grace_period: 45s` gives the fleet daemon time to finish any in-flight
post and shut down cleanly (not just get killed) before Compose replaces the
container.

## Coming from an existing legacy (one-container-per-bot) deployment?

See [`rmdes/bsky.rss`'s `documentation/fleet.md`](https://github.com/rmdes/bsky.rss/blob/main/documentation/fleet.md)
for the importer (migrates your existing `docker-compose.yml`-per-bot setup
into this fleet's config + state), the rollback exporter (reverses it, in
case you need to fall back), and the cutover sequence end-to-end.
