# vpn-yt-dlp   

Routes `yt-dlp` downloads through an isolated WireGuard SOCKS5 proxy via `wireproxy` — leaving all other system traffic untouched und not using root. Supports multiple simultaneous connections; pairs well with `gnu parallel`.
---
## Prepare wireproxy
```bash
curl -sL https://api.github.com/repos/windtf/wireproxy/releases/latest \
  | grep browser_download_url | grep linux_amd64 | cut -d'"' -f4
```
if ok, downoad and install it.
```bash
wget -O /tmp/wireproxy.tar.gz \
  $(curl -sL https://api.github.com/repos/windtf/wireproxy/releases/latest \
    | grep browser_download_url | grep linux_amd64 | cut -d'"' -f4)
tar -xzf /tmp/wireproxy.tar.gz -C ~/.local/bin/ wireproxy
chmod +x ~/.local/bin/wireproxy
~/.local/bin/wireproxy --version
```
---
## Install
```bash
cp vpn-yt-dlp ~/.local/bin/ && chmod a+x ~/.local/bin/vpn-yt-dlp
```
---
## The bash script:
```bash
#!/bin/bash
set -e;W=/etc/wireguard;B(){ xargs -n1 basename -s .conf|tr '\n' ' ';};B2(){ xargs -n1 basename -s .conf;}
T="";WP="";LH=0;WL=""
cleanup(){ [[ -n "$WP" ]]&&{ kill -9 "$WP" 2>/dev/null;wait "$WP" 2>/dev/null||true;};[[ -n "$T" ]]&&rm -f "$T";[[ -n "$WL" ]]&&rm -f "$WL"
[[ $LH -eq 1 ]]&&{ exec 9>&-;[[ -n "$V" ]]&&rm -f "/tmp/vpn_lock_$V"; } }
trap cleanup EXIT INT TERM HUP
if [[ $# -lt 2 ]];then echo "Usage: $0 <mode>; rand: all"
for r in eu:EU ap:AP am:AM us:AM ea:EA;do case ${r%:*} in
us) L=$(grep -l '"regionCode":"AM"' "$W"/*|grep "/us-"|B);;
am) L=$(grep -l '"regionCode":"AM"' "$W"/*|grep -v "/us-"|B);;
*) RC="${r#*:}";L=$(grep -l "\"regionCode\":\"$RC\"" "$W"/*|B);;esac
echo "rand-${r%:*}: $L";done;exit 1;fi
case "$1" in
rand) A=($(ls "$W"/*.conf|B2));;
rand-eu) A=($(grep -l '"regionCode":"EU"' "$W"/*|B2));;
rand-ap) A=($(grep -l '"regionCode":"AP"' "$W"/*|B2));;
rand-am) A=($(grep -l '"regionCode":"AM"' "$W"/*|grep -v "/us-"|B2));;
rand-us) A=($(grep -l '"regionCode":"AM"' "$W"/*|grep "/us-"|B2));;
rand-ea) A=($(grep -l '"regionCode":"EA"' "$W"/*|B2));;
*) [[ -f "$W/$1.conf" ]]&&V="$1"||exit 1;;esac
if [[ -z "$V" ]];then [[ ${#A[@]} -eq 0 ]]&&exit 1
for((i=0;i<100;i++));do TRY_V="${A[$RANDOM%${#A[@]}]}"
exec 9>"/tmp/vpn_lock_$TRY_V";if flock -n 9;then V="$TRY_V";LH=1;break;else exec 9>&-;fi;done
[[ -z "$V" ]]&&{ echo "Error: Busy">&2;exit 1;};else exec 9>"/tmp/vpn_lock_$V";flock 9;LH=1;fi
shift;CN=$(grep -m1 '"country":' "$W/$V.conf"|cut -d'"' -f4);echo "VPN: $V - $CN">&2
if [[ -n "$PARALLEL_JOBSLOT" ]];then P=$((20000+PARALLEL_JOBSLOT))
else P=$((20000+($(printf '%s' "$V"|cksum|cut -d' ' -f1)%20000)));fi
C=$(sed 's/#.*//;/^[[:space:]]*$/d' "$W/$V.conf")
G(){ grep -m1 "^$1" <<<"$C"|cut -d= -f2-|xargs;}
PK=$(G PrivateKey);AD=$(G Address);PB=$(G PublicKey);EP=$(G Endpoint)
[[ -z $PK||-z $AD||-z $PB||-z $EP ]]&&{ echo "Err: config">&2;exit 1;}
IP=$(getent hosts "${EP%:*}"|awk '{print $1;exit}');[[ -z $IP ]]&&{ echo "Err: DNS">&2;exit 1;}
T=$(mktemp);WL=$(mktemp)
printf '[Interface]\nPrivateKey=%s\nAddress=%s\nDNS=1.1.1.1\nMTU=1280\n'\
'[Peer]\nPublicKey=%s\nEndpoint=%s:%s\nAllowedIPs=0.0.0.0/0\n'\
'[Socks5]\nBindAddress=127.0.0.1:%s\n' "$PK" "$AD" "$PB" "$IP" "${EP##*:}" "$P">"$T"
echo "Wait...">&2;$HOME/.local/bin/wireproxy -c "$T" >"$WL" 2>&1 &WP=$!
OK=0;for i in $(seq 1 30);do
kill -0 "$WP" 2>/dev/null||{ echo "Err: wireproxy died">&2;cat "$WL">&2;exit 1;}
curl -x "socks5h://127.0.0.1:$P" -sI -m3 https://1.1.1.1>/dev/null 2>&1&&{ OK=1;break;}
sleep 1;done
[[ $OK -eq 0 ]]&&{ echo "VPN Fail">&2;exit 1;}
ALL_PROXY="socks5h://127.0.0.1:$P" $HOME/.local/bin/yt-dlp "$@"
```
---
## Usage
```bash
vpn-yt-dlp <vpn|mode> [yt-dlp args...]
```
| Mode | Description |
|---|---|
| `rand` | Random config from all available |
| `rand-eu` `rand-ap` `rand-am` `rand-us` `rand-ea` | Random from that region |
| `us-slc` | Specific named config |
All standard `yt-dlp` flags pass through unchanged.
---
## Examples

