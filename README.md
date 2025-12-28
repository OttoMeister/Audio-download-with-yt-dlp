# Efficient Parallel Download for Car Music MP3 Playlist

Previously, it was not possible to perform parallel downloads with yt-dlp while preserving the playlist name as the directory and retaining metadata. However, an effective solution using AWK has now been discovered. This method enables true parallel downloads while maintaining the desired structure and metadata, resulting in a significant performance boost. With this approach, download speeds improve by roughly 20×, allowing approximately 1,000 songs to be downloaded in just 15 minutes.

## Tool Preparation
Install all the necessary programs:
```shell
sudo apt install parallel detox normalize-audio mp3gain mp3info mp3check convmv detox
wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O ~/.local/bin/yt-dlp
chmod a+rx ~/.local/bin/yt-dlp 
yt-dlp --update-to nightly
```
## Acquire the Playlist ID Semi-Automatically or Manually
Open your web browser and navigate to YouTube.
Utilize the search function to find playlists and music genres.
Once you find a suitable playlist, save the HTML file.
Use the following command to extract playlist IDs:
```shell
echo "salsa 2025" | sed 's/ /+/g' | xargs -I QUERY nice yt-dlp --playlist-end 10 --flat-playlist --simulate --print id "https://www.youtube.com/results?search_query=QUERY&sp=EgIQAw==" | awk 'length($1)==34 && !seen[$0]++' | xargs
```
You will obtain a list like this:

PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL

## Download your manual precesed list:
```shell
echo PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL | tr ' ' '\n' | awk 'length($1)==34 && !seen[$0]++' | awk '{print "yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s.mp3\" --flat-playlist " $1}' | parallel --max-procs 20 --silent | awk -F';' '{gsub(/[^a-zA-Z0-9 ._-]/,"",$2); gsub(/[^a-zA-Z0-9 ._-]/,"",$3); print "yt-dlp --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"$HOME/Downloads/CarPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --max-procs 20 --bar --eta
```
## 100% automatic download:
The initial approach is the most appealing. I've created 10 playlists containing 1050 MP3 songs, which will then be condensed to ?? files, totaling ?? hours of music playback. These files will consume 2.3 gigabytes of storage space. The anticipated download duration is approximately 9 minutes.
```shell
echo "salsa 2025" | sed 's/ /+/g' | xargs -I QUERY nice yt-dlp --playlist-end 10 --flat-playlist --simulate --print id "https://www.youtube.com/results?search_query=QUERY&sp=EgIQAw==" | awk 'length($1)==34 && !seen[$0]++' | awk '{print "yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s.mp3\" --flat-playlist " $1}' | parallel --max-procs 20 --silent | awk -F';' '{gsub(/[^a-zA-Z0-9 ._-]/,"",$2); gsub(/[^a-zA-Z0-9 ._-]/,"",$3); print "yt-dlp --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"$HOME/Downloads/CarPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --max-procs 20 --bar --eta
```
## Automatic clean up:
- Use detox to further clean filenames for optimal compatibility.
- Remove filenames containing characters beyond the ASCII range.
- Eliminate files that are either excessively large or too small.
- Delete files with filenames that are too short.
- Remove files with filenames that exceed a certain length.
- Delete files and subdirectories beyond a depth of 2 levels.
- Exclude non-MP3 files from the directory.
- Eradicate directories containing less than 10 files.
- Shorts the long filenames
```shell
ionice -c 3 detox -vr ~/Downloads/CarPlaylist
find ~/Downloads/CarPlaylist -type f -exec sh -c "echo \"{}\" | grep -qP '[^[:ascii:]]'" \; -exec rm {} \;
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{1,10}$" -delete
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{100}[^/]+$" -delete
find ~/Downloads/CarPlaylist -type f -name "*.mp3" \( -size -3M -o -size +8M \) -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 2 -type d -exec rm -rf {} \;
find ~/Downloads/CarPlaylist -type f ! -name "*.mp3" -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 1 -type d -exec sh -c 'if [ $(find "$0" -type f | wc -l) -lt 10 ]; then rm -r "$0"; fi' {} \;
find ~/Downloads/CarPlaylist -type f -name '*.mp3' -exec bash -c 'f="$1"; d=$(dirname "$f"); b=$(basename "$f" .mp3); s=$(echo "$b" | sed "s/[^a-zA-Z0-9._-]/_/g" | cut -c1-20); h=$(echo -n "$f" | md5sum | cut -c1-6); mv -n "$f" "$d/${s}_${h}.mp3"' _ {} \;
```
## Normalize audio file volumes without new encoding 
```shell
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel --eta --max-procs 20 mp3gain -r {}
```
## Usefull information tools:
```shell
tree -d ~/Downloads/CarPlaylist # schow tree
find ~/Downloads/CarPlaylist -type f -name "*.mp3"| wc -l | tr '\n' ' ' && echo mp3 files # counts the files
find ~/Downloads/CarPlaylist -type f -name "*.mp3" -exec mp3info -p "%S\n" {} + | awk '{ total += $1 } END { printf "Total runtime: %d hours %d minutes\n", total / 3600, (total % 3600) / 60 }' # runtime
du -sh ~/Downloads/CarPlaylist # filesize together
```
## Manual clean up: 
- Remove all folders containing music you do not like.
- Shorten directory names.
- Copy to an MP3 stick.
- Erase from the hard disk.
 
## Open Issues or Improvements:
- There is still around 5% of content that my car stereo cannot play, additional investigation is necessary. Despite using mp3check, it consistently reports that all of my files are faulty. So far, I've been resorting to manually skipping to the next track.
- Improving audio quality: I've chosen "--audio-quality 5", but I haven't noticed any improvement with lower numbers. Files getting biger, but not better.
- Many MP3 files are not actually in the MP3 format internally; instead, they are in MPEG-2 format. How can I force them into the MP3 format?

## License
This project is released under the WTFPL LICENSE.
<a href="http://www.wtfpl.net/"><img src="http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png" width="80" height="15" alt="WTFPL" /></a>
