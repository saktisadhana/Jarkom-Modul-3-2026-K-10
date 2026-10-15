### Soal 16

##### `gojo` : install nginx dan script penulis config

```bash
apt-get update
apt-get install -y nginx curl python3
mkdir -p /root/nginx /etc/nginx/snippets
printf '# API limiting disabled for the routing baseline\n' \
    > /root/nginx/api-rate.conf
cp /root/nginx/api-rate.conf /etc/nginx/snippets/shibuya-rate.conf
cat > /root/write_gojo.sh <<'SCRIPT'
#!/bin/sh
set -eu
. /root/lab.env
stage=${1:-baseline}
case "$stage" in
    baseline|cached) ;;
    *) echo 'Usage: write_gojo.sh baseline|cached' >&2; exit 1 ;;
esac
CACHE_HTTP=''
CACHE_LOCATION=''
if [ "$stage" = cached ]; then
    mkdir -p /var/cache/nginx/shibuya
    chown -R www-data:www-data /var/cache/nginx/shibuya
    CACHE_HTTP='proxy_cache_path /var/cache/nginx/shibuya levels=1:2
        keys_zone=frontend_cache:10m max_size=100m inactive=20m
        use_temp_path=off;'
    CACHE_LOCATION='proxy_cache frontend_cache;
            proxy_cache_key "$scheme|$host|$request_uri";
            proxy_cache_valid 200 10m;
            proxy_cache_lock on;
            proxy_cache_bypass $http_authorization $http_cookie;
            proxy_no_cache $http_authorization $http_cookie;
            add_header X-Cache-Status $upstream_cache_status always;'
fi
cat > /root/nginx/api-proxy.conf <<EOF
include /etc/nginx/snippets/shibuya-rate.conf;
proxy_pass http://$GETO;
proxy_set_header Host api.$ZONE;
proxy_set_header X-Real-IP \$remote_addr;
# Gojo is the edge: replace any client-supplied address chain.
proxy_set_header X-Forwarded-For \$remote_addr;
proxy_cache off;
proxy_connect_timeout 2s;
proxy_read_timeout 10s;
EOF
cp /root/nginx/api-proxy.conf /etc/nginx/snippets/shibuya-api.conf
cat > /root/nginx/gojo.conf <<EOF
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;
events { worker_connections 2048; }
http {
    include /etc/nginx/mime.types;
    $CACHE_HTTP
    limit_req_zone \$binary_remote_addr zone=api_per_ip:10m rate=10r/s;
    log_format audit escape=json
        '{"client":"\$remote_addr","uri":"\$request_uri",'
        '"status":\$status,"request_time":"\$request_time",'
        '"upstream":"\$upstream_addr",'
        '"upstream_status":"\$upstream_status",'
        '"upstream_time":"\$upstream_response_time",'
        '"cache":"\$upstream_cache_status",'
        '"limit":"\$limit_req_status"}';
    access_log /var/log/nginx/gojo_access.jsonl audit;
    upstream frontend_cluster {
        zone frontend_cluster 64k;
        server $MAKI:80;
        server $PANDA:80;
    }
    server {
        listen 80;
        server_name $ZONE www.$ZONE;
        location = /api {
            include /etc/nginx/snippets/shibuya-api.conf;
        }
        location ^~ /api/ {
            include /etc/nginx/snippets/shibuya-api.conf;
        }
        location / {
            proxy_pass http://frontend_cluster;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$remote_addr;
            $CACHE_LOCATION
        }
    }
}
EOF
cp /root/nginx/gojo.conf /etc/nginx/nginx.conf
nginx -t
service nginx restart
printf '%s\n' "$stage" > /root/gojo-stage
SCRIPT
chmod +x /root/write_gojo.sh
/root/write_gojo.sh baseline
```

##### `itadori` atau `ino` : cek kedua jalur dan kedua nama

```bash
. /root/lab.env
for n in $(seq 1 8); do
    curl -fsS "http://$ZONE/"
done
for n in $(seq 1 8); do
    curl -fsS "http://$ZONE/api"
done
curl -i "http://www.$ZONE/"
curl -i "http://www.$ZONE/api"
```
