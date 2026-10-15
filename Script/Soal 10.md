### Soal 10

##### `tengen` : script startup

```bash
cat > /root/start_lab.sh <<'EOF'
#!/bin/sh
set -eu
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server start
EOF
chmod +x /root/start_lab.sh
update-rc.d isc-dhcp-server defaults
```

##### `jichi` : script startup

```bash
cat > /root/start_lab.sh <<'EOF'
#!/bin/sh
set -eu
/root/start_router.sh
service isc-dhcp-relay start
EOF
chmod +x /root/start_lab.sh
update-rc.d isc-dhcp-relay defaults
```

```bash
/bin/sh -c '/root/start_lab.sh; exec /bin/bash'
```
