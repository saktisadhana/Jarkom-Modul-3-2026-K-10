### Soal 19

##### `gojo` : aktifkan cache frontend

```bash
/root/write_gojo.sh cached
```

##### Client : MISS lalu HIT di URL yang sama

```bash
. /root/lab.env
TOKEN=$(date +%s)
TEST_URL="http://$ZONE/?cachetest=$TOKEN"
curl -sS -D /tmp/cache-first.headers -o /tmp/cache-first.html "$TEST_URL"
curl -sS -D /tmp/cache-second.headers -o /tmp/cache-second.html "$TEST_URL"
cat /tmp/cache-first.headers
cat /tmp/cache-second.headers
cmp /tmp/cache-first.html /tmp/cache-second.html
```

##### `gojo` : aktifkan rate limit API

```bash
cat > /root/nginx/api-rate.conf <<'EOF'
limit_req zone=api_per_ip;
limit_req_status 429;
limit_req_log_level notice;
EOF
cp /root/nginx/api-rate.conf /etc/nginx/snippets/shibuya-rate.conf
nginx -t && service nginx reload
```

##### Client : kirim request API bersamaan

```bash
. /root/lab.env
mkdir -p /tmp/shibuya-rate
for n in $(seq 1 40); do
    curl -sS -o /dev/null -w '%{http_code}\n' \
        "http://$ZONE/api?ratecase=$n" > "/tmp/shibuya-rate/$n.status" &
done
wait
cat /tmp/shibuya-rate/*.status | sort | uniq -c
```

##### `gojo` : cek alasan di access log

```bash
tail -n 60 /var/log/nginx/gojo_access.jsonl
```
