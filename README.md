# StreamForge Hub — source registry template

Publish this repository (or a copy) and point the hub at `registry.json`. After
that, channels, movies, series and radio are managed **here** — by editing files
on GitHub — instead of by retyping entries in the Android settings screen.

Contents:

```
registry.json                 the manifest the hub reads (entry point)
providers/*.json              per-provider entry lists
playlists/live.m3u            an M3U playlist of live channels
playlists/movies.m3u          an M3U playlist of movies (media_type: movie)
playlists/radio.m3u           an M3U playlist of radio (media_type: radio)
playlists/series.json         a series collection, episodes as direct URLs
collections/movies.json       a movie collection (JSON)
collections/series.json       a series collection, episodes as provider references
schemas/registry.schema.json  JSON Schema for editor validation
```

The files under `providers/`, `playlists/` and `collections/` are **examples**.
Replace the titles, ids and URLs with your own. Entries whose title starts with
`Replace me:` are placeholders and must be changed (or their source disabled)
before you rely on them.

## 1. Publish the repository

1. Create a **public** GitHub repository (a private one cannot be read: the hub
   holds no GitHub token, by design — see "What must never be in this
   repository").
2. Upload these files, keeping the directory layout (`registry.json` at the
   root).
3. Copy the URL of the raw manifest:

   ```
   https://raw.githubusercontent.com/<OWNER>/<REPO>/<BRANCH>/registry.json
   ```

`raw.githubusercontent.com`, `github.com`, `objects.githubusercontent.com`,
`gist.githubusercontent.com` and `codeload.github.com` are accepted out of the
box. A different host must be added under **Extra repository hosts** in the hub
first, and a private, loopback or link-local address is refused even then.

## 2. Connect the hub

In the hub: **Remote Sources** →

1. paste the `registry.json` URL into the URL field and press *Save URL*;
2. press *Import preview* to see, without changing anything, exactly what the
   document would add, update and remove;
3. press *Sync now* to apply it. The hub reports `applied: N added, M updated,
   K removed`.

*Automatic synchronization* is on by default and refreshes the registry on the
same schedule as the provider catalogs. *Last-known-good rollback* re-applies
the last document that was successfully applied, from the device, without a
network — use it if a change to the repository turns out to be wrong.

`Sync now` and the automatic refresh are the same code path, so there is nothing
to configure for either.

## 3. Where the entries appear

Imported entries become ordinary hub content:

| Registry entry | Xtream namespace | M3U export |
|---|---|---|
| `media_type: live` | `get_live_streams` | `group-title` = source category |
| `media_type: movie` | `get_vod_streams` | movie section |
| `media_type: series` (+ `seasons`) | `get_series` / `get_series_info` | series section |
| `media_type: radio` | live namespace | radio group |

They are served through the hub's existing gateway (port 8890) and its existing
M3U/Xtream routes; nothing new is exposed. A client (TiviMate, IPTVnator, VLC)
needs a playlist refresh to see an update.

**The numeric Xtream id of an entry never changes** when you edit unrelated
entries. It is derived from the entry's own identity — provider, canonical
reference and media type — never from its position in a file or the repository
revision. Adding one channel therefore does not renumber the others.

## 4. The manifest

```json
{
  "schema_version": 1,
  "name": "My StreamForge registry",
  "updated": "2026-10-13",
  "sources": [
    { "id": "live-channels", "type": "m3u",     "name": "Live TV",
      "category": "Live", "path": "playlists/live.m3u" },
    { "id": "movies",        "type": "m3u",     "name": "Movies",
      "category": "Movies", "media_type": "movie", "path": "playlists/movies.m3u" },
    { "id": "shows",         "type": "series",  "name": "Series",
      "path": "collections/series.json" },
    { "id": "rutube",        "type": "provider","provider": "rutube",
      "path": "providers/rutube.json" },
    { "id": "direct",        "type": "hls",     "items": [
        { "reference": "https://cdn.example.com/live/a.m3u8",
          "media_type": "live", "title": "A channel" } ] }
  ]
}
```

| Field | Meaning |
|---|---|
| `schema_version` | Must be exactly `1`. A newer version is refused rather than half-understood. |
| `id` | Stable source id. It **owns** its entries: disabling or deleting the source removes only its own rows. |
| `type` | `m3u`, `movies`, `series`, `provider` or `hls`. |
| `provider` | Required for `type: provider`. `rutube`, `dailymotion`, `snrt`, `forja`, `iptvorg`, `my2m`, `quran` or `bilibili`. |
| `path` | A repository-relative path. Required for `m3u`/`movies`/`series`, optional for `provider`/`hls`. |
| `media_type` | The default media type of the source's entries: `live`, `movie`, `series`, `radio` or `auto`. |
| `enabled` | `false` keeps the source in the file but contributes nothing. |
| `items` | Inline entries, instead of `path`. Declaring both is refused. |

**Only the keys listed above (and in `schemas/registry.schema.json`) are
accepted.** An unknown key refuses the whole document — deliberately. A silently
ignored `http_headers` or `path` typo would otherwise look like an entry that
works.

### Entry fields

`reference` (or `url`) is required and is either:

* a **media URL** — `https://…`; any public host is allowed, but never a private,
  loopback, link-local or `.local` address, and never a URL with credentials; or
* a **provider reference** — a provider page URL (`https://www.dailymotion.com/video/x8ab`)
  or the internal `provider://content-id` form (`rutube://1bf2ee…`, `my2m://2m_monde`).

A provider reference is never fetched as media. The hub asks that provider's
plugin to resolve it **at playback time**, so an expiring token is generated per
playback instead of being frozen into this repository.

Optional: `media_type`, `title`, `description`, `poster`/`logo`,
`category`/`group`, `duration`, `season`, `episode`, `series_id`, `series_title`.

### What must never be in this repository

This repository is public, and the hub will never store a secret beside it.

* **No provider credentials** (username, password, token, cookie).
* **No signed or expiring playback URLs** — they will be posted publicly, and
  they expire. Reference the provider instead.
* **No URLs with embedded credentials** (`https://user:pass@…`): refused.
* **No attempts to configure the hub.** A registry can only express content. It
  cannot name a plugin class, set an HTTP header, change a port, grant a
  permission or execute anything — there is no key for any of that, and an
  unknown key refuses the document.

## 5. Limits

Per manifest: 64 sources, 5000 entries per source, 20000 entries in total,
512 KiB for `registry.json`, 1 MiB per referenced file, 4 MiB in total, 3
redirects, 15 s connect and 30 s read timeouts. Exceeding a limit refuses the
sync and keeps the previously applied catalog; nothing is truncated, because a
truncated import looks like a successful one.

## 6. Offline behaviour

The last successfully applied document is stored on the device. If GitHub is
unreachable — no network, a typo in the URL, a rate limit — the hub keeps
listing and playing everything it already knows, reports the failure, and
retries later. A failed sync never deletes a row.

## 7. Validating locally

Any JSON Schema validator (or your editor, via the `$schema` in
`schemas/registry.schema.json`) catches typos in key names before you commit.
The strongest check is the hub itself: *Import preview* reports every problem in
the document at once, without changing the catalog.
