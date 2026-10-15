### Soal 6

##### `tengen` : host fixed-address

```bash
. /root/lab.env
. /root/client-macs.env
cat > /root/dhcp/hosts.conf <<EOF
host kugisaki {
    hardware ethernet $KUGISAKI_MAC;
    fixed-address ${P}.3.99;
}
EOF
/root/apply_dhcp.sh
```

##### `kugisaki` : perbarui dan cek

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
ip -4 addr show eth0
ip route
cat /etc/resolv.conf
```
