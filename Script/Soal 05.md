### Soal 5

##### `tengen` : ganti scope Network 3

```bash
. /root/lab.env
cat > /root/dhcp/net3.conf <<EOF
subnet ${P}.3.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option routers ${P}.3.1;
    option broadcast-address ${P}.3.255;
    pool {
        allow members of "division-b-short";
        range ${P}.3.10 ${P}.3.89;
        min-lease-time 1800;
        default-lease-time 1800;
        max-lease-time 1800;
    }
    pool {
        allow members of "division-b-long";
        range ${P}.3.100 ${P}.3.199;
        min-lease-time 43200;
        default-lease-time 43200;
        max-lease-time 43200;
    }
}
EOF
/root/apply_dhcp.sh
cat /etc/dhcp/dhcpd.conf
```
