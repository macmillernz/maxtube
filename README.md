# MaXTube

A self-hosted web UI for `yt-dlp` — fork of [MeTube](https://github.com/alexta69/metube) with additional features.

Supports downloading media from YouTube and [hundreds of other sites](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md).

## What's different from MeTube

| Feature | MeTube | MaXTube |
|---------|--------|---------|
| Multi-URL input | One URL at a time | Paste a comma-separated list — each URL is queued as a separate download |
| Built-in player | — | Play completed downloads in a bottom media bar without leaving the page |

## Branches

| Branch | Purpose |
|--------|---------|
| `main` | Tracks upstream MeTube master. Updated automatically every day via GitHub Actions. |
| `custom` | MaXTube features on top of `main`. Auto-merged after each upstream sync. |

The Docker image is built from the `custom` branch.

---

## Run with Docker

```bash
docker run -d \
  -p 8081:8081 \
  -v /path/to/downloads:/downloads \
  ghcr.io/macmillernz/maxtube:latest
```

## Run with Docker Compose

```yaml
services:
  maxtube:
    image: ghcr.io/macmillernz/maxtube:latest
    container_name: maxtube
    restart: unless-stopped
    ports:
      - "8081:8081"
    volumes:
      - /path/to/downloads:/downloads
    environment:
      - PUID=1000
      - PGID=1000
```

## Unraid

An Unraid Community Applications template is included at [`unraid-template.xml`](unraid-template.xml).

To install manually:
1. In the Unraid UI go to **Apps → Install from URL**.
2. Paste `https://raw.githubusercontent.com/macmillernz/maxtube/main/unraid-template.xml`.

Recommended paths:

| Container path | Host path |
|---------------|-----------|
| `/downloads` | `/mnt/user/downloads/maxtube` |

---

## Configuration

All options are set via environment variables.

### Identity

| Variable | Default | Description |
|----------|---------|-------------|
| `PUID` | `1000` | User ID MaXTube runs as |
| `PGID` | `1000` | Group ID MaXTube runs as |
| `UMASK` | `022` | Umask for created files |

### Download behaviour

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_CONCURRENT_DOWNLOADS` | `3` | Maximum simultaneous downloads |
| `DELETE_FILE_ON_TRASHCAN` | `false` | Delete file from disk when trashed in the UI |
| `CLEAR_COMPLETED_AFTER` | `0` | Seconds before completed entries auto-clear (0 = disabled) |
| `DEFAULT_OPTION_PLAYLIST_ITEM_LIMIT` | `0` | Max playlist items to download (0 = no limit) |
| `SUBSCRIPTION_DEFAULT_CHECK_INTERVAL` | `60` | Minutes between subscription checks |
| `SUBSCRIPTION_SCAN_PLAYLIST_END` | `50` | Max entries to fetch per subscription check |

### Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `DOWNLOAD_DIR` | `/downloads` | Where video files are saved |
| `AUDIO_DOWNLOAD_DIR` | same as `DOWNLOAD_DIR` | Separate directory for audio-only downloads |
| `STATE_DIR` | `/downloads/.metube` | Persistent state files (`queue.json`, etc.) |
| `TEMP_DIR` | `/downloads` | Intermediary download location. Point at an SSD or `tmpfs` for faster muxing. |
| `CUSTOM_DIRS` | `true` | Show a folder dropdown in the UI |
| `CREATE_CUSTOM_DIRS` | `true` | Auto-create subdirectories that don't exist |
| `DOWNLOAD_DIRS_INDEXABLE` | `false` | Make download directories browsable via the web server |

### File naming

| Variable | Default | Description |
|----------|---------|-------------|
| `OUTPUT_TEMPLATE` | `%(title)s.%(ext)s` | yt-dlp filename template for videos |
| `OUTPUT_TEMPLATE_CHAPTER` | `%(title)s - %(section_number)s %(section_title)s.%(ext)s` | Template when splitting by chapters |
| `OUTPUT_TEMPLATE_PLAYLIST` | `%(playlist_title)s/%(title)s.%(ext)s` | Template for playlist downloads |
| `OUTPUT_TEMPLATE_CHANNEL` | `%(channel)s/%(title)s.%(ext)s` | Template for channel downloads |

### yt-dlp options

| Variable | Default | Description |
|----------|---------|-------------|
| `YTDL_OPTIONS` | — | Inline JSON of global yt-dlp options |
| `YTDL_OPTIONS_FILE` | — | Path to a JSON file of global options (auto-reloaded on change) |
| `YTDL_OPTIONS_PRESETS` | — | Inline JSON of named option presets selectable in the UI |
| `YTDL_OPTIONS_PRESETS_FILE` | — | Path to a JSON file of presets (auto-reloaded on change) |
| `ALLOW_YTDL_OPTIONS_OVERRIDES` | `false` | Expose a per-download free-text options field in the UI |

#### Example — global options

```yaml
environment:
  - 'YTDL_OPTIONS={"writesubtitles": true, "subtitleslangs": ["en"], "writethumbnail": true}'
```

#### Example — presets file

Mount a file at `/config/presets.json` and point to it:

```yaml
volumes:
  - /path/to/presets.json:/config/presets.json
environment:
  - YTDL_OPTIONS_PRESETS_FILE=/config/presets.json
```

```json
{
  "sponsorblock": {
    "postprocessors": [
      { "key": "SponsorBlock", "categories": ["sponsor", "selfpromo"] },
      { "key": "ModifyChapters", "remove_sponsor_segments": ["sponsor", "selfpromo"] }
    ]
  },
  "limit-rate": {
    "ratelimit": 5000000
  }
}
```

> **Security:** `ALLOW_YTDL_OPTIONS_OVERRIDES=true` lets anyone with UI access supply arbitrary yt-dlp options, which may allow command execution inside the container. Enable only in trusted environments.

### Web server

| Variable | Default | Description |
|----------|---------|-------------|
| `HOST` | `0.0.0.0` | Bind address |
| `PORT` | `8081` | Listen port |
| `URL_PREFIX` | `/` | Base path (for reverse-proxy sub-path hosting) |
| `PUBLIC_HOST_URL` | — | Override base URL for video download links in the UI |
| `PUBLIC_HOST_AUDIO_URL` | — | Override base URL for audio download links in the UI |
| `CORS_ALLOWED_ORIGINS` | — | Comma-separated allowed origins (required for browser extensions; use `*` for extensions) |
| `HTTPS` | `false` | Enable HTTPS (requires `CERTFILE` and `KEYFILE`) |
| `CERTFILE` | — | Path to TLS certificate |
| `KEYFILE` | — | Path to TLS private key |

### Appearance

| Variable | Default | Description |
|----------|---------|-------------|
| `DEFAULT_THEME` | `auto` | Default UI theme: `auto`, `light`, or `dark` |
| `LOGLEVEL` | `INFO` | Log level: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`, `NONE` |
| `ENABLE_ACCESSLOG` | `false` | Enable HTTP access log |

---

## Reverse proxy

### Nginx

```nginx
location /maxtube/ {
    proxy_pass         http://127.0.0.1:8081/;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection "upgrade";
    proxy_set_header   Host $host;
}
```

Set `URL_PREFIX=/maxtube/` in the container environment.

### Traefik

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.maxtube.rule=Host(`media.example.com`) && PathPrefix(`/maxtube`)"
  - "traefik.http.routers.maxtube.middlewares=maxtube-strip"
  - "traefik.http.middlewares.maxtube-strip.stripprefix.prefixes=/maxtube"
  - "traefik.http.services.maxtube.loadbalancer.server.port=8081"
```

---

## Building locally

The `custom` branch contains the MaXTube feature changes.

```bash
git clone https://github.com/macmillernz/maxtube.git
cd maxtube
git checkout custom
docker build -t maxtube .
```

---

## Credits

MaXTube is a fork of [MeTube](https://github.com/alexta69/metube) by [alexta69](https://github.com/alexta69), which is itself built on [yt-dlp](https://github.com/yt-dlp/yt-dlp).
