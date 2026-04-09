# Audio Download with yt-dlp
A high-performance, parallelized CLI pipeline to download YouTube playlists as MP3 files. By combining the power of yt-dlp and GNU Parallel, this tool can download ~1,000 songs in ~15 minutes—complete with full metadata and embedded cover art.   
It also includes advanced guides for routing downloads through isolated VPN connections (WireGuard/SOCKS5) for maximum privacy and bypassing geo-restrictions.   
## Features   
- Blazing Fast: Downloads multiple files simultaneously using GNU Parallel (up to 16+ concurrent processes).  
- Perfect Metadata: Automatically embeds ID3 tags, album art, and thumbnails.  
- High Quality: Extracts audio and converts it directly to MP3 (Quality 5 / VBR ~130kbps).  
- Auto-Discovery: Includes a pipeline to automatically search YouTube for playlists based on a keyword.  
- VPN Integration: Advanced setup to route yt-dlp through isolated WireGuard proxies (e.g., Surfshark) using wireproxy.  
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
# Install
```bash
wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O ~/.local/bin/yt-dlp
chmod a+x ~/.local/bin/yt-dlp
```
This single pipeline searches for a topic, finds playlists, cleans up the filenames, and downloads the audio in parallel.
Change "salsa 2025" to whatever you want to search for, and $HOME/Downloads/CarPlaylist to your desired output folder.
```bash
echo "salsa 2025" | \
# Step 1: Search YouTube for playlists
parallel --ungroup --silent -- 'yt-dlp --playlist-end 20 --flat-playlist --simulate --match-filter "id~=^PL" --print id "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' | \
# Step 2: Get video IDs, Playlist names, and Titles
parallel --ungroup --silent -- 'yt-dlp --ignore-errors --no-warnings --flat-playlist --print "%(id)s§%(playlist)s§%(title)s" "https://www.youtube.com/playlist?list={}"' | \
# Step 3: Filter out deleted/private videos and sanitize filenames
grep -vE "Deleted video|Private video" | sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' | \
# Step 4: Generate the yt-dlp download commands
parallel --ungroup --silent --colsep § -- 'echo nice ionice -c 3 yt-dlp --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata {1} -o "$HOME/Downloads/CarPlaylist/{2}/{3}.mp3"' | \
# Step 5: Execute all download commands with 16 parallel processes
parallel --ungroup --max-procs 16
```
---
## License
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
