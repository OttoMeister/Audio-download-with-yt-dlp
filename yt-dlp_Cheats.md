# More tools for yt-dlp

## Download compleat playlists of videos in max resolution and anhanced audio
```shell
echo PLffX9yMQbaQhZbTRat79oh1kHhGHwv1Sn | parallel -d ' ' 'yt-dlp --ignore-errors --no-abort-on-error --no-warnings --no-check-certificate --print "https://www.youtube.com/watch?v=%(id)s;%(playlist)s;%(playlist_index)03d;%(title)s" --flat-playlist {}' | nice ionice -c 3 parallel --colsep ';' --ungroup --max-procs 4 'yt-dlp --force-ipv4 -f \"bestvideo\[height\<=1080\]\[vcodec=vp9\]+bestaudio\[ext=webm\]/bestvideo\[height\<=1080\]+bestaudio/best\[height\<=1080\]\" --extractor-args \"youtube:player_client=web,default\" --postprocessor-args \"ffmpeg:-c:v copy -c:a libopus -b:a 128k -filter:a loudnorm=I=-14:TP=-1.5:LRA=11\" --continue --retries 10 --fragment-retries 15 --retry-sleep fragment:10 --concurrent-fragments 4 --socket-timeout 20 --buffer-size 64K --http-chunk-size 5M --no-playlist --embed-metadata --paths home:$HOME/Downloads/VideoPlaylist --paths temp:$HOME/Downloads/ {1} -o \"{=2 s/[^a-zA-Z0-9 ._-]//g =}_{=3 s/[^a-zA-Z0-9 ._-]//g =}_{=4 s/[^a-zA-Z0-9 ._-]//g =}\"'
```

## Use VPN to Download with yt-dlp


Routes yt-dlp downloads through an isolated WireGuard network namespace and keeping the rest of the system's traffic unaffected.
You can have muliple random conections at the same time. Perfect for use with gnu parallel.


Check first WireGuard is working: 
```shell
sudo -i 
ls /etc/wireguard/*.conf | xargs -n 1 basename -s .conf | tr '\n' ' '; echo
# ar-bua at-vie ch-zur co-bog de-ber de-fra ec-uio es-mad fr-par ie-dub it-mil jp-tok kr-seo mx-qro pa-pac pe-lim pt-lis py-asu se-sto tr-ist tw-tai us-bos us-chi us-dal us-mia us-nyc us-slc ve-car
# start - status - stop
wg-quick up ch-zur
wg
ip a show ch-zur
wg-quick down ch-zur
```
Here is the script Move script to ~/.local/bin/vpn-yt-dlp and add execution "chmod a+x ~/.local/bin/vpn-yt-dlp". Use like this: "vpn-yt-dlp rand https://www.youtube.com/watch?v=1nnatyEvxQU" or "vpn-yt-dlp us-slc https://www.youtube.com/watch?v=1nnatyEvxQU".
You cann use all the parameter as in yt-dlp just put rand or you VPN at first argument.
```shell
#!/usr/bin/env bash
set -euo pipefail
WG_DIR="/etc/wireguard"; NS="vpn-yt-$$"; WG_IF="wg-yt-$$"
[[ $EUID -ne 0 ]] && exec sudo --preserve-env=HOME "$0" "$@"
REAL_HOME=$(getent passwd "${SUDO_USER:-$USER}" | cut -d: -f6)
YTDLP="$REAL_HOME/.local/bin/yt-dlp"; DENO="$REAL_HOME/.local/bin/deno"
die() { echo "Error: $*"; exit 1; }
[[ $# -lt 2 ]] && { echo "Usage: $(basename "$0") <rand|vpn> [yt-dlp-opts]"
  echo -n "VPNs: "; ls "$WG_DIR"/*.conf | xargs -n1 basename -s .conf | tr '\n' ' '; echo; exit 1; }
AVAILABLE=($(ls "$WG_DIR"/*.conf | xargs -n1 basename -s .conf))
[[ "$1" == "rand" ]] && VPN="${AVAILABLE[$RANDOM % ${#AVAILABLE[@]}]}" \
  || { [[ -f "$WG_DIR/$1.conf" ]] && VPN="$1" || die "'$1' unknown."; }
shift
trap 'ip netns exec "$NS" ip link delete "$WG_IF" 2>/dev/null||true
      ip netns delete "$NS" 2>/dev/null||true; rm -rf "/etc/netns/$NS"' EXIT INT TERM
S=$(sed 's/#.*//;/^[[:space:]]*$/d' "$WG_DIR/$VPN.conf")
gi() { echo "$S" | grep -m1 "^$1" | cut -d= -f2- | sed 's/^ *//;s/ *$//'; }
gp() { echo "$S" | grep -A5 '^\[Peer\]' | grep -m1 "^$1" | cut -d= -f2- | sed 's/^ *//;s/ *$//'; }
PK=$(gi PrivateKey); ADDR=$(gi Address); DNS=$(gi DNS)
PUBK=$(gp PublicKey); EP=$(gp Endpoint); AIPS=$(gp AllowedIPs); PSK=$(gp PresharedKey||true)
EP_IP=$(getent hosts "${EP%:*}" | awk '{print $1;exit}')
[[ -z "$EP_IP" ]] && die "DNS lookup failed for ${EP%:*}."
echo "[vpn-yt-dlp] VPN=$VPN EP=$EP_IP"
ip netns add "$NS"; ip netns exec "$NS" ip link set lo up
ip link add "$WG_IF" type wireguard; ip link set "$WG_IF" netns "$NS"
ip netns exec "$NS" wg set "$WG_IF" private-key <(echo "$PK") listen-port 0 \
  peer "$PUBK" ${PSK:+preshared-key <(echo "$PSK")} \
  endpoint "$EP_IP:${EP##*:}" allowed-ips "$AIPS" persistent-keepalive 25
echo "$ADDR" | tr ',' '\n' | sed 's/^ *//;s/ *$//' \
  | xargs -r -I{} ip netns exec "$NS" ip addr add {} dev "$WG_IF"
ip netns exec "$NS" ip link set "$WG_IF" up; ip netns exec "$NS" ip route add default dev "$WG_IF"
mkdir -p "/etc/netns/$NS"; echo "nameserver ${DNS%%,*}" > "/etc/netns/$NS/resolv.conf"
echo -n "[vpn-yt-dlp] External IP: "
ip netns exec "$NS" curl -s --max-time 5 https://api.ipify.org || echo "unknown"; echo
ip netns exec "$NS" sudo -u "${SUDO_USER:-$USER}" "$YTDLP" --js-runtimes "deno:$DENO" "$@"
```