```bash
# Single download via random VPN
vpn-yt-dlp rand https://www.youtube.com/watch?v=_Es1NbQXDtE
# Specific server
vpn-yt-dlp us-slc https://www.youtube.com/watch?v=_Es1NbQXDtE
```
---
## Batch Download (Parallel)
**1. Build download list**
Search YouTube for playlists, then expand each to individual track commands:
```bash
echo "salsa 2025" | \
parallel --ungroup --silent 'yt-dlp --playlist-end 10 --flat-playlist --simulate --match-filter "id~=^PL" --print id "https://www.youtube.com/results?search_query={= s/ /+/g =}&sp=EgIQAw=="' | \
parallel --ungroup --silent 'yt-dlp --ignore-errors --no-warnings --flat-playlist --print "%(id)s§%(playlist)s§%(title)s" "https://www.youtube.com/playlist?list={}"' | \
grep -vE "Deleted video|Private video" | sed -r ':a; s/^(.{11}.*)[^a-zA-Z0-9 .§_äöüÄÖÜßáéíóúÁÉÍÓÚñÑ-]/\1/; s/^(.{11}.*) /\1_/; s/^(.{11}.*)__+/\1_/; ta' | \
parallel --ungroup --silent --colsep § -- 'echo nice ionice -c 3 vpn-yt-dlp rand-am --extract-audio --audio-format mp3 --audio-quality 5 --embed-thumbnail --embed-metadata --restrict-filename {1} -o "$HOME/Downloads/CarPlaylist/{2}/{3}.mp3"' >1.tmp
> do1.sh
```
**2. Run — 16 parallel workers, shuffled across all regions**

