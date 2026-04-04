# More tools for yt-dlp

# Download compleat playlists of videos in max resolution and anhanced audio
```shell
echo PLffX9yMQbaQhZbTRat79oh1kHhGHwv1Sn \
| parallel -d ' ' 'yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(playlist_index)03d;%(title)s" --flat-playlist {}' \
| parallel --colsep ';' --ungroup --max-procs 4 'nice ionice -c 3 yt-dlp --force-ipv4 -f \"bestvideo\[height\<=1080\]\[vcodec=vp9\]+bestaudio\[ext=webm\]/bestvideo\[height\<=1080\]+bestaudio/best\[height\<=1080\]\" --extractor-args \"youtube:player_client=web,default\" --postprocessor-args \"ffmpeg:-c:v copy -c:a libopus -b:a 128k -filter:a loudnorm=I=-14:TP=-1.5:LRA=11\" --continue --retries 10 --fragment-retries 15 --retry-sleep fragment:10 --concurrent-fragments 4 --socket-timeout 20 --buffer-size 64K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:$HOME/Downloads/VideoPlaylist --paths temp:$HOME/Downloads/ {1} -o \"{=2 s/[^a-zA-Z0-9 ._-]//g =}_{=3 s/[^a-zA-Z0-9 ._-]//g =}_{=4 s/[^a-zA-Z0-9 ._-]//g =}\"'
```
---
# Download directly with yt-dlp in a format suitable for WhatsApp
## High Quality Variante
```shell
yt-dlp "https://www.facebook.com/doblea.noticiasoxapampa/videos/850424721349638" -o "$(mktemp -u /tmp/fb.XXXXXX).mp4" --exec "ffmpeg -y -i {} -vf \"scale='min(640,iw)':-2,format=yuv420p\" -c:v libx265 -crf 28 -preset slow -tag:v hvc1 -c:a aac -b:a 64k -ac 1 -af loudnorm=I=-12 -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_HQ_WA.mp4 && rm {}"
```
## Smarter Variante
```shell
yt-dlp "https://www.facebook.com/doblea.noticiasoxapampa/videos/850424721349638" -o "$(mktemp -u /tmp/fb.XXXXXX).mp4" --exec "ffmpeg -y -i {} -vf \"scale='min(400,iw)':-2,format=yuv420p\" -c:v libx265 -crf 28 -preset slow -tag:v hvc1 -c:a aac -b:a 64k -ac 1 -af loudnorm=I=-12 -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_Smarter_WA.mp4 && rm {}"
```
## UltraTiny Variante
```shell 
yt-dlp "https://www.facebook.com/doblea.noticiasoxapampa/videos/850424721349638" -o "$(mktemp -u /tmp/fb.XXXXXX).mp4" --exec "ffmpeg -y -i {} -vf \"scale='min(320,iw)':-2,format=yuv420p\" -c:v libx265 -crf 28 -preset slow -tag:v hvc1 -c:a aac -b:a 64k -ac 1 -af loudnorm=I=-12 -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_UltraTiny_WA.mp4 && rm {}"
```
---


# Download Video HD (slow and secure) mit Kompresser und Normalisierung
```shell 
echo ZIskmUYZQfo nCNfE2Fcmms _XC1ys67P1M ZFyCEtSiNaA tQ-6uEN9WYM 3CQU1bC8mCM ZbHRP5nfio0 | tr ' ' '\n' | parallel --ungroup -j 2 'until nice yt-dlp -v --force-ipv4 -f "bestvideo[height<=1080][vcodec=vp9]+bestaudio[ext=webm]/bestvideo[height<=1080]+bestaudio/best[height<=1080]" --postprocessor-args "Merger+ffmpeg:-filter:a loudnorm=I=-14:TP=-1.5:LRA=11" --continue --retries 10 --fragment-retries 10 --retry-sleep fragment:10 --concurrent-fragments 6 --socket-timeout 20 --buffer-size 16K --http-chunk-size 5M --no-playlist --embed-metadata --paths temp:~/tmp --paths home:~/Downloads -o "%(uploader)s -HD- %(title)s [%(id)s].%(ext)s" {}; do echo "[{}] Fehler – neuer Versuch in 15s..." && sleep 15; done; echo "[{}] fertig!"'
```
# Download Video SD (slow and secure) mit Kompresser und Normalisierung
```shell 
echo ZIskmUYZQfo nCNfE2Fcmms _XC1ys67P1M ZFyCEtSiNaA tQ-6uEN9WYM 3CQU1bC8mCM ZbHRP5nfio0 | tr ' ' '\n' | parallel --ungroup -j 1 'until nice yt-dlp -v --force-ipv4 -f "bestvideo[height<=480][vcodec=vp9]+bestaudio[ext=webm]/bestvideo[height<=480]+bestaudio/best[height<=480]" --postprocessor-args "Merger+ffmpeg:-filter:a loudnorm=I=-14:TP=-1.5:LRA=11" --continue --retries 10 --fragment-retries 10 --retry-sleep fragment:3 --concurrent-fragments 6 --socket-timeout 20 --buffer-size 16K --http-chunk-size 1M --no-playlist --embed-metadata --paths temp:~/tmp --paths home:~/Downloads -o "%(uploader)s -SD- %(title)s [%(id)s].%(ext)s" {}; do echo "[{}] Fehler – neuer Versuch in 15s..." && sleep 15; done; echo "[{}] fertig!"'
```
---
# The Dynamic Audio Normalizer in ffmpeg has several useful parameters for fine-tuning
```shell 
# Test 0: Without Normalization
yt-dlp -v --extract-audio --audio-format mp3 --audio-quality 5 -o "test_original_%(title)s.%(ext)s" https://www.youtube.com/watch?v=cKVUCgb9N5U
# Test 1: Basic Normalization
yt-dlp -v --extract-audio --audio-format mp3 --audio-quality 5 --postprocessor-args "ffmpeg:-filter:a dynaudnorm" -o "test_basic_%(title)s.%(ext)s" https://www.youtube.com/watch?v=cKVUCgb9N5U
# Test 2: Gentle Normalization (Music)
yt-dlp -v --extract-audio --audio-format mp3 --audio-quality 5 --postprocessor-args "ffmpeg:-filter:a dynaudnorm=framelen=1000:gausssize=31:peak=0.95" -o "test_music_%(title)s.%(ext)s" https://www.youtube.com/watch?v=cKVUCgb9N5U
# Test 3: Aggressive Normalization (Podcasts/Audiobooks)
yt-dlp -v --extract-audio --audio-format mp3 --audio-quality 5 --postprocessor-args "ffmpeg:-filter:a dynaudnorm=framelen=500:gausssize=15:maxgain=20:targetrms=0.25" -o "test_podcast_%(title)s.%(ext)s" https://www.youtube.com/watch?v=cKVUCgb9N5U
# Test 4: Compression (Uniform Volume)
yt-dlp -v --extract-audio --audio-format mp3 --audio-quality 5 --postprocessor-args "ffmpeg:-filter:a dynaudnorm=compress=10:peak=0.9:targetrms=0.2" -o "test_compression_%(title)s.%(ext)s" https://www.youtube.com/watch?v=cKVUCgb9N5U
# Test 5: Gentle Normalization (Preserve Original Dynamics)
yt-dlp -v --extract-audio --audio-format mp3 --audio-quality 5 --postprocessor-args "ffmpeg:-filter:a dynaudnorm=framelen=2000:gausssize=51:maxgain=5:peak=0.95" -o "test_gentle_%(title)s.%(ext)s" https://www.youtube.com/watch?v=cKVUCgb9N5U
```
---



---
## License
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
