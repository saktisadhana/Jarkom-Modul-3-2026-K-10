### Soal 18

##### `geto` : config upstream akhir

```bash
/root/write_geto.sh backup
cat /root/nginx/geto.conf
```

##### Client : kondisi normal

```bash
. /root/lab.env
for n in $(seq 1 8); do
    curl -fsS "http://$ZONE/api?case=healthy"
done
```

##### `nanami` & `todo` : matikan nginx

```bash
service nginx stop
ss -ltnp | grep ':80' || true
```

##### Client : lihat respons dari backup

```bash
. /root/lab.env
for n in $(seq 1 12); do
    curl -sS -w ' HTTP=%{http_code}\n' \
        "http://$ZONE/api?case=failover"
    sleep 1
done
```

##### `nanami` & `todo` : nyalakan lagi

```bash
service nginx start
```
