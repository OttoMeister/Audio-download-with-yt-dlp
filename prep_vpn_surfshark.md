# Surfshark WireGuard Setup Guide
## Download Surfshark WireGuard Configuration
Install Extension: Install the "Get cookies.txt LOCALLY" extension in a Chromium-based browser.  
Export Cookies: Log in to Surfshark WireGuard Setup and export the cookies as my.surfshark.com_cookies.txt to your Downloads folder. Run the following command to download the raw configuration data:  (Downloadfile is like 200kByte)
```shell
curl -Lb ~/Downloads/my.surfshark.com_cookies.txt -A "Mozilla/5.0" -L -o ~/Downloads/surfshark_wireguard.conf https://my.surfshark.com/vpn/manual-setup/main/wireguard
```
# Generate Profiles
Save the script below as ~/Downloads/make-wg-conf-surfshark.sh, update the Address and PrivateKey fields, and execute it with bash ~/Downloads/make-wg-conf-surfshark.sh. Individual profiles will be created in ~/Downloads/wireguard/.
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
Address = <insert_your_ip_addewss_here>
PrivateKey = <insert_your_private_key_here>
DNS = 1.1.1.1, 8.8.8.8
[Peer]
PublicKey = $pub_key
AllowedIPs = 0.0.0.0/0
Endpoint = $conn_name:51820
EOF
done
```
## Check first WireGuard is working: 
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
## Debugg
```shell
journalctl -xe
curl ifconfig.me; echo
ip a | sed -n '/scope global/ { /-/ s/.* //p }'
```
Check website "https://browserleaks.com/ip"






