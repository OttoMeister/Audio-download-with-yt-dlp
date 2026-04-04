# More tools for yt-dlp

## Download compleat playlists of videos in max resolution and anhanced audio
```shell
echo PLffX9yMQbaQhZbTRat79oh1kHhGHwv1Sn | parallel -d ' ' 'yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(playlist_index)03d;%(title)s" --flat-playlist {}' | nice ionice -c 3 parallel --colsep ';' --ungroup --max-procs 4 'yt-dlp --force-ipv4 -f \"bestvideo\[height\<=1080\]\[vcodec=vp9\]+bestaudio\[ext=webm\]/bestvideo\[height\<=1080\]+bestaudio/best\[height\<=1080\]\" --extractor-args \"youtube:player_client=web,default\" --postprocessor-args \"ffmpeg:-c:v copy -c:a libopus -b:a 128k -filter:a loudnorm=I=-14:TP=-1.5:LRA=11\" --continue --retries 10 --fragment-retries 15 --retry-sleep fragment:10 --concurrent-fragments 4 --socket-timeout 20 --buffer-size 64K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:$HOME/Downloads/VideoPlaylist --paths temp:$HOME/Downloads/ {1} -o \"{=2 s/[^a-zA-Z0-9 ._-]//g =}_{=3 s/[^a-zA-Z0-9 ._-]//g =}_{=4 s/[^a-zA-Z0-9 ._-]//g =}\"'
```
---
## License
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
