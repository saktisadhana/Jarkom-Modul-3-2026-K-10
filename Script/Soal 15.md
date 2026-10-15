### Soal 15

##### `geto` : install nginx dan script penulis config

```bash
apt-get update
apt-get install -y nginx curl python3
mkdir -p /root/nginx
cat > /root/write_geto.sh <<'SCRIPT'
#!/bin/sh
set -eu
. /root/lab.env
stage=${1:-basic}
case "$stage" in
    basic|trusted|backup) ;;
    *) echo 'Usage: write_geto.sh basic|trusted|backup' >&2; exit 1 ;;
esac
TRUST_CONF=''
PEER_VAR='$remote_addr'
PRIMARY_FLAGS='max_fails=0'
BACKUP_LINE=''
if [ "$stage" != basic ]; then
    TRUST_CONF="set_real_ip_from ${GOJO};
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;"
    PEER_VAR='$realip_remote_addr'
fi
if [ "$stage" = backup ]; then
    PRIMARY_FLAGS='max_fails=2 fail_timeout=5s'
    BACKUP_LINE="server ${KIRARA}:80 backup;"
fi
cat > /root/nginx/geto.conf <<EOF
worker_processes 1;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;
events { worker_connections 1024; }
http {
    include /etc/nginx/mime.types;
    $TRUST_CONF
    geo $PEER_VAR \$allowed_api_peer {
        default 0;
        $GOJO 1;
        ${P}.4.0/24 1;
    }
    log_format api '\$remote_addr peer=$PEER_VAR '
        'status=\$status upstream=\$upstream_addr '
        'upstream_status=\$upstream_status uri=\$request_uri';
    access_log /var/log/nginx/geto_access.log api;
    upstream api_cluster {
        zone api_cluster 64k;
        server $NANAMI:80 weight=3 $PRIMARY_FLAGS;
        server $TODO:80 weight=1 $PRIMARY_FLAGS;
        $BACKUP_LINE
    }
    server {
        listen 80;
        server_name api.$ZONE $ZONE www.$ZONE;
        if (\$allowed_api_peer = 0) { return 403; }
        location / {
            proxy_pass http://api_cluster;
            proxy_set_header Host api.$ZONE;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_connect_timeout 1s;
            proxy_read_timeout 5s;
            proxy_next_upstream error timeout http_502 http_503 http_504;
            proxy_next_upstream_tries 3;
        }
    }
}
EOF
cp /root/nginx/geto.conf /etc/nginx/nginx.conf
nginx -t
service nginx restart
printf '%s\n' "$stage" > /root/geto-stage
SCRIPT
chmod +x /root/write_geto.sh
/root/write_geto.sh basic
```

##### `tengen` : 40 request berurutan dari Network 4

```bash
apt-get install -y curl python3
. /root/lab.env
: > /root/weighted-responses.jsonl
for n in $(seq 1 40); do
    curl -fsS -H "Host: api.$ZONE" "http://$GETO/api" \
        >> /root/weighted-responses.jsonl || break
done
python3 - <<'EOF'
import json
from collections import Counter
from pathlib import Path
rows = [json.loads(s) for s in Path('/root/weighted-responses.jsonl').read_text().splitlines()]
print('Successful responses:', len(rows))
print('Worker counts:', dict(Counter(r['worker'] for r in rows)))
EOF
```
