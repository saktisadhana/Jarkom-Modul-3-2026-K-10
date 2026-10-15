### Soal 11

##### `utahime` & `shoko` : install BIND

```bash
apt-get update
apt-get install -y bind9 bind9-utils dnsutils
mkdir -p /root/bind
```

##### `utahime` & `shoko` : recursion untuk client internal

```bash
. /root/lab.env
cat > /root/bind/named.conf.options <<EOF
acl lab_clients {
    127.0.0.1;
    ${P}.1.0/24;
    ${P}.2.0/24;
    ${P}.3.0/24;
    ${P}.4.0/24;
};
options {
    directory "/var/cache/bind";
    listen-on { any; };
    listen-on-v6 { none; };
    allow-query { lab_clients; };
    recursion yes;
    allow-recursion { lab_clients; };
    forwarders { 192.168.122.1; };
    forward only;
    dnssec-validation auto;
};
EOF
cp /root/bind/named.conf.options /etc/bind/named.conf.options
```

##### `utahime` : zona primary dan record SOA/NS

```bash
. /root/lab.env
cat > /root/bind/named.conf.local <<EOF
zone "$ZONE" {
    type primary;
    file "/etc/bind/db.$ZONE";
    notify yes;
    also-notify { $SHOKO; };
    allow-transfer { $SHOKO; };
};
EOF
cat > "/root/bind/db.$ZONE" <<EOF
\$TTL 300
@ IN SOA utahime.$ZONE. admin.$ZONE. (
    2026101501 300 120 604800 300 )
@       IN NS utahime.$ZONE.
@       IN NS shoko.$ZONE.
utahime IN A $UTAHIME
shoko   IN A $SHOKO
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
cp "/root/bind/db.$ZONE" "/etc/bind/db.$ZONE"
named-checkconf
named-checkzone "$ZONE" "/etc/bind/db.$ZONE"
service named restart
service named status
```

##### `shoko` : zona secondary

```bash
. /root/lab.env
cat > /root/bind/named.conf.local <<EOF
zone "$ZONE" {
    type secondary;
    primaries { $UTAHIME; };
    file "/var/cache/bind/db.$ZONE";
    allow-transfer { none; };
};
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
named-checkconf
service named restart
service named status
```

##### Satu client Alpine : install dig dan bandingkan

```bash
apk update
apk add bind-tools curl
. /root/lab.env
dig @"$UTAHIME" "$ZONE" SOA +norecurse
dig @"$SHOKO" "$ZONE" SOA +norecurse
dig @"$UTAHIME" "$ZONE" NS +short
dig @"$SHOKO" "$ZONE" NS +short
```

##### `utahime` : uji transfer otomatis

```bash
. /root/lab.env
# Run this change once; use a higher serial if already advanced.
sed -i 's/2026101501/2026101502/' "/root/bind/db.$ZONE"
printf 'sync-test IN A %s\n' "$GOJO" >> "/root/bind/db.$ZONE"
cp "/root/bind/db.$ZONE" "/etc/bind/db.$ZONE"
named-checkzone "$ZONE" "/etc/bind/db.$ZONE" &&
    rndc reload "$ZONE"
```
