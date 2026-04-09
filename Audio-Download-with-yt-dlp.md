# Efficient Parallel MP3 Playlist Download
### True parallel downloads with `yt-dlp` preserving playlist name as directory and full metadata. ~20× speedup — ~1,000 songs in 15 min.
---
## 1. Install Tools
```shell
sudo apt install parallel normalize-audio mp3gain mp3info loudgain mp3check detox fdupes eyed3 exiftool imagemagick id3v2 unzip ionice
# mkdir -p ~/.local/bin && echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O ~/.local/bin/yt-dlp
wget https://github.com/denoland/deno/releases/latest/download/deno-x86_64-unknown-linux-gnu.zip -qO- | funzip >~/.local/bin/deno
chmod a+rx ~/.local/bin/yt-dlp ~/.local/bin/deno
yt-dlp --update-to nightly
```
---
## 2. Get Playlist IDs
Use the following command to extract playlist IDs or search maually in youtube.com
```shell
echo "salsa 2025" | parallel --ungroup --silent 'yt-dlp --playlist-end 5 --flat-playlist --simulate --match-filter "id~=^PL" --print id "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' | xargs
```
You will obtain a list like this:   
PLyawAyeZltbE-jkoIE5-Ex2A1uMX6GpJu PLFI2TZoGC6pGG4bJO_vKzUQTHapMrHi0a PLI3noN557CbkillhdPHL5K7YNyVRa5XWF PLzkaGyrLBf-uOwFlQxuqRe0_B98nKqJ-7 PLVwXkzK42QWq6LY36q6JEFBCrc4aWD_ka

## 3. Download — Manual Playlist List

```shell
echo PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL | \
parallel --ungroup --silent --delimiter ' ' -- 'yt-dlp --ignore-errors --no-warnings --flat-playlist --print "%(id)s§%(playlist)s§%(title)s" "https://www.youtube.com/playlist?list={}"' | \
grep -vE "Deleted video|Private video" | sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' | \
parallel --ungroup --silent --colsep § -- 'echo nice ionice -c 3 yt-dlp --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata --restrict-filename {1} -o "$HOME/Downloads/CarPlaylist/{2}/{3}.mp3"' | \
parallel --ungroup --max-procs 16
```
---
## 4. Download — Fully Automatic
10 playlists → ~519 tracks → 434 files → 30h playback → 1.8 GB → ~9 min.
```shell
echo "salsa 2025" | \
parallel --ungroup --silent -- 'yt-dlp --playlist-end 20 --flat-playlist --simulate --match-filter "id~=^PL" --print id "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' | \
parallel --ungroup --silent -- 'yt-dlp --ignore-errors --no-warnings --flat-playlist --print "%(id)s§%(playlist)s§%(title)s" "https://www.youtube.com/playlist?list={}"' | \
grep -vE "Deleted video|Private video" | sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' | \
parallel --ungroup --silent --colsep § -- 'echo nice ionice -c 3 yt-dlp --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata  {1} -o "$HOME/Downloads/CarPlaylist/{2}/{3}.mp3"' | \
parallel --ungroup --max-procs 16
```
---
## 5. Download — With Loudness Normalization (slow)
Applies `loudnorm` via ffmpeg post-processor:
```shell
echo "salsa 2025" | \
parallel --ungroup --silent -- 'yt-dlp --playlist-end 20 --flat-playlist --simulate --match-filter "id~=^PL" --print id "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' | \
parallel --ungroup --silent -- 'yt-dlp --ignore-errors --no-warnings --flat-playlist --print "%(id)s§%(playlist)s§%(title)s" "https://www.youtube.com/playlist?list={}"' | \
grep -vE "Deleted video|Private video" | sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' | \
parallel --ungroup --silent --colsep §  -- 'echo nice ionice -c 3 yt-dlp --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata --postprocessor-args \047ExtractAudio+ffmpeg:-filter:a loudnorm=I=-14:TP=-1.5:LRA=11\047 {1} -o "$HOME/Downloads/CarPlaylist/{2}/{3}.mp3"' | \
parallel --ungroup --max-procs 16
```
---
## 6. Automatic Cleanup
Cleans filenames, removes junk files, resizes cover art, strips unwanted metadata tags.
```shell
detox -vr ~/Downloads/CarPlaylist
find ~/Downloads/CarPlaylist -type f ! -name "*.mp3" -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 1 -depth -type d -exec sh -c   'if [ $(find "$0" -type f | wc -l) -lt 10 ]; then rm -r "$0"; fi' {} \;
find ~/Downloads/CarPlaylist -type f -exec sh -c "echo \"{}\" | grep -qP '[\x{0100}-\x{FFFF}]'" \; -exec rm {} \;
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{1,10}$" -delete
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{100}[^/]+$" -delete
find ~/Downloads/CarPlaylist -type f -name "*.mp3" \( -size -3M -o -size +8M \) -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 2 -type d -delete
fdupes ~/Downloads/CarPlaylist --noprompt --delete --recurse
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel 'ytid=$(exiftool -Comment -s3 {} | sed -n "s/.*v=\([a-zA-Z0-9_-]\{11\}\).*/\1/p"); if [ -n "$ytid" ]; then b=$(basename {} .mp3); s=$(echo "${b}" | sed "s/[^a-zA-Z0-9._-]/_/g" | cut -c1-20); mv -n {} "$(dirname {})/${s}_${ytid}.mp3"; fi'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel 'if exiftool -Picture -b {} 2>/dev/null | file - | grep -q image; then exiftool -Picture -b {} | convert - -resize 500x500 -quality 80 /tmp/cover_{#}.jpg && eyeD3 --remove-all-images {} && eyeD3 --add-image /tmp/cover_{#}.jpg:FRONT_COVER {} && rm /tmp/cover_{#}.jpg; fi'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel 'if ! exiftool -Picture -b {} 2>/dev/null | file - | grep -q image; then rm {}; fi'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel 'eyeD3 {} --remove-all-comments --user-text-frame="description:" --user-text-frame="purl:" --user-text-frame="synopsis:"'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel id3v2 -s {}
```
---
## 7. Volume Normalization
Use one only (do not mix):
```shell
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice parallel --eta --max-procs 20 loudgain -q -s e {}
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice parallel --eta --max-procs 20 mp3gain -r {}
```
| Tool | Method | Notes |
|---|---|---|
| `loudgain -q -s e` | Writes ReplayGain tags only | Recommended, audio untouched |
| `mp3gain -r` | Modifies audio data (lossless) | Better for older hardware |
---
## 8. Info & Stats
```shell
tree -d ~/Downloads/CarPlaylist # show tree
find ~/Downloads/CarPlaylist -type f -name "*.mp3"| wc -l | tr '\n' ' ' && echo mp3 files # counts the files
find ~/Downloads/CarPlaylist -type f -name "*.mp3" -exec mp3info -p "%S\n" {} + | awk '{ total += $1 } END { printf "Total runtime: %d hours %d minutes\n", total / 3600, (total % 3600) / 60 }' # runtime
du -sh ~/Downloads/CarPlaylist # filesize together
```
---
## 9. Manual Cleanup
- Remove folders with unwanted music
- Shorten directory names
- Copy to MP3 stick
- Delete from disk

---
