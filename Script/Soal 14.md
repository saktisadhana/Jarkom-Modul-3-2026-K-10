### Soal 14

##### Kelima worker : install nginx dan PHP-FPM

```bash
apt-get update
apt-get install -y nginx php-fpm php-mysql php-cli curl
mkdir -p /root/nginx /root/web /var/www/shibuya
PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
service "php${PHPV}-fpm" start
ls -l "/run/php/php${PHPV}-fpm.sock"
```

##### `maki` & `panda` : halaman frontend

```bash
cat > /root/web/index.php <<'EOF'
<?php
header('Content-Type: text/html; charset=utf-8');
// A fixed-width marker keeps ab's content-length check meaningful.
$name = htmlspecialchars(sprintf('%-16s', gethostname()), ENT_QUOTES, 'UTF-8');
echo "<!doctype html><html><body><pre>Frontend: {$name}</pre></body></html>\n";
EOF
install -m 0644 /root/web/index.php /var/www/shibuya/index.php
. /root/lab.env
PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
cat > /root/nginx/shibuya.conf <<EOF
server {
    listen 80;
    server_name $ZONE www.$ZONE;
    root /var/www/shibuya;
    index index.php;
    allow $GOJO;
    deny all;
    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }
    location ~ \.php\$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php${PHPV}-fpm.sock;
    }
}
EOF
rm -f /etc/nginx/sites-enabled/default
cp /root/nginx/shibuya.conf /etc/nginx/sites-available/shibuya
ln -sfn /etc/nginx/sites-available/shibuya /etc/nginx/sites-enabled/shibuya
php -l /var/www/shibuya/index.php
nginx -t && service nginx restart
```

##### `nanami`, `todo`, `kirara` : akses database

```bash
. /root/lab.env
APP_DB_PASS='PASTE_APPLICATION_PASSWORD_FROM_YAGA'
cat > /root/shibuya-db.ini <<EOF
db_host="$YAGA"
db_name="$DB_NAME"
db_user="shibuya_app"
db_password="$APP_DB_PASS"
trusted_proxy="$GETO"
EOF
install -o root -g www-data -m 0640 /root/shibuya-db.ini /etc/shibuya-db.ini
```

##### Ketiga API worker : script pencatat request

```bash
cat > /root/web/index.php <<'EOF'
<?php
header('Content-Type: application/json; charset=utf-8');
header('Cache-Control: no-store');
$cfg = parse_ini_file('/etc/shibuya-db.ini', false, INI_SCANNER_RAW);
$peer = $_SERVER['REMOTE_ADDR'] ?? '';
if (!$cfg || $peer !== $cfg['trusted_proxy']) {
    http_response_code(403);
    echo json_encode(['error' => 'Untrusted upstream']);
    exit;
}
// Geto overwrites this header after validating the client-IP chain.
$client = $_SERVER['HTTP_X_REAL_IP'] ?? '';
if (!filter_var($client, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)) {
    http_response_code(400);
    echo json_encode(['error' => 'Missing or invalid verified client IP']);
    exit;
}
try {
    $pdo = new PDO(
        'mysql:host='.$cfg['db_host'].';dbname='.$cfg['db_name'].';charset=utf8mb4',
        $cfg['db_user'], $cfg['db_password'],
        [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
         PDO::ATTR_TIMEOUT => 3]
    );
    $worker = gethostname();
    $uri = substr($_SERVER['REQUEST_URI'] ?? '/', 0, 2048);
    $stmt = $pdo->prepare(
        'INSERT INTO request_logs (worker,client_ip,request_uri) VALUES (?,?,?)'
    );
    $stmt->execute([$worker, $client, $uri]);
    echo json_encode([
        'worker' => $worker,
        'client_ip' => $client,
        'proxy_peer' => $peer,
        'log_id' => $pdo->lastInsertId()
    ])."\n";
} catch (Throwable $e) {
    error_log($e->getMessage());
    http_response_code(503);
    echo json_encode(['error' => 'Database logging unavailable'])."\n";
}
EOF
install -m 0644 /root/web/index.php /var/www/shibuya/index.php
php -l /var/www/shibuya/index.php
```

##### Ketiga API worker : virtual host dan pembatasan akses

```bash
. /root/lab.env
PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
cat > /root/nginx/shibuya.conf <<EOF
server {
    listen 80;
    server_name api.$ZONE $ZONE www.$ZONE;
    root /var/www/shibuya;
    index index.php;
    allow $GETO;
    deny all;
    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }
    location ~ \.php\$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php${PHPV}-fpm.sock;
    }
}
EOF
rm -f /etc/nginx/sites-enabled/default
cp /root/nginx/shibuya.conf /etc/nginx/sites-available/shibuya
ln -sfn /etc/nginx/sites-available/shibuya /etc/nginx/sites-enabled/shibuya
nginx -t && service nginx restart
```

##### Client : buktikan akses langsung ditolak

```bash
. /root/lab.env
for ip in "$MAKI" "$PANDA"; do
    curl -i -H "Host: $ZONE" "http://$ip/"
done
for ip in "$NANAMI" "$TODO" "$KIRARA"; do
    curl -i -H "Host: api.$ZONE" "http://$ip/api"
done
```

##### `geto` : uji akses API yang diizinkan

```bash
. /root/lab.env
curl -i -H "Host: api.$ZONE" -H "X-Real-IP: $GETO" "http://$NANAMI/api"
```

##### Semua worker : script startup

```bash
#!/bin/sh
set -eu
PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
service "php${PHPV}-fpm" start
nginx -t
service nginx start
```
