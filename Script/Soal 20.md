### Soal 20

##### `kusakabe` : install ab

```bash
apk update
apk add apache2-utils curl
mkdir -p /root/benchmark
. /root/lab.env
```

##### `gojo` : matikan rate limit untuk baseline

```bash
printf '# API limiting disabled for controlled baseline\n' \
    > /root/nginx/api-rate.conf
cp /root/nginx/api-rate.conf /etc/nginx/snippets/shibuya-rate.conf
nginx -t && service nginx reload
```

##### `kusakabe` : baseline

```bash
. /root/lab.env
curl -fsS -D /root/benchmark/root-warm.headers \
    -o /dev/null "http://$ZONE/"
date -Iseconds > /root/benchmark/baseline-start.txt
ab -n 600 -c 30 "http://$ZONE/" \
    > /root/benchmark/frontend-baseline.txt 2>&1
ab -l -n 600 -c 30 "http://$ZONE/api?bench=baseline" \
    > /root/benchmark/api-baseline.txt 2>&1
cat /root/benchmark/frontend-baseline.txt
cat /root/benchmark/api-baseline.txt
```

##### `gojo` : tandai log dan nyalakan rate limit

```bash
wc -l < /var/log/nginx/gojo_access.jsonl > /root/api-active-log-start.txt
cat > /root/nginx/api-rate.conf <<'EOF'
limit_req zone=api_per_ip;
limit_req_status 429;
limit_req_log_level notice;
EOF
cp /root/nginx/api-rate.conf /etc/nginx/snippets/shibuya-rate.conf
nginx -t && service nginx reload
```

##### `kusakabe` : ulangi dengan rate limit aktif

```bash
. /root/lab.env
date -Iseconds > /root/benchmark/active-start.txt
ab -n 600 -c 30 "http://$ZONE/" \
    > /root/benchmark/frontend-active.txt 2>&1
ab -l -n 600 -c 30 "http://$ZONE/api?bench=active" \
    > /root/benchmark/api-active.txt 2>&1
cat /root/benchmark/frontend-active.txt
cat /root/benchmark/api-active.txt
```

##### `gojo` : ringkas request API

```bash
START_LINE=$(cat /root/api-active-log-start.txt)
tail -n "+$((START_LINE + 1))" /var/log/nginx/gojo_access.jsonl \
    > /root/api-active-run.jsonl
python3 - <<'EOF'
import json
from collections import Counter
from pathlib import Path
rows = []
for line in Path('/root/api-active-run.jsonl').read_text().splitlines():
    item = json.loads(line)
    if item['uri'].startswith('/api?bench=active'):
        rows.append(item)
print('Requests recorded at gojo:', len(rows))
print('HTTP status counts:', dict(Counter(r['status'] for r in rows)))
print('Rate-limit states:', dict(Counter(r['limit'] for r in rows)))
print('Policy rejections:', sum(r['status'] == 429 and r['limit'] == 'REJECTED' for r in rows))
print('HTTP 5xx:', sum(500 <= r['status'] < 600 for r in rows))
EOF
```
