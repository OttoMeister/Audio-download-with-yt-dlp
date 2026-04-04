# vpn-yt-dlp

Routes `yt-dlp` downloads through an isolated WireGuard SOCKS5 proxy via `wireproxy` — leaving all other system traffic untouched. Supports multiple simultaneous connections; pairs well with `gnu parallel`.

---

## Install

```bash
cp vpn-yt-dlp ~/.local/bin/ && chmod a+x ~/.local/bin/vpn-yt-dlp
```

**Requires:** `wireproxy`, `yt-dlp`, `netcat` — WireGuard configs in `/etc/wireguard/*.conf`

---

## Usage

```bash
vpn-yt-dlp <vpn|mode> [yt-dlp args...]
```

| Mode | Description |
|---|---|
| `rand` | Random config from all available |
| `rand-eu` `rand-ap` `rand-am` `rand-us` `rand-ea` | Random from that region |
| `us-slc` | Specific named config |

All standard `yt-dlp` flags pass through unchanged.

---

## Examples

```bash
# Single download via random VPN
vpn-yt-dlp rand https://www.youtube.com/watch?v=1nnatyEvxQU

# Specific server
vpn-yt-dlp us-slc https://www.youtube.com/watch?v=1nnatyEvxQU
```

---

## Batch Download (Parallel)

**1. Build download list**

Search YouTube for playlists, then expand each to individual track commands:

```bash
echo "Banda Sinaloa" \
| parallel --ungroup --silent \
  'yt-dlp --playlist-end 20 --flat-playlist --simulate \
   --match-filter "id~=^PL" --print id \
   "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' \
| parallel --ungroup --silent \
  'yt-dlp --ignore-errors --no-warnings \
   --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s" \
   --flat-playlist {}' \
| parallel --colsep ';' --ungroup --silent \
  'printf "nice ionice -c 3 vpn-yt-dlp rand \
   --extract-audio --audio-format mp3 --audio-quality 5 \
   --embed-thumbnail --embed-metadata {1} \
   -o \"$HOME/Downloads/CarPlaylist8/{=2 s/[^a-zA-Z0-9 .-]//g =}/{=3 s/[^a-zA-Z0-9 .-]//g =}.mp3\"\n"' \
> do1.sh
```

**2. Run — 16 parallel workers, shuffled across all regions**

```bash
cat do1.sh | shuf | parallel --max-procs 16 --ungroup --silent
```

---

## VPN Status Check

Tests every config against a sample video and reports which servers are blocked:

```bash
vpn-check-status.sh
```

```
ID              STATUS          COUNTRY              MESSAGE
----------------------------------------------------------------------
us-slc          WORKING         United States        OK
de-fra          BLOCKED         Germany              Bot-Check
nl-ams          ERROR           Netherlands          Other Error
sg-sin          CRASHED         Singapore            Proxy Fail
```

Exit states: `WORKING` · `BLOCKED` (bot-check) · `ERROR` (other) · `CRASHED` (proxy failed to start)
