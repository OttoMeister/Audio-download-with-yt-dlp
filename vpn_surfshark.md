# Surfshark WireGuard Setup
---
## 1. Download Config Data
Install the **[Get cookies.txt LOCALLY](https://chrome.google.com/webstore/detail/get-cookiestxt-locally/)** extension in a Chromium browser.  
Log in to `my.surfshark.com`, export cookies as `my.surfshark.com_cookies.txt` to `~/Downloads/`, then fetch the raw config (~200 KB):
```shell
curl -Lb ~/Downloads/my.surfshark.com_cookies.txt -A "Mozilla/5.0" -L -o ~/Downloads/surfshark_wireguard.conf https://my.surfshark.com/vpn/manual-setup/main/wireguard
```
---
## 2. Generate Per-Server Profiles
Save as `~/Downloads/make-wg-conf-surfshark.sh`.  
Edit `Address` and `PrivateKey`, then run `bash ~/Downloads/make-wg-conf-surfshark.sh`.  
Output: individual `.conf` files in `~/Downloads/wireguard/`.
```shell
#!/bin/bash
INPUT_FILE=$HOME/Downloads/surfshark_wireguard.conf
WG_DIR=$HOME/Downloads/wireguard
mkdir -p $WG_DIR
grep -o 'window.initialState={.*};' $INPUT_FILE | sed 's/window.initialState=//; s/;$//' | jq -c '.manualSetup.clusters[]' | while read -r line; do
conn_name=$(echo "$line" | jq -r .connectionName)
pub_key=$(echo "$line" | jq -r .pubKey)
filename=$(echo $conn_name | cut -c 1-6).conf
echo "$line" | jq -r 'to_entries | .[] | if .key == "info" then .value[] | to_entries | .[] | if .key == "entry" then "\"info\":\"entry\":\"value\":\(.value.value | @json)" else "\"info\":\"\(.key)\":\(.value | @json)" end else "\"\(.key)\":\(.value | @json)" end' | sed 's/^/# /' > $WG_DIR/$filename
cat <<EOF >> $WG_DIR/$filename
[Interface]
Address = <insert_your_ip_address_here>
PrivateKey = <insert_your_private_key_here>
DNS = 1.1.1.1, 8.8.8.8
[Peer]
PublicKey = $pub_key
AllowedIPs = 0.0.0.0/0
Endpoint = $conn_name:51820
EOF
done
```
---
## 3. Deploy to System
```shell
sudo cp ~/Downloads/wireguard/*.conf /etc/wireguard
sudo bash -c 'chown root:root /etc/wireguard/*.conf'
sudo bash -c 'chmod 600 /etc/wireguard/*.conf'
```
---
## 4. Verify
```shell
sudo -i 
ls /etc/wireguard/*.conf | xargs -n 1 basename -s .conf | tr '\n' ' '; echo
# ar-bua at-vie ch-zur co-bog de-ber de-fra ec-uio es-mad fr-par ie-dub it-mil jp-tok kr-seo mx-qro pa-pac pe-lim pt-lis py-asu se-sto tr-ist tw-tai us-bos us-chi us-dal us-mia us-nyc us-slc ve-car
wg-quick up ch-zur    # start 
wg                    # status
ip a show ch-zur      # status
wg-quick down ch-zur  # stop
```
---
## 5. Debug
```shell
journalctl -xe
curl ifconfig.me; echo
ip a | sed -n '/scope global/ { /-/ s/.* //p }'
curl -s --max-time 5 https://api.ipify.org || echo "unknown"; echo
xdg-open "https://browserleaks.com/ip"&
```
---
## 6. List Configs by Region

```shell
for r in "EU:Europe" "AP:Asia Pacific" "AM:Americas" "EA:Europe & Africa / Middle East"; do echo -e "\n\"regionCode\":\"${r%%:*}\" = ${r#*:}"; grep -l "\"regionCode\":\"${r%%:*}\"" /etc/wireguard/*.conf | sed 's|.*/||;s|\.conf||' | tr '\n' ' '; echo; done; echo
```
| Code | Region |
|---|---|
| `EU` | Europe |
| `AP` | Asia Pacific |
| `AM` | Americas |
| `EA` | Europe & Africa / Middle East |


---
## License
[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)
