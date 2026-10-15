### Soal 2

##### `tengen` : install dan siapkan file konfigurasi

```bash
apt-get update
apt-get install -y isc-dhcp-server tcpdump
mkdir -p /root/dhcp /etc/dhcp/lab
. /root/lab.env
cat > /root/dhcp/dhcpd.conf <<EOF
authoritative;
ddns-update-style none;
default-lease-time 3600;
max-lease-time 7200;
option domain-name "$SEARCH_DOMAIN";
option domain-search "$SEARCH_DOMAIN";
option domain-name-servers $UTAHIME, $SHOKO;
include "/etc/dhcp/lab/classes.conf";
include "/etc/dhcp/lab/net2.conf";
include "/etc/dhcp/lab/net3.conf";
include "/etc/dhcp/lab/net4.conf";
include "/etc/dhcp/lab/hosts.conf";
EOF
: > /root/dhcp/classes.conf
: > /root/dhcp/hosts.conf
cat > /root/dhcp/net4.conf <<EOF
subnet ${P}.4.0 netmask 255.255.255.0 {
    option routers ${P}.4.1;
    option broadcast-address ${P}.4.255;
}
EOF
cat > /root/dhcp/net2.conf <<EOF
subnet ${P}.2.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option routers ${P}.2.1;
    option broadcast-address ${P}.2.255;
    range ${P}.2.10 ${P}.2.35;
}
EOF
cat > /root/dhcp/net3.conf <<EOF
subnet ${P}.3.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option routers ${P}.3.1;
    option broadcast-address ${P}.3.255;
    range ${P}.3.10 ${P}.3.89;
}
EOF
cat > /root/isc-dhcp-server.default <<'EOF'
INTERFACESv4="eth0"
INTERFACESv6=""
EOF
cp /root/isc-dhcp-server.default /etc/default/isc-dhcp-server
```

##### `tengen` : script deploy yang bisa dipakai ulang

```bash
cat > /root/apply_dhcp.sh <<'EOF'
#!/bin/sh
set -eu
mkdir -p /etc/dhcp/lab
cp /root/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf
for file in classes net2 net3 net4 hosts; do
    cp "/root/dhcp/${file}.conf" "/etc/dhcp/lab/${file}.conf"
done
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
service isc-dhcp-server status
EOF
chmod +x /root/apply_dhcp.sh
/root/apply_dhcp.sh
ss -lunp | grep ':67'
```
