# Efficient Parallel Download for Car Music MP3 Playlist

Previously, it wasn't feasible to conduct parallel downloads with yt-dlp while retaining the playlist name as a directory name and preserving other metadata. However, I've discovered a method to achieve this by utilizing the trusty AWK. With this approach, download times are accelerated by approximately 20 times. On my laptop, I managed to download 1000 songs within 15 minutes using this method.

## Tool Preparation

Install all the necessary programs:
```
sudo apt install parallel detox normalize-audio mp3gain mp3info mp3check
wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O ~/.local/bin/yt-dlp
chmod a+rx ~/.local/bin/yt-dlp  
yt-dlp --update-to nightly
```
Use a cookie in your home directory. There is a Firefox extention to save a cookie in a file: https://github.com/hrdl-github/cookies-txt. Save it as ~/cookies.txt

## Get the playlist ID

A convenient way to gather Playlist IDs involves employing direct curl commands. In this instance, we employ the search term "salsa+2023," in 10 pages which yields 74 playlists.

```
time for page in {1..10}; do echo "https://www.youtube.com/results?search_query=salsa+2023+playlist&page=$page"  ; done  | parallel -P20 --silent curl -s {} | tr '"' '\n' | grep "playlist?list=PL" | grep -oP '(?<=list=)[\w-]+' | awk -F= '{if(length($1) == 34) print $1}' | wc -l
```
However, you can manually choose playlists and gather tags to compile a list similar to this one. For a car playlist, around 1000 songs should be adequate. Therefore, avoid gathering an excessive number of playlists, as it will result in an overwhelming number of songs.

In this instance, I've chosen approximately 4753 MP3 songs. You can verify this by using the following command:
```
time for page in {1..10}; do echo "https://www.youtube.com/results?search_query=salsa+2023+playlist&page=$page"  ; done  | parallel -P20 --silent curl -s {} | tr '"' '\n' | grep "playlist?list=PL" | grep -oP '(?<=list=)[\w-]+' | awk -F= '{if(length($1) == 34) print $1}' | awk '{for(i=1;i<=NF;i++) print "yt-dlp --no-warnings --print \"https://www.youtube.com/watch?v=%(id)s\" --flat-playlist " $i}' | nice ionice -c 3 parallel --silent  -P20 | wc -l
```

## Now it's time to massive download:

```
time for page in {1..10}; do echo "https://www.youtube.com/results?search_query=salsa+2023+playlist&page=$page"  ; done  | parallel -P20 --silent curl -s {} | tr '"' '\n' | grep "playlist?list=PL" | grep -oP '(?<=list=)[\w-]+' | awk -F= '{if(length($1) == 34) print $1}' | awk  '{print "yt-dlp --ignore-errors -no-abort-on-error --no-warnings --no-check-certificate --print \"https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s.mp3\" --flat-playlist " $1}' | parallel -P20 --silent | awk -F';' '{print "yt-dlp --ignore-errors -no-abort-on-error --no-warnings --no-check-certificate --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata " $1 " -o \"~/Downloads/SalasPlaylist/" $2 "/" $3"\""}' | nice ionice -c 3 parallel --ungroup --eta -P20
```
## Automaic clean up:
Clean up filenames with detox - the car stereo likes clean filenames:
```
ionice -c 3 detox -vr ~/Downloads/SalasPlaylist
```
Delete too big and too small files. Check first with "echo" in front of "rm" and maybe "| wc -l" at the end:
```
find ~/Downloads/SalasPlaylist -type f -name "*.mp3" \( -size -3M -o -size +8M \) -exec rm {} \; 
```
Delete files and subdirectories that are more than 2 levels deep from ~/Downloads/SalasPlaylist. Check first with "echo" in front of "rm" and maybe "| wc -l" at the end:
```
find ~/Downloads/SalasPlaylist -mindepth 2 -type d  -exec rm -rf {} \;
```
Normalize, with audio file volume normalizer. Depends on your audio player to work. 
```
time find ~/Downloads/SalasPlaylist -type f -name "*.mp3" | parallel --eta -P20 nice ionice -c 3 normalize-audio {}
time find ~/Downloads/SalasPlaylist -type f -name "*.mp3" | parallel --eta -P20 nice ionice -c 3 mp3gain {}
```
## Usefull utilitis:
```
tree -d  ~/Downloads/SalasPlaylist # schow tree
find ~/Downloads/SalasPlaylist -type f -name "*.mp3"| wc -l # count files
find ~/Downloads/SalasPlaylist -type f -name "*.mp3" -exec mp3info -p "%S\n" {} + | awk '{ total += $1 } END { printf "Total runtime: %d hours %d minutes\n", total / 3600, (total % 3600) / 60 }' # runtime
du -sh ~/Downloads/SalasPlaylist # filesize together
```
## Manual clean up: 
- Remove all folders containing music you do not like.
- Shorten directory names.
- Copy to an MP3 stick.
- Erase from the hard disk.

## Open Issues or Improvements:
- There is still around 5% of content that my car stereo cannot play; additional investigation is necessary. Despite using mp3check, it consistently reports that all of my files are faulty. So far, I've been resorting to manually skipping to the next track.
- Improving audio quality: I've chosen "--audio-quality 5", but I haven't noticed any improvement with higher numbers.
- Many MP3 files are not actually in the MP3 format internally; instead, they are in MPEG-2 format. How can I force them into the MP3 format?
- MP3 normalization is achieved through the use of normalize-audio and mp3gain. These tools do not modify the audio content; rather, they store a correction value that is interpreted by the audio player. However, not all players utilize this data. Is it preferable to implement a hard-coded normalizer instead?

