### Soal 4

##### Semua client : catat MAC eth0

```bash
hostname
cat /sys/class/net/eth0/address
```

##### `tengen` : simpan MAC client

```bash
cat > /root/client-macs.env <<'EOF'
ITADORI_MAC='REPLACE_ITADORI_MAC'
FUSHIGURO_MAC='REPLACE_FUSHIGURO_MAC'
ZENIN_MAC='REPLACE_ZENIN_MAC'
OKKOTSU_MAC='REPLACE_OKKOTSU_MAC'
INO_MAC='REPLACE_INO_MAC'
KUSAKABE_MAC='REPLACE_KUSAKABE_MAC'
MIWA_MAC='REPLACE_MIWA_MAC'
KUGISAKI_MAC='REPLACE_KUGISAKI_MAC'
EOF
```

##### `tengen` : class dan pool

```bash
. /root/lab.env
. /root/client-macs.env
cat > /root/dhcp/classes.conf <<EOF
class "division-a-pool1" {
    match hardware;
}
subclass "division-a-pool1" 1:$ITADORI_MAC;
subclass "division-a-pool1" 1:$FUSHIGURO_MAC;
class "division-a-pool2" {
    match hardware;
}
subclass "division-a-pool2" 1:$ZENIN_MAC;
subclass "division-a-pool2" 1:$OKKOTSU_MAC;
class "division-b-short" {
    match hardware;
}
subclass "division-b-short" 1:$INO_MAC;
subclass "division-b-short" 1:$KUSAKABE_MAC;
class "division-b-long" {
    match hardware;
}
subclass "division-b-long" 1:$MIWA_MAC;
EOF
cat > /root/dhcp/net2.conf <<EOF
subnet ${P}.2.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option routers ${P}.2.1;
    option broadcast-address ${P}.2.255;
    pool {
        allow members of "division-a-pool1";
        range ${P}.2.10 ${P}.2.35;
    }
    pool {
        allow members of "division-a-pool2";
        range ${P}.2.70 ${P}.2.95;
    }
}
EOF
/root/apply_dhcp.sh
```
