# Audio Download with yt-dlp

Parallel MP3 playlist downloader using `yt-dlp` and `gnu parallel`. Downloads ~1,000 songs in ~15 minutes with full metadata, cover art, and optional VPN routing.

---

## Documentation

| File | Description |
|---|---|
| [Audio-Download-with-yt-dlp.md](Audio-Download-with-yt-dlp.md) | Main guide: tool install, playlist discovery, parallel download, audio normalization, cleanup, and stats |
| [vpn-yt-dlp.md](vpn-yt-dlp.md) | Routes `yt-dlp` through isolated WireGuard SOCKS5 proxies via `wireproxy` — supports multiple simultaneous VPN connections with `gnu parallel` |
| [vpn_surfshark.md](vpn_surfshark.md) | Surfshark WireGuard setup: cookie-based config download, per-server profile generation, deployment to `/etc/wireguard/` |
| [yt-dlp_Cheats.md](yt-dlp_Cheats.md) | `yt-dlp` command reference and useful one-liners |

---

## Quick Start

```bash
# Install
wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O ~/.local/bin/yt-dlp
chmod a+x ~/.local/bin/yt-dlp

# Find playlists and download
echo "salsa 2025" \
| parallel --ungroup --silent 'yt-dlp --playlist-end 10 --flat-playlist --simulate --match-filter "id~=^PL" --print id "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' \
| parallel --ungroup --silent 'yt-dlp --ignore-errors --no-warnings --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s" --flat-playlist {}' \
| parallel --colsep ';' --ungroup --silent 'printf "yt-dlp --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata {1} -o \"$HOME/Downloads/CarPlaylist/{=2 s/[^a-zA-Z0-9 .-]//g =}/{=3 s/[^a-zA-Z0-9 .-]//g =}.mp3\"\n"' \
| parallel --max-procs 16 --bar --eta
```

---
## License
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
