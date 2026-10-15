### Soal 1

##### `jichi` : konfigurasi interface permanen

```ini
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.216.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 192.216.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 192.216.3.1
    netmask 255.255.255.0

auto eth4
iface eth4 inet static
    address 192.216.4.1
    netmask 255.255.255.0
```

##### `jichi` : install paket dan buat script router

```bash
apt-get update
apt-get install -y iptables iproute2 iputils-ping curl tcpdump
cat > /root/start_router.sh <<'EOF'
#!/bin/sh
set -eu
. /root/lab.env
sysctl -w net.ipv4.ip_forward=1
for n in 1 2 3 4; do
    lan="eth${n}"
    iptables -C FORWARD -i "$lan" -j ACCEPT 2>/dev/null ||
        iptables -A FORWARD -i "$lan" -j ACCEPT
    iptables -t nat -C POSTROUTING -s "${P}.${n}.0/24" \
        -o eth0 -j MASQUERADE 2>/dev/null ||
        iptables -t nat -A POSTROUTING -s "${P}.${n}.0/24" \
        -o eth0 -j MASQUERADE
 done
iptables -C FORWARD -i eth0 -m conntrack \
    --ctstate ESTABLISHED,RELATED -j ACCEPT 2>/dev/null ||
    iptables -A FORWARD -i eth0 -m conntrack \
    --ctstate ESTABLISHED,RELATED -j ACCEPT
EOF
chmod +x /root/start_router.sh
/root/start_router.sh
printf 'net.ipv4.ip_forward=1\n' > /root/99-shibuya-forward.conf
cp /root/99-shibuya-forward.conf /etc/sysctl.d/
```

##### Semua host infrastruktur : alamat statis

```ini
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.216.4.10
    netmask 255.255.255.0
    gateway 192.216.4.1
```

##### Semua host infrastruktur : cek koneksi

```bash
ip -br addr
ip route
ip route get 1.1.1.1
# Replace this with the gateway for the current node.
ping -c 3 192.216.4.1
ping -c 3 1.1.1.1
ping -c 3 google.com
```

##### `jichi` : cek forwarding dan NAT

```bash
sysctl net.ipv4.ip_forward
iptables -S FORWARD
iptables -t nat -S POSTROUTING
```
