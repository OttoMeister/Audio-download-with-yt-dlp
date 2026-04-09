# More tools for yt-dlp

# Download compleat playlists of videos in max resolution and anhanced audio
```shell
echo PLlsxoL4OXm7Vd6UXJ3ahyhshbaltz_phP PLlsxoL4OXm7XLqAQY_Q-_hCB6EZ8AUpjg \
| parallel --ungroup --silent --delimiter ' ' --trim lr -- 'yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print "%(id)s§%(playlist)s§%(playlist_index)03d§%(title)s" --flat-playlist {}' \
| grep -vE "Deleted video|Private video" \
| sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' \
| parallel --ungroup --silent --colsep § -- 'echo yt-dlp --force-ipv4 -f \"bestvideo\[height\<=1080\]\[vcodec=vp9\]+bestaudio\[ext=webm\]/bestvideo\[height\<=1080\]+bestaudio/best\[height\<=1080\]\" --postprocessor-args \"ffmpeg:-c:v copy -c:a libopus -b:a 128k -filter:a loudnorm=I=-14:TP=-1.5:LRA=11\" --continue --retries 10 --fragment-retries 15 --retry-sleep fragment:10 --concurrent-fragments 4 --socket-timeout 20 --buffer-size 64K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:$HOME/Downloads/VideoPlaylist  {1} -o {2}/{3}-{4}' \
| parallel --ungroup --max-procs 16
```
---
# Download directly with yt-dlp in a format suitable for WhatsApp
```shell
# High Quality Variante
yt-dlp _Es1NbQXDtE -o "$(mktemp -u /tmp/yt.XXXXXX)" --exec "ffmpeg -nostdin -y -i \"{}\" -vf \"scale='min(640,iw)':-2,format=yuv420p\" -c:v libx265 -crf 28 -preset fast -tag:v hvc1 -x265-params info=0 -c:a aac -b:a 32k -ac 1 -ar 16000 -af loudnorm=I=-12 -map_metadata -1 -map_metadata:s:v -1 -map_metadata:s:a -1 -metadata title=\"\" -bitexact -fflags +bitexact -flags:v +bitexact -flags:a +bitexact -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_HDWA.mp4 && rm -f \"{}\"" 
# Medium Quality Variante 
yt-dlp _Es1NbQXDtE -o "$(mktemp -u /tmp/yt.XXXXXX)" --exec "ffmpeg -nostdin -y -i \"{}\" -vf \"scale='min(400,iw)':-2,format=yuv420p\" -c:v libx265 -crf 28 -preset slow -tag:v hvc1 -x265-params info=0 -c:a aac -b:a 32k -ac 1 -ar 16000 -af loudnorm=I=-12 -map_metadata -1 -map_metadata:s:v -1 -map_metadata:s:a -1 -metadata title=\"\" -bitexact -fflags +bitexact -flags:v +bitexact -flags:a +bitexact -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_MDWA.mp4 && rm -f \"{}\"" 
# Low Quality Variante
yt-dlp _Es1NbQXDtE -o "$(mktemp -u /tmp/yt.XXXXXX)" --exec "ffmpeg -nostdin -y -i \"{}\" -vf \"scale='min(320,iw)':-2,format=yuv420p\" -c:v libx265 -crf 28 -preset slow -tag:v hvc1 -x265-params info=0 -c:a aac -b:a 24k -ac 1 -ar 16000 -af loudnorm=I=-12 -map_metadata -1 -map_metadata:s:v -1 -map_metadata:s:a -1 -metadata title=\"\" -bitexact -fflags +bitexact -flags:v +bitexact -flags:a +bitexact -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_SDWA.mp4 && rm -f \"{}\"" 
# Nano Quality Variante
yt-dlp _Es1NbQXDtE -o "$(mktemp -u /tmp/yt.XXXXXX)" --exec "ffmpeg -nostdin -y -i \"{}\" -vf \"scale='min(320,iw)':-2,format=yuv420p\" -c:v libx265 -crf 33 -preset veryslow -r 15 -tag:v hvc1 -x265-params info=0 -c:a aac -b:a 24k -ac 1 -ar 16000 -af loudnorm=I=-12 -map_metadata -1 -map_metadata:s:v -1 -map_metadata:s:a -1 -metadata title=\"\" -bitexact -fflags +bitexact -flags:v +bitexact -flags:a +bitexact -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_NDWA.mp4 && rm -f \"{}\"" 
# Ultra Quality Variante
yt-dlp _Es1NbQXDtE -o "$(mktemp -u /tmp/yt.XXXXXX)" --exec "ffmpeg -nostdin -y -i \"{}\" -vf \"scale='min(240,iw)':-2,format=yuv420p\" -c:v libx265 -crf 38 -preset veryslow -r 12 -tag:v hvc1 -x265-params info=0 -c:a aac -b:a 12k -ac 1 -ar 12000 -af loudnorm=I=-12 -map_metadata -1 -map_metadata:s:v -1 -map_metadata:s:a -1 -metadata title=\"\" -bitexact -fflags +bitexact -flags:v +bitexact -flags:a +bitexact -movflags +faststart ~/Dokumente/\$(date +%Y%m%d_%H%M%S)_UDWA.mp4 && rm -f \"{}\"" 
```
---
# Download Video (slow and secure) anhanced audio
```shell
# HD
echo ZIskmUYZQfo nCNfE2Fcmms _XC1ys67P1M ZFyCEtSiNaA tQ-6uEN9WYM 3CQU1bC8mCM ZbHRP5nfio0 | parallel --delimiter ' ' --trim lr --keep-order --ungroup 'until nice yt-dlp -v --force-ipv4 -f "bestvideo[height<=1080][vcodec=vp9]+bestaudio[ext=webm]/bestvideo[height<=1080]+bestaudio/best[height<=1080]" --postprocessor-args "Merger+ffmpeg:-c:a libopus -filter:a loudnorm=I=-14:TP=-1.5:LRA=11" --continue --retries 10 --fragment-retries 10 --retry-sleep fragment:10 --concurrent-fragments 6 --socket-timeout 20 --buffer-size 16K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:~/Downloads -o "%(uploader)s -HD- %(title)s [%(id)s].%(ext)s" {}; do echo "[{}] Error, new try in 15s..." && sleep 15; done; echo "[{}] ok!"'
# SD
echo ZIskmUYZQfo nCNfE2Fcmms _XC1ys67P1M ZFyCEtSiNaA tQ-6uEN9WYM 3CQU1bC8mCM ZbHRP5nfio0 | parallel --delimiter ' ' --trim lr --keep-order --ungroup 'until nice yt-dlp -v --force-ipv4 -f "bestvideo[height<=480][vcodec=vp9]+bestaudio[ext=webm]/bestvideo[height<=480]+bestaudio/best[height<=480]"    --postprocessor-args "Merger+ffmpeg:-c:a libopus -filter:a loudnorm=I=-14:TP=-1.5:LRA=11" --continue --retries 10 --fragment-retries 10 --retry-sleep fragment:3  --concurrent-fragments 6 --socket-timeout 20 --buffer-size 16K --http-chunk-size 1M --no-playlist --embed-metadata --paths home:~/Downloads -o "%(uploader)s -SD- %(title)s [%(id)s].%(ext)s" {};do echo "[{}] Error, new try in 15s..." && sleep 15; done; echo "[{}] ok!"'
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
# Download podcasts from playlists
```shell 
echo PLlsxoL4OXm7Xie34V7Jokal23ml-i9vjM PLlsxoL4OXm7Xb58SgB_Tq4zYW8rOMr05E PLlsxoL4OXm7V7TvxEu62LDt2juCgQZEc0 \
| parallel --ungroup --silent --delimiter ' ' 'yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print "%(id)s§%(playlist)s§%(playlist_index)03d§%(title)s" --flat-playlist {}' \
| grep -vE "Deleted video|Private video" \
| sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' \
| parallel --ungroup --silent --colsep § 'echo nice ionice -c 3 yt-dlp --force-ipv4 --extract-audio --audio-format mp3 --audio-quality 5 --continue --retries 10 --fragment-retries 15 --retry-sleep fragment:10 --concurrent-fragments 4 --socket-timeout 20 --buffer-size 64K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:$HOME/Downloads/AudioPLPodcast {1} -o {2}/{3}-{4} --exec "\"ffmpeg -y -i \\\"%(filepath)s\\\" -ac 1 -ar 22050 -b:a 24k -filter:a dynaudnorm=framelen=500:gausssize=15:maxgain=20:targetrms=0.25 -map_metadata 0 -metadata description= -metadata synopsis= -metadata purl= -id3v2_version 3 \\\"%(filepath)s.tmp.mp3\\\" && mv \\\"%(filepath)s.tmp.mp3\\\" \\\"%(filepath)s\\\"\""' \
| parallel --ungroup --max-procs 16
```
---
# Download podcasts from youtube
```shell 
echo "_yy74tEVMp0 QCb5Aq98ZKc yqd4btA5L_A FVogvwhcYWw 9o0yXTd8kVE y6WZbBxA_tY XD3YB-lhmcY" \
| parallel --ungroup --silent --delimiter ' ' -- 'yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --no-playlist --print "%(id)s§%(title)s" --flat-playlist {}' \
| grep -vE "Deleted video|Private video" \
| sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' \
| parallel --ungroup --silent --colsep § -- 'echo nice ionice -c 3 yt-dlp --force-ipv4 --extract-audio --audio-format mp3 --audio-quality 5 --continue --retries 10 --fragment-retries 15 --retry-sleep fragment:10 --concurrent-fragments 4 --socket-timeout 20 --buffer-size 64K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:$HOME/Downloads/AudioPodcast {1} -o {2} --exec "\"ffmpeg -y -i \\\"%(filepath)s\\\" -ac 1 -ar 22050 -b:a 24k -filter:a dynaudnorm=framelen=500:gausssize=15:maxgain=20:targetrms=0.25 -map_metadata 0 -metadata description= -metadata synopsis= -metadata purl= -id3v2_version 3 \\\"%(filepath)s.tmp.mp3\\\" && mv \\\"%(filepath)s.tmp.mp3\\\" \\\"%(filepath)s\\\"\""' \
| parallel --ungroup --max-procs 16
```
---
## License
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
