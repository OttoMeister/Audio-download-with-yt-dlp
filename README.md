# Efficient Parallel Download for Car Music MP3 Playlist

Previously, conducting parallel downloads with yt-dlp while maintaining the playlist name as a directory name and preserving metadata wasn't feasible. However, a method has been discovered to achieve this using AWK effectively. This approach accelerates download times by approximately 20 times. 1000 songs can be downloaded within 15 minutes using this method.

## Tool Preparation
Install all the necessary programs:
```shell
sudo apt install parallel detox normalize-audio mp3gain mp3info mp3check convmv
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
cat '/home/boss/Downloads/salsa 2023 playlist - YouTube.html' | tr ' ' '\n'| tr '"' ' ' | sed -n 's/.*list=\(.*\)>.*/\1/p' | sort | awk '!seen[$0]++' | awk -F= '{if(length($1) == 35) print $1}' | tr '\n ' '
```
You will obtain a list like this:

PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL

To estimate the number of MP3 files your playlists will generate, execute:
```shell
echo PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL | tr ' ' '\n' | awk '{for(i=1;i<=NF;i++) print "yt-dlp --no-warnings --print \"https://www.youtube.com/watch?v=%(id)s\" --flat-playlist " $i}' | nice ionice -c 3 parallel --silent -P20 | wc -l
```
However, you can alternatively curate playlists manually and gather tags to compile a list. For a car playlist, perhaps 1000 songs are adequate. Therefore, avoid accumulating an excessive number of playlists, as it will result in an overwhelming number of songs.

## Download your manual precesed list:
```shell
time echo PLD0kvNhPZ444CoLU7Z2ri3nbMn6uVDscR PLGx8vKOKHzlGkJlSeHL4HC7fWjLki_mH5 PLJzWprC5a8Ad49KnLX6_FgX0VAsp8J-h1 PL4U35lg0iKyZGrx9YITNqfgBwlah7Rm8A PLXl9q53Jut6k_WLWfIK3zv-3kwnBnA5fm PLFxMfmFGz8rFggUvGY8G_m1JIPQLKxPcq PLWEEt0QgQFIn8neNfE8EzRi1hsNn8CovL | tr ' ' '\n' | awk '{print "yt-dlp --ignore-errors -no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)#s;%(title)#s.mp3\" --flat-playlist " $1}' | parallel --max-procs 20 --silent | awk -F';' '{print "yt-dlp --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"~/Downloads/CarPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --bar --eta --max-procs 20
```
## 100% automatic download:

The initial approach is the most appealing. I've created 7 playlists containing 503 MP3 songs, which will then be condensed to 475 files, totaling 35 hours of music playback. These files will consume 2.3 gigabytes of storage space. The anticipated download duration is approximately 9 minutes.
```shell
time curl https://www.youtube.com/results?search_query=salsa+2023+playlist | tr '"' '\n' | grep "playlist?list=PL" | grep -oP '(?<=list=)[\w-]+' | awk -F= '{if(length($1) == 34) print $1}' | awk '{print "yt-dlp --ignore-errors -no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)#s;%(title)#s.mp3\" --flat-playlist " $1}' | parallel --max-procs 20 --silent | awk -F';' '{print "yt-dlp --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"~/Downloads/CarPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --bar --eta --max-procs 20
```
Implementing this approach leads to extensive downloads, encompassing 118 playlists containing 19,520 MP3 files. This procedure will require a considerable amount of time and occupy a substantial portion of disk space. The download process lasted 10 hours, resulting in 14,813 MP3 files totaling 70 gigabytes after cleaning. 1029 hours of music!
```shell
time yt-dlp --flat-playlist --simulate --print id -v "https://www.youtube.com/results?search_query=salsa+2023+playlist" | awk -F= '{if(length($1) == 34) print $1}' | awk '!seen[$0]++' | awk '{print "yt-dlp --ignore-errors -no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)#s;%(title)#s.mp3\" --flat-playlist " $1}' | parallel --max-procs 20 --silent | awk -F';' '{print "yt-dlp --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"~/Downloads/CarPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --bar --eta --max-procs 20
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
- Remove empty directories.
```shell
ionice -c 3 detox -vr ~/Downloads/CarPlaylist
find ~/Downloads/CarPlaylist -type f -exec sh -c "echo \"{}\" | grep -qP '[^[:ascii:]]'" \; -exec rm {} \;
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{1,10}$" -delete
find ~/Downloads/CarPlaylist -type f -regextype posix-egrep -regex ".*/[^/]{100}[^/]+$" -delete
find ~/Downloads/CarPlaylist -type f -name "*.mp3" \( -size -3M -o -size +8M \) -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 2 -type d -exec rm -rf {} \;
find ~/Downloads/CarPlaylist -type f ! -name "*.mp3" -exec rm {} \;
find ~/Downloads/CarPlaylist -mindepth 1 -type d -exec sh -c 'if [ $(find "$0" -type f | wc -l) -lt 10 ]; then rm -r "$0"; fi' {} \;
find ~/Downloads/CarPlaylist -type d -empty -delete
```
## Normalize audio file volumes
(relying on the audio player for functionality)
```shell
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel --eta --max-procs 20 normalize-audio {}
find ~/Downloads/CarPlaylist -type f -name "*.mp3" | nice ionice -c 3 parallel --eta --max-procs 20 mp3gain {}
```
## Usefull information tools:
```shell
tree -d ~/Downloads/CarPlaylist # schow tree
find ~/Downloads/CarPlaylist -type f -name "*.mp3"| wc -l | tr '\n' ' ' && echo mp3 files
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
- MP3 normalization is achieved through the use of normalize-audio and mp3gain. These tools do not modify the audio content; rather, they store a correction value that is interpreted by the audio player. However, not all players utilize this data. Is it preferable to implement a hard-coded normalizer instead?
