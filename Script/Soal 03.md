### Soal 3

##### `jichi` : konfigurasi isc-dhcp-relay

```bash
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-relay
. /root/lab.env
cat > /root/isc-dhcp-relay.default <<EOF
SERVERS="$TENGEN"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
EOF
cp /root/isc-dhcp-relay.default /etc/default/isc-dhcp-relay
service isc-dhcp-relay restart
service isc-dhcp-relay status
ps -ef | grep '[d]hcrelay'
```

##### `itadori` & `ino` : minta lease uji

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
ip link set eth0 up
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
ip -4 addr show eth0
ip route
cat /etc/resolv.conf
```
