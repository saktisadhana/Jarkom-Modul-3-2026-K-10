### Soal 17

##### `geto` : aktifkan trusted real-IP

```bash
nginx -V 2>&1 | grep -- --with-http_realip_module
/root/write_geto.sh trusted
```

##### `itadori` : kirim request bertanda

```bash
. /root/lab.env
ip -4 addr show eth0
curl -fsS "http://$ZONE/api?case=network2"
```

##### `ino` : kirim request bertanda

```bash
. /root/lab.env
ip -4 addr show eth0
curl -fsS "http://$ZONE/api?case=network3"
```

##### Client : coba header palsu

```bash
. /root/lab.env
curl -fsS -H 'X-Real-IP: 203.0.113.123' \
    -H 'X-Forwarded-For: 203.0.113.123' \
    "http://$ZONE/api?case=spoof-check"
```

##### `yaga` : cek database

```bash
. /root/lab.env
mariadb "$DB_NAME" -e "SELECT id,worker,client_ip,request_uri,created_at FROM request_logs WHERE request_uri LIKE '%case=%' ORDER BY id DESC LIMIT 12;"
```
