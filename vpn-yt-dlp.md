# vpn-yt-dlp
---
### Routes `yt-dlp` downloads through an isolated WireGuard SOCKS5 proxy via `wireproxy` — leaving all other system traffic untouched und not using root. Supports multiple simultaneous connections; pairs well with `gnu parallel`.
---
## Install
```bash
cp vpn-yt-dlp ~/.local/bin/ && chmod a+x ~/.local/bin/vpn-yt-dlp
```
## The bash script:
---
```bash
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

