### Soal 12

##### `utahime` : update zona forward dan reverse

```bash
. /root/lab.env
cat > "/root/bind/db.$ZONE" <<EOF
\$TTL 300
@ IN SOA utahime.$ZONE. admin.$ZONE. (
    2026101503 300 120 604800 300 )
@         IN NS utahime.$ZONE.
@         IN NS shoko.$ZONE.
@         IN A $GOJO
www       IN CNAME $ZONE.
utahime   IN A $UTAHIME
shoko     IN A $SHOKO
gojo      IN A $GOJO
sync-test IN A $GOJO
api       IN A $GETO
maki      IN A $MAKI
panda     IN A $PANDA
nanami    IN A $NANAMI
todo      IN A $TODO
kirara    IN A $KIRARA
EOF
cat > "/root/bind/db.reverse-net1" <<EOF
\$TTL 300
@ IN SOA utahime.$ZONE. admin.$ZONE. (
    2026101503 300 120 604800 300 )
@ IN NS utahime.$ZONE.
@ IN NS shoko.$ZONE.
$GOJO_PTR IN PTR $ZONE.
EOF
cat > /root/bind/named.conf.local <<EOF
zone "$ZONE" {
    type primary;
    file "/etc/bind/db.$ZONE";
    notify yes;
    also-notify { $SHOKO; };
    allow-transfer { $SHOKO; };
};
zone "$REV_ZONE" {
    type primary;
    file "/etc/bind/db.reverse-net1";
    notify yes;
    also-notify { $SHOKO; };
    allow-transfer { $SHOKO; };
};
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
cp "/root/bind/db.$ZONE" "/etc/bind/db.$ZONE"
cp /root/bind/db.reverse-net1 /etc/bind/db.reverse-net1
named-checkconf
named-checkzone "$ZONE" "/etc/bind/db.$ZONE"
named-checkzone "$REV_ZONE" /etc/bind/db.reverse-net1
service named reload
```

##### `shoko` : reverse zone secondary

```bash
. /root/lab.env
cat >> /root/bind/named.conf.local <<EOF
zone "$REV_ZONE" {
    type secondary;
    primaries { $UTAHIME; };
    file "/var/cache/bind/db.reverse-net1";
    allow-transfer { none; };
};
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
named-checkconf && service named reload
```

##### Client : cek kedua server dan forwarding

```bash
. /root/lab.env
for dns in "$UTAHIME" "$SHOKO"; do
    dig @"$dns" "$ZONE" A +short
    dig @"$dns" "www.$ZONE" CNAME +short
    dig @"$dns" -x "$GOJO" +short
    dig @"$dns" google.com A +short
done
cat /etc/resolv.conf
ping -c 3 google.com
```
