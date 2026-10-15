### Soal 9

##### `jichi` : hentikan relay

```bash
service isc-dhcp-relay stop
pgrep -a dhcrelay || true
```

##### `itadori` atau `ino` : minta IP tanpa IP lama

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
udhcpc -i eth0 -n -q -t 3 -T 2 -O 119
printf 'DHCP command exit status: %s\n' "$?"
ip -4 addr show eth0
ip route
```

##### `jichi` : nyalakan relay

```bash
service isc-dhcp-relay start
service isc-dhcp-relay status
```

##### Client yang sama : minta IP lagi

```bash
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
ip -4 addr show eth0
ip route
```
