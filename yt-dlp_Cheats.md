# More tools for yt-dlp

## Download compleat playlists of videos in max resolution and anhanced audio
```shell
echo PLffX9yMQbaQhZbTRat79oh1kHhGHwv1Sn | parallel -d ' ' 'yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(playlist_index)03d;%(title)s" --flat-playlist {}' | nice ionice -c 3 parallel --colsep ';' --ungroup --max-procs 4 'yt-dlp --force-ipv4 -f \"bestvideo\[height\<=1080\]\[vcodec=vp9\]+bestaudio\[ext=webm\]/bestvideo\[height\<=1080\]+bestaudio/best\[height\<=1080\]\" --extractor-args \"youtube:player_client=web,default\" --postprocessor-args \"ffmpeg:-c:v copy -c:a libopus -b:a 128k -filter:a loudnorm=I=-14:TP=-1.5:LRA=11\" --continue --retries 10 --fragment-retries 15 --retry-sleep fragment:10 --concurrent-fragments 4 --socket-timeout 20 --buffer-size 64K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:$HOME/Downloads/VideoPlaylist --paths temp:$HOME/Downloads/ {1} -o \"{=2 s/[^a-zA-Z0-9 ._-]//g =}_{=3 s/[^a-zA-Z0-9 ._-]//g =}_{=4 s/[^a-zA-Z0-9 ._-]//g =}\"'
```

## Use VPN to Download with yt-dlp


Routes yt-dlp downloads through an isolated WireGuard network namespace and keeping the rest of the system's traffic unaffected.
You can have muliple random conections at the same time. Perfect for use with gnu parallel.   

Here is the script Move script to ~/.local/bin/vpn-yt-dlp and add execution "chmod a+x ~/.local/bin/vpn-yt-dlp". Use like this: "vpn-yt-dlp rand https://www.youtube.com/watch?v=1nnatyEvxQU" or "vpn-yt-dlp us-slc https://www.youtube.com/watch?v=1nnatyEvxQU".
You cann use all the parameter as in yt-dlp just put rand or you VPN at first argument.
```shell
#!/bin/bash
set -e
if [[ $# -lt 2 ]]; then
echo "Usage: $0 <mode|config> <yt-dlp args>"
echo "rand: all (random)"
for r in eu:EU ap:AP am:AM us:AM ea:EA; do
case ${r%:*} in
us) L=$(grep -l '"regionCode":"AM"' /etc/wireguard/*.conf | grep "/us-" | xargs -n1 basename -s .conf | tr '\n' ' ');;
am) L=$(grep -l '"regionCode":"AM"' /etc/wireguard/*.conf | grep -v "/us-" | xargs -n1 basename -s .conf | tr '\n' ' ');;
*) L=$(grep -l "\"regionCode\":\"${r#*:}\"" /etc/wireguard/*.conf | xargs -n1 basename -s .conf | tr '\n' ' ');;
esac
echo "rand-${r%:*}: $L"
done
exit 1
fi
case "$1" in
rand) A=($(ls /etc/wireguard/*.conf | xargs -n1 basename -s .conf));;
rand-eu) A=($(grep -l '"regionCode":"EU"' /etc/wireguard/*.conf | xargs -n1 basename -s .conf));;
rand-ap) A=($(grep -l '"regionCode":"AP"' /etc/wireguard/*.conf | xargs -n1 basename -s .conf));;
rand-am) A=($(grep -l '"regionCode":"AM"' /etc/wireguard/*.conf | grep -v "/us-" | xargs -n1 basename -s .conf));;
rand-us) A=($(grep -l '"regionCode":"AM"' /etc/wireguard/*.conf | grep "/us-" | xargs -n1 basename -s .conf));;
rand-ea) A=($(grep -l '"regionCode":"EA"' /etc/wireguard/*.conf | xargs -n1 basename -s .conf));;
*) [[ -f "/etc/wireguard/$1.conf" ]] && V="$1" || exit 1;;
esac
[[ -z "$V" ]] && { [[ ${#A[@]} -eq 0 ]] && exit 1; V="${A[$RANDOM % ${#A[@]}]}"; }
shift
CN=$(grep -m1 '"country":' "/etc/wireguard/$V.conf" | cut -d'"' -f4)
echo "VPN: $V - $CN" >&2
P=$(shuf -i 2000-65000 -n 1)
while ss -ltn | grep -q ":$P "; do P=$(shuf -i 2000-65000 -n 1); done
C=$(sed 's/#.*//;/^[[:space:]]*$/d' "/etc/wireguard/$V.conf")
PK=$(grep -m1 '^PrivateKey' <<<"$C" | cut -d= -f2- | xargs)
AD=$(grep -m1 '^Address' <<<"$C" | cut -d= -f2- | xargs)
PB=$(grep -m1 '^PublicKey' <<<"$C" | cut -d= -f2- | xargs)
EP=$(grep -m1 '^Endpoint' <<<"$C" | cut -d= -f2- | xargs)
[[ -z "$PK" || -z "$AD" || -z "$PB" || -z "$EP" ]] && exit 1
IP=$(getent hosts "${EP%:*}" | awk '{print $1; exit}')
T=$(mktemp)
trap 'rm -f "$T"; [[ -n "$WP" ]] && kill "$WP" 2>/dev/null' EXIT
printf "[Interface]\nPrivateKey=%s\nAddress=%s\nMTU=1280\n[Peer]\nPublicKey=%s\nEndpoint=%s:%s\nAllowedIPs=0.0.0.0/0\n[Socks5]\nBindAddress=127.0.0.1:%s\n" "$PK" "$AD" "$PB" "$IP" "${EP##*:}" "$P" > "$T"
$HOME/.local/bin/wireproxy -c "$T" >/dev/null 2>&1 & WP=$!
for i in $(seq 1 20); do kill -0 "$WP" 2>/dev/null && nc -z 127.0.0.1 "$P" 2>/dev/null && break || { [[ $i -eq 20 ]] && exit 1 || sleep 0.3; }; done
ALL_PROXY="socks5h://127.0.0.1:$P" $HOME/.local/bin/yt-dlp "$@"
```
For exampe I produce a download list with "nice ionice -c 3 vpn-yt-dlp rand"
```shell
echo "Banda Sinaloa" | parallel --ungroup --silent 'yt-dlp --playlist-end 20 --flat-playlist --simulate --match-filter "id~=^PL" --print id "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' | parallel --ungroup --silent 'yt-dlp --ignore-errors --no-warnings --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(title)s" --flat-playlist {}' | parallel --colsep ';' --ungroup --silent 'printf "nice ionice -c 3 vpn-yt-dlp rand --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata {1} -o \"$HOME/Downloads/CarPlaylist8/{=2 s/[^a-zA-Z0-9 .-]//g; s/^\s+|\s+$//g =}/{=3 s/[^a-zA-Z0-9 .-]//g; s/^\s+|\s+$//g =}.mp3\"\n"'  >do1.sh
```
And then I download 2 times form very cuntry posible:
```shell
cat do1.sh | shuf | parallel --max-procs 16 --ungroup --silent
```
