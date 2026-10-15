### Persiapan

##### Semua node : variabel bersama

```bash
cat > /root/lab.env <<'EOF'
P='192.216'
GROUP_ID='K10'
case "$P" in
    REPLACE*) echo 'Set P in /root/lab.env first' >&2; return 1 ;;
esac
case "$GROUP_ID" in
    REPLACE*|''|*[!a-zA-Z0-9_]*)
        echo 'Set a valid GROUP_ID in /root/lab.env' >&2
        return 1 ;;
esac
GROUP_DNS=$(printf '%s' "$GROUP_ID" | tr '[:upper:]' '[:lower:]')
SEARCH_DOMAIN="${GROUP_DNS}.com"
ZONE="jujutsu.${SEARCH_DOMAIN}"
DB_NAME="db_shibuya_${GROUP_ID}"
UTAHIME="${P}.1.2"
SHOKO="${P}.1.3"
GOJO="${P}.1.4"
TENGEN="${P}.4.10"
GETO="${P}.4.11"
MAKI="${P}.4.20"
PANDA="${P}.4.21"
NANAMI="${P}.4.22"
TODO="${P}.4.23"
KIRARA="${P}.4.24"
YAGA="${P}.4.30"
INUMAKI="${P}.4.31"
FIRST_OCTET=${P%.*}
SECOND_OCTET=${P#*.}
REV_ZONE="1.${SECOND_OCTET}.${FIRST_OCTET}.in-addr.arpa"
GOJO_PTR=${GOJO##*.}
EOF
. /root/lab.env
```

##### Semua node : resolver awal

```bash
printf 'nameserver 192.168.122.1\n' > /etc/resolv.conf
```

```bash
ls /root/lab.env && . /root/lab.env && echo "$P $ZONE"
```
