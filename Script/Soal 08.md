### Soal 8

##### `tengen` : mulai capture

```bash
mkdir -p /root/captures
tcpdump -ni eth0 -s 0 -w /root/captures/dora.pcap \
    'udp port 67 or udp port 68'
```

##### Satu client Network 2 atau 3 : mulai DORA baru

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
```

##### `tengen` : baca hasil capture

```bash
tcpdump -nn -vvv -r /root/captures/dora.pcap
```
