# Efficient Parallel Download for Car Music MP3 Playlist

Previously, it was not possible to perform parallel downloads with yt-dlp while preserving the playlist name as the directory and retaining metadata. However, an effective solution using AWK has now been discovered. This method enables true parallel downloads while maintaining the desired structure and metadata, resulting in a significant performance boost. With this approach, download speeds improve by roughly 20×, allowing approximately 1,000 songs to be downloaded in just 15 minutes.

## Tool Preparation
Install all the necessary programs: 
```shell
sudo apt install parallel normalize-audio mp3gain mp3info loudgain mp3check detox eyed3 exiftool imagemagick id3v2 unzip
# mkdir -p ~/.local/bin && echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O ~/.local/bin/yt-dlp
wget https://github.com/denoland/deno/releases/latest/download/deno-x86_64-unknown-linux-gnu.zip -qO- | funzip >~/.local/bin/deno
chmod a+rx ~/.local/bin/yt-dlp ~/.local/bin/deno
yt-dlp --update-to nightly
```
## Acquire the Playlist ID Semi-Automatically or Manually
Use the following command to extract playlist IDs:
```shell
echo "salsa 2025" | sed 's/ /+/g' | xargs -I QUERY nice yt-dlp --playlist-end 10 --flat-playlist --simulate --print id "https://www.youtube.com/results?search_query=QUERY&sp=EgIQAw==" | awk 'length($1)==34 && !seen[$0]++' | xargs
```
You will obtain a list like this:
PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL

## Download your manual precesed list:
```shell
echo PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL | tr ' ' '\n' | awk 'length($1)==34 && !seen[$0]++' | awk '{print "yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s.mp3\" --flat-playlist " $1}' | parallel --max-procs 20 --silent | awk -F';' '{gsub(/[^a-zA-Z0-9 ._-]/,"",$2); gsub(/[^a-zA-Z0-9 ._-]/,"",$3); print "yt-dlp --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"$HOME/Downloads/CarPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --max-procs 16 --bar --eta
```
## 100% automatic download:
The initial approach is the most appealing. I've created 10 playlists containing 519 MP3 songs, which will then be condensed to 434 files, totaling 30 hours of music playback. These files will consume 1.8 gigabytes of storage space. The anticipated download duration is approximately 9 minutes.
```shell
echo "salsa 2025" | sed 's/ /+/g' | xargs -I QUERY nice yt-dlp --playlist-end 10 --flat-playlist --simulate --print id "https://www.youtube.com/results?search_query=QUERY&sp=EgIQAw==" | awk 'length($1)==34 && !seen[$0]++' | awk '{print "yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s.mp3\" --flat-playlist " $1}' | parallel --max-procs 20 --silent | awk -F';' '{gsub(/[^a-zA-Z0-9 ._-]/,"",$2); gsub(/[^a-zA-Z0-9 ._-]/,"",$3); print "yt-dlp --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"$HOME/Downloads/CarPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --max-procs 16 --bar --eta
```
## Automatic clean up:
- Clean filenames by removing or replacing problematic characters 
- Delete all non-MP3 files 
- Delete directories containing fewer than 10 files 
- Delete files with Unicode characters in path 
- Delete files with filenames shorter than 10 characters 
- Delete files with filenames longer than 100 characters 
- Delete MP3s smaller than 3MB or larger than 8MB 
- Delete all subdirectories deeper then 2 
- Rename files based on YouTube ID extracted from metadata 
- Resize and compress cover art to 500x500 at 80% quality 
- Delete MP3s without cover art 
- Clear description, comment and synopsis metadata fields 
- Converts all tags to ID3v2.3 
```shell
detox -vr ~/Downloads/CarPlaylist
find ~/Downloads/CarPlaylist -type f ! -name "*.mp3" -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 1 -type d -exec sh -c 'if [ $(find "$0" -type f | wc -l) -lt 10 ]; then rm -r "$0"; fi' {} \;
find ~/Downloads/CarPlaylist -type f -exec sh -c "echo \"{}\" | grep -qP '[\x{0100}-\x{FFFF}]'" \; -exec rm {} \;
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{1,10}$" -delete
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{100}[^/]+$" -delete
find ~/Downloads/CarPlaylist -type f -name "*.mp3" \( -size -3M -o -size +8M \) -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 2 -type d -delete
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel 'ytid=$(exiftool -Comment -s3 {} | sed -n "s/.*v=\([a-zA-Z0-9_-]\{11\}\).*/\1/p"); if [ -n "$ytid" ]; then b=$(basename {} .mp3); s=$(echo "${b}" | sed "s/[^a-zA-Z0-9._-]/_/g" | cut -c1-20); mv -n {} "$(dirname {})/${s}_${ytid}.mp3"; fi'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel 'if exiftool -Picture -b {} 2>/dev/null | file - | grep -q image; then exiftool -Picture -b {} | convert - -resize 500x500 -quality 80 /tmp/cover_{#}.jpg && eyeD3 --remove-all-images {} && eyeD3 --add-image /tmp/cover_{#}.jpg:FRONT_COVER {} && rm /tmp/cover_{#}.jpg; fi'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel 'if ! exiftool -Picture -b {} 2>/dev/null | file - | grep -q image; then rm {}; fi'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel 'eyeD3 {} --remove-all-comments --user-text-frame="description:" --user-text-frame="synopsis:"'
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | parallel id3v2 -s {}
```
## Normalize audio file volumes without new encoding (use only one!)
- "loudgain -q -s e" (recommended) - Writes Metadata Tags (Leaves audio data untouched). Best for new player. 
- "mp3gain -r" (deprecated) - Modifies Audio Data (Lossless but changes file structure). Best for older hardware. 
```shell
# find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel --eta --max-procs 20 loudgain -q -s e {}
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel --eta --max-procs 20 mp3gain -r {}
```
## Normalize audio file volumes with new encoding to 128k CBR. Best for very old hardware. 
- Basic Normalization: -filter:a dynaudnorm 
- Gentle Normalization (Music): -filter:a dynaudnorm=framelen=1000:gausssize=31:peak=0.95 
- Aggressive Normalization (Podcasts/Audiobooks): -filter:a dynaudnorm=framelen=500:gausssize=15:maxgain=20:targetrms=0.25 
- Compression (Uniform Volume): -filter:a dynaudnorm=compress=10:peak=0.9:targetrms=0.2 
- Gentle Normalization (Preserve Original Dynamics): -filter:a dynaudnorm=framelen=2000:gausssize=51:maxgain=5:peak=0.95 
```shell
find /home/boss/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel --max-procs 16 "ffmpeg -y -i {} -filter:a dynaudnorm -c:a libmp3lame -b:a 128k -c:v copy -map_metadata 0 -id3v2_version 3 {.}_tmp.mp3 && mv {.}_tmp.mp3 {}"
```
## Usefull information tools:
```shell
tree -d ~/Downloads/CarPlaylist # schow tree
find ~/Downloads/CarPlaylist -type f -name "*.mp3"| wc -l | tr '\n' ' ' && echo mp3 files # counts the files
find ~/Downloads/CarPlaylist -type f -name "*.mp3" -exec mp3info -p "%S\n" {} + | awk '{ total += $1 } END { printf "Total runtime: %d hours %d minutes\n", total / 3600, (total % 3600) / 60 }' # runtime
du -sh ~/Downloads/CarPlaylist # filesize together
```
## Manual clean up: 
- Remove all folders containing music you do not like 
- Shorten directory names 
- Copy to an MP3 stick 
- Erase from the hard disk 

## License
This project is released under the WTFPL LICENSE.
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