```bash
cat do1.sh | shuf | parallel --max-procs 16 --ungroup --silent
```
---
## VPN Status Check
Tests every config against a sample video and reports which servers are blocked:
```bash
#!/bin/bash
VID="_Es1NbQXDtE"; D="/etc/wireguard"; WP="$HOME/.local/bin/wireproxy"; YT="$HOME/.local/bin/yt-dlp"
printf "ID\t\tSTATUS\t\tCOUNTRY\t\t\t   TIME\t\tMESSAGE\n%s\n" "$(printf '%0.s-' {1..78})"
for f in $D/*.conf; do
  V=$(basename -s .conf "$f"); CN=$(grep -m1 '"country":' "$f" | cut -d'"' -f4)
  P=$(shuf -i 2000-65000 -n1); while ss -ltn | grep -q ":$P "; do P=$(shuf -i 2000-65000 -n1); done
  C=$(sed 's/#.*//;/^[[:space:]]*$/d' "$f"); T=$(mktemp)
  PK=$(grep -m1 '^PrivateKey' <<<"$C"|cut -d= -f2-|xargs); AD=$(grep -m1 '^Address' <<<"$C"|cut -d= -f2-|xargs)
  PB=$(grep -m1 '^PublicKey' <<<"$C"|cut -d= -f2-|xargs); EP=$(grep -m1 '^Endpoint' <<<"$C"|cut -d= -f2-|xargs)
  IP=$(getent hosts "${EP%:*}"|awk '{print $1;exit}')
  printf "[Interface]\nPrivateKey=%s\nAddress=%s\nMTU=1280\n[Peer]\nPublicKey=%s\n" "$PK" "$AD" "$PB" >"$T"
  printf "Endpoint=%s:%s\nAllowedIPs=0.0.0.0/0\n[Socks5]\nBindAddress=127.0.0.1:%s\n" "$IP" "${EP##*:}" "$P" >>"$T"
  START=$(date +%s%3N); $WP -c "$T" >/dev/null 2>&1 & WID=$!; READY=0
  for i in $(seq 1 10); do kill -0 "$WID" 2>/dev/null && nc -z 127.0.0.1 "$P" 2>/dev/null && READY=1 && break || sleep 0.2; done
  if [ $READY -eq 1 ]; then
 OUT=$(ALL_PROXY="socks5h://127.0.0.1:$P" $YT --get-title --simulate "$VID" 2>&1); RET=$?
 E=$(( $(date +%s%3N)-START ))
 if   [ $RET -eq 0 ];   then printf "%-15s \e[32mWORKING\e[0m\t\t%-24s%7dms\tOK\n" "$V" "$CN" "$E"
 elif echo "$OUT"|grep -qi "confirm you're not a bot"; then printf "%-15s \e[31mBLOCKED\e[0m\t\t%-24s%7dms\tBot-Check\n" "$V" "$CN" "$E"
 else printf "%-15s \e[33mERROR\e[0m\t\t%-24s%7dms\tOther Error\n" "$V" "$CN" "$E"; fi
  else
 E=$(( $(date +%s%3N)-START ))
 printf "%-15s \e[35mCRASHED\e[0m\t\t%-24s%7dms\tProxy Fail\n" "$V" "$CN" "$E"
  fi
  kill "$WID" 2>/dev/null; wait "$WID" 2>/dev/null; rm -f "$T"
done
```
Here you see the result:
```
ID              STATUS          COUNTRY              MESSAGE
----------------------------------------------------------------------
us-slc          WORKING         United States        OK
de-fra          BLOCKED         Germany              Bot-Check
nl-ams          ERROR           Netherlands          Other Error
sg-sin          CRASHED         Singapore            Proxy Fail
```
Exit states: `WORKING` · `BLOCKED` (bot-check) · `ERROR` (other) · `CRASHED` (proxy failed to start)
---
## License
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)

