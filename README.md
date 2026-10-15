## Identitas Praktikan

| Nama                     | NRP        | Kelas |
| ------------------------ | ---------- | ----- |
| Putu Putra Sakti Sadhana | 5027251101 | B     |
| Danish Faeyza Rusmawan   | 5027251028 | B     |
## Reporting
### Persiapan
#### Rencana Alamat

Semua subnet internal memakai `/24` (`255.255.255.0`) dengan prefix `192.216`. Node infrastruktur, worker, dan database memakai `ardhptr21/debinet:latest`, sedangkan client memakai `ardhptr21/alpinet:latest`. Alamat bertanda **usulan** tidak tertulis di soal, jadi sesuaikan dengan tabel resmi kalau berbeda.
![](topologi%20-%20shibuya.png)

| Node | Network | Alamat | Sumber | Peran |
| --- | --- | --- | --- | --- |
| `jichi` | WAN | `eth0` lewat DHCP dari NAT | Soal | Router WAN |
| `jichi` | 1–4 | `eth1` 192.216.1.1; `eth2` 192.216.2.1; `eth3` 192.216.3.1; `eth4` 192.216.4.1 | Soal | Gateway LAN dan relay |
| `utahime` | 1 | 192.216.1.2 | Soal | DNS master |
| `shoko` | 1 | 192.216.1.3 | Soal | DNS slave |
| `gojo` | 1 | 192.216.1.4 | Usulan | Reverse proxy publik |
| `tengen` | 4 | 192.216.4.10 | Soal | DHCP server |
| `geto` | 4 | 192.216.4.11 | Usulan | Load balancer API internal |
| `maki` | 4 | 192.216.4.20 | Usulan | Worker frontend |
| `panda` | 4 | 192.216.4.21 | Usulan | Worker frontend |
| `nanami` | 4 | 192.216.4.22 | Usulan | Worker API utama, bobot 3 |
| `todo` | 4 | 192.216.4.23 | Usulan | Worker API utama, bobot 1 |
| `kirara` | 4 | 192.216.4.24 | Usulan | Worker API cadangan |
| `yaga` | 4 | 192.216.4.30 | Usulan | MariaDB master |
| `inumaki` | 4 | 192.216.4.31 | Usulan | MariaDB slave |
| `itadori`, `fushiguro` | 2 | 192.216.2.10–35 | Soal | Pool 1 |
| `zenin`, `okkotsu` | 2 | 192.216.2.70–95 | Soal | Pool 2 |
| `ino`, `kusakabe` | 3 | 192.216.3.10–89 | Soal | Lease 1800 detik |
| `miwa` | 3 | 192.216.3.100–199 | Soal | Lease 43200 detik |
| `kugisaki` | 3 | 192.216.3.99 | Soal | Reservasi MAC |

Setiap network terhubung ke switch sendiri dan ke interface `jichi` yang sesuai. Hanya `eth0` milik `jichi` yang terhubung ke NAT. Host infrastruktur memakai gateway `192.216.N.1` untuk network N.

#### Step by Step

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

Dalam step ini kami membuat `/root/lab.env` di setiap node, berisi semua variabel yang dipakai script di soal-soal berikutnya.

	P='192.216'
	GROUP_ID='K10'

Berfungsi untuk menentukan prefix IP dan nama kelompok. Dari dua nilai ini terbentuk zona DNS `jujutsu.k10.com`, search domain DHCP `k10.com`, dan nama database `db_shibuya_K10`. Sedangkan:

	case "$P" in
	    REPLACE*) echo 'Set P in /root/lab.env first' >&2; return 1 ;;

Berfungsi untuk menghentikan script kalau nilai `P` atau `GROUP_ID` belum diganti, atau kalau `GROUP_ID` berisi karakter selain huruf, angka, dan garis bawah.

Variabel IP seperti `UTAHIME`, `GOJO`, dan `YAGA` berisi alamat dari tabel di atas, dan `REV_ZONE` berisi nama reverse zone Network 1 (`1.216.192.in-addr.arpa`). Kalau ada alamat usulan yang diubah, ubah juga di semua node.

##### Semua node : resolver awal

```bash
printf 'nameserver 192.168.122.1\n' > /etc/resolv.conf
```

Dalam step ini resolver diarahkan ke NAT (`192.168.122.1`) supaya bisa menginstal paket.

Semua script dan file config sumber disimpan di `/root`, lalu disalin ke folder service yang dibutuhkan. Jalankan blok heredoc sampai penutupnya. `<<'EOF'` menulis variabel apa adanya, sedangkan `<<EOF` mengganti variabel shell, jadi variabel nginx di blok itu ditulis dengan `\$`.

Catatan: `lab.env` dan resolver harus dibuat di **semua 21 node** sebelum menjalankan script apa pun. Hampir semua kendala yang kami alami berawal dari `lab.env` yang belum dibuat, sehingga variabel seperti `$GETO` dan `$ZONE` kosong. Sebelum mulai di sebuah node, kami cek dulu:

```bash
ls /root/lab.env && . /root/lab.env && echo "$P $ZONE"
```

Command sebaiknya di-copy dari file teks mentah (`Script/*.md` dibuka di Notepad atau VS Code), bukan dari tampilan markdown yang sudah dirender, karena teks `www.` bisa berubah menjadi link markdown saat di-copy.

### Soal 1
#### Brief Overview

Mengaktifkan routing di `jichi` (WAN lewat DHCP, empat gateway LAN, IP forwarding, dan NAT), lalu memberi alamat statis untuk semua host infrastruktur.

#### Step by Step

##### `jichi` : konfigurasi interface permanen

```ini
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.216.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 192.216.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 192.216.3.1
    netmask 255.255.255.0

auto eth4
iface eth4 inet static
    address 192.216.4.1
    netmask 255.255.255.0
```

Dalam step ini kami mengisi config jaringan `jichi` lewat GNS3, lalu me-restart node.

	auto eth0
	iface eth0 inet dhcp

Berfungsi agar `eth0` meminta IP dari NAT lewat DHCP. Default route `jichi` juga didapat dari lease ini, jadi `jichi` tidak punya gateway di sisi LAN. Sedangkan:

	iface eth1 inet static
	    address 192.216.1.1

Berfungsi untuk menjadikan `jichi` gateway Network 1. `eth2`, `eth3`, dan `eth4` mengikuti pola yang sama untuk Network 2, 3, dan 4.

Kalau `eth0` belum dapat IP, jalankan `dhclient -v eth0` atau `udhcpc -i eth0`, tapi jangan dua-duanya di interface yang sama.

##### `jichi` : install paket dan buat script router

```bash
apt-get update
apt-get install -y iptables iproute2 iputils-ping curl tcpdump
cat > /root/start_router.sh <<'EOF'
#!/bin/sh
set -eu
. /root/lab.env
sysctl -w net.ipv4.ip_forward=1
for n in 1 2 3 4; do
    lan="eth${n}"
    iptables -C FORWARD -i "$lan" -j ACCEPT 2>/dev/null ||
        iptables -A FORWARD -i "$lan" -j ACCEPT
    iptables -t nat -C POSTROUTING -s "${P}.${n}.0/24" \
        -o eth0 -j MASQUERADE 2>/dev/null ||
        iptables -t nat -A POSTROUTING -s "${P}.${n}.0/24" \
        -o eth0 -j MASQUERADE
 done
iptables -C FORWARD -i eth0 -m conntrack \
    --ctstate ESTABLISHED,RELATED -j ACCEPT 2>/dev/null ||
    iptables -A FORWARD -i eth0 -m conntrack \
    --ctstate ESTABLISHED,RELATED -j ACCEPT
EOF
chmod +x /root/start_router.sh
/root/start_router.sh
printf 'net.ipv4.ip_forward=1\n' > /root/99-shibuya-forward.conf
cp /root/99-shibuya-forward.conf /etc/sysctl.d/
```

Dalam step ini kami menginstal tools jaringan, lalu membuat `/root/start_router.sh` supaya aturan routing bisa dijalankan ulang kapan saja.

	sysctl -w net.ipv4.ip_forward=1

Berfungsi untuk menyalakan IP forwarding, sama seperti di Modul 2.

	iptables -t nat -C POSTROUTING -s "${P}.${n}.0/24" \

Berfungsi untuk memasang MASQUERADE untuk setiap subnet LAN yang keluar lewat `eth0`. Pola `-C ... || -A ...` mencegah aturan ganda kalau script dijalankan ulang. NAT hanya berlaku untuk traffic ke `eth0`, jadi IP asli client tetap terlihat antar LAN.

	iptables -C FORWARD -i eth0 -m conntrack \

Berfungsi untuk mengizinkan paket balasan dari internet masuk kembali ke LAN.

File `/etc/sysctl.d/99-shibuya-forward.conf` membuat IP forwarding tetap aktif setelah restart. Kalau sudah ada aturan `DROP` sebelumnya, cek dulu urutan aturannya.

##### Semua host infrastruktur : alamat statis

```ini
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.216.4.10
    netmask 255.255.255.0
    gateway 192.216.4.1
```

Dalam step ini setiap host infrastruktur diberi alamat statis sesuai tabel. Contoh di atas untuk `tengen` (192.216.4.10, gateway 192.216.4.1). Ulangi untuk `utahime`, `shoko`, `gojo`, `geto`, semua worker, dan kedua database dengan alamat masing-masing. Simpan juga salinannya di `/root/network-interfaces.conf`, dan pastikan `hostname` sama dengan nama node.

##### Semua host infrastruktur : cek koneksi

```bash
ip -br addr
ip route
ip route get 1.1.1.1
# Replace this with the gateway for the current node.
ping -c 3 192.216.4.1
ping -c 3 1.1.1.1
ping -c 3 google.com
```

Dalam step ini kami mengecek alamat, route, dan koneksi setiap host.

	ip route get 1.1.1.1

Berfungsi untuk menampilkan jalur yang dipakai menuju internet, yaitu lewat gateway LAN masing-masing. Ganti `192.216.4.1` dengan gateway node yang sedang dicek.

##### `jichi` : cek forwarding dan NAT

```bash
sysctl net.ipv4.ip_forward
iptables -S FORWARD
iptables -t nat -S POSTROUTING
```

Dalam step ini kami memastikan IP forwarding bernilai `1`, serta aturan `FORWARD` dan `POSTROUTING` sudah terpasang.

### Soal 2
#### Brief Overview

Memasang ISC DHCP server di `tengen` (192.216.4.10), mendeklarasikan Network 4 tanpa pool, dan membuat scope untuk Network 2 dan 3.

#### Step by Step

##### `tengen` : install dan siapkan file konfigurasi

```bash
apt-get update
apt-get install -y isc-dhcp-server tcpdump
mkdir -p /root/dhcp /etc/dhcp/lab
. /root/lab.env
cat > /root/dhcp/dhcpd.conf <<EOF
authoritative;
ddns-update-style none;
default-lease-time 3600;
max-lease-time 7200;
option domain-name "$SEARCH_DOMAIN";
option domain-search "$SEARCH_DOMAIN";
option domain-name-servers $UTAHIME, $SHOKO;
include "/etc/dhcp/lab/classes.conf";
include "/etc/dhcp/lab/net2.conf";
include "/etc/dhcp/lab/net3.conf";
include "/etc/dhcp/lab/net4.conf";
include "/etc/dhcp/lab/hosts.conf";
EOF
: > /root/dhcp/classes.conf
: > /root/dhcp/hosts.conf
cat > /root/dhcp/net4.conf <<EOF
subnet ${P}.4.0 netmask 255.255.255.0 {
    option routers ${P}.4.1;
    option broadcast-address ${P}.4.255;
}
EOF
cat > /root/dhcp/net2.conf <<EOF
subnet ${P}.2.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option routers ${P}.2.1;
    option broadcast-address ${P}.2.255;
    range ${P}.2.10 ${P}.2.35;
}
EOF
cat > /root/dhcp/net3.conf <<EOF
subnet ${P}.3.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option routers ${P}.3.1;
    option broadcast-address ${P}.3.255;
    range ${P}.3.10 ${P}.3.89;
}
EOF
cat > /root/isc-dhcp-server.default <<'EOF'
INTERFACESv4="eth0"
INTERFACESv6=""
EOF
cp /root/isc-dhcp-server.default /etc/default/isc-dhcp-server
```

Dalam step ini kami menginstal `isc-dhcp-server` dan menulis config DHCP yang dipecah per file di `/root/dhcp/`.

	option domain-name-servers $UTAHIME, $SHOKO;

Berfungsi untuk membagikan `utahime` dan `shoko` sebagai DNS ke semua client. DNS ini baru bisa dipakai setelah Soal 11–12.

	include "/etc/dhcp/lab/net2.conf";

Berfungsi untuk memuat config tiap network dari file terpisah, supaya di soal berikutnya cukup mengganti satu file.

	subnet ${P}.4.0 netmask 255.255.255.0 {
	    option routers ${P}.4.1;

Berfungsi untuk mendeklarasikan Network 4 tanpa `range`. `tengen` butuh deklarasi subnet tempat ia berada supaya daemon bisa jalan, tapi tidak boleh membagikan IP ke host infrastruktur.

	range ${P}.2.10 ${P}.2.35;

Berfungsi sebagai pool sementara untuk uji relay di Soal 3. Soal 4–6 menggantinya dengan pool per class.

	INTERFACESv4="eth0"

Berfungsi agar DHCP server mendengarkan di `eth0`.

##### `tengen` : script deploy yang bisa dipakai ulang

```bash
cat > /root/apply_dhcp.sh <<'EOF'
#!/bin/sh
set -eu
mkdir -p /etc/dhcp/lab
cp /root/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf
for file in classes net2 net3 net4 hosts; do
    cp "/root/dhcp/${file}.conf" "/etc/dhcp/lab/${file}.conf"
done
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
service isc-dhcp-server status
EOF
chmod +x /root/apply_dhcp.sh
/root/apply_dhcp.sh
ss -lunp | grep ':67'
```

Dalam step ini kami membuat `/root/apply_dhcp.sh`, yang menyalin config ke `/etc/dhcp/`, mengecek config, lalu me-restart service.

	dhcpd -t -cf /etc/dhcp/dhcpd.conf

Berfungsi untuk mengecek syntax config sebelum service dijalankan. Sedangkan:

	ss -lunp | grep ':67'

Berfungsi untuk memastikan DHCP server mendengarkan di port UDP 67.

Kalau service gagal start saat `apt-get install`, itu normal karena config belum ada. Yang penting config akhir lolos `dhcpd -t` dan service berjalan tanpa pesan `No subnet declaration`. Kalau tetap gagal, cek apakah `tengen` benar memegang 192.216.4.10/24.

### Soal 3
#### Brief Overview

Meneruskan request DHCP dari `eth2` dan `eth3` di `jichi` ke `tengen` memakai DHCP relay.

#### Step by Step

##### `jichi` : konfigurasi isc-dhcp-relay

```bash
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-relay
. /root/lab.env
cat > /root/isc-dhcp-relay.default <<EOF
SERVERS="$TENGEN"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
EOF
cp /root/isc-dhcp-relay.default /etc/default/isc-dhcp-relay
service isc-dhcp-relay restart
service isc-dhcp-relay status
ps -ef | grep '[d]hcrelay'
```

Dalam step ini kami menginstal `isc-dhcp-relay` dan menulis `/etc/default/isc-dhcp-relay`.

	SERVERS="$TENGEN"

Berfungsi untuk menentukan tujuan relay, yaitu `tengen` (192.216.4.10). Sedangkan:

	INTERFACES="eth2 eth3 eth4"

Berfungsi untuk menentukan interface relay. `eth2` dan `eth3` menerima broadcast dari client, sedangkan `eth4` dipakai untuk berkomunikasi dengan `tengen`. `eth0` dan `eth1` tidak ikut.

`tengen` memilih subnet berdasarkan field `giaddr` yang diisi relay, bukan dari IP sumber paket.

##### `itadori` & `ino` : minta lease uji

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
ip link set eth0 up
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
ip -4 addr show eth0
ip route
cat /etc/resolv.conf
```

Dalam step ini kami meminta IP dari client Alpine di Network 2 (`itadori`) dan Network 3 (`ino`). Jalankan dari console GNS3, karena IP lama dihapus dulu dan koneksi sempat terputus.

	udhcpc -i eth0 -n -q -t 5 -T 2 -O 119

Berfungsi untuk meminta lease DHCP. `-n` keluar kalau gagal, `-q` keluar setelah dapat lease, `-t 5 -T 2` mencoba 5 kali dengan jeda 2 detik, dan `-O 119` meminta option domain search. Jangan menjalankan dua DHCP client di interface yang sama.

Hasil yang benar: `itadori` mendapat 192.216.2.10–35 dengan gateway 192.216.2.1, dan `ino` mendapat 192.216.3.10–89 dengan gateway 192.216.3.1. Untuk melihat paketnya di `tengen`, jalankan `tcpdump -ni eth0 -vvv 'udp port 67 or udp port 68'`.

### Soal 4
#### Brief Overview

Membagi client Network 2 ke dua pool: Pool 1 (192.216.2.10–35) untuk `itadori` dan `fushiguro`, dan Pool 2 (192.216.2.70–95) untuk `zenin` dan `okkotsu`. Rentang 192.216.2.36–69 tidak boleh dibagikan.

#### Step by Step

##### Semua client : catat MAC eth0

```bash
hostname
cat /sys/class/net/eth0/address
```

Dalam step ini kami mencatat MAC address `eth0` di kedelapan client. Jangan pakai MAC contoh atau MAC router.

##### `tengen` : simpan MAC client

```bash
cat > /root/client-macs.env <<'EOF'
ITADORI_MAC='REPLACE_ITADORI_MAC'
FUSHIGURO_MAC='REPLACE_FUSHIGURO_MAC'
ZENIN_MAC='REPLACE_ZENIN_MAC'
OKKOTSU_MAC='REPLACE_OKKOTSU_MAC'
INO_MAC='REPLACE_INO_MAC'
KUSAKABE_MAC='REPLACE_KUSAKABE_MAC'
MIWA_MAC='REPLACE_MIWA_MAC'
KUGISAKI_MAC='REPLACE_KUGISAKI_MAC'
EOF
```

Dalam step ini kami menyimpan MAC tiap client ke `/root/client-macs.env`. Ganti semua `REPLACE_...` dengan MAC asli dalam format titik dua.

##### `tengen` : class dan pool

```bash
. /root/lab.env
. /root/client-macs.env
cat > /root/dhcp/classes.conf <<EOF
class "division-a-pool1" {
    match hardware;
}
subclass "division-a-pool1" 1:$ITADORI_MAC;
subclass "division-a-pool1" 1:$FUSHIGURO_MAC;
class "division-a-pool2" {
    match hardware;
}
subclass "division-a-pool2" 1:$ZENIN_MAC;
subclass "division-a-pool2" 1:$OKKOTSU_MAC;
class "division-b-short" {
    match hardware;
}
subclass "division-b-short" 1:$INO_MAC;
subclass "division-b-short" 1:$KUSAKABE_MAC;
class "division-b-long" {
    match hardware;
}
subclass "division-b-long" 1:$MIWA_MAC;
EOF
cat > /root/dhcp/net2.conf <<EOF
subnet ${P}.2.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option routers ${P}.2.1;
    option broadcast-address ${P}.2.255;
    pool {
        allow members of "division-a-pool1";
        range ${P}.2.10 ${P}.2.35;
    }
    pool {
        allow members of "division-a-pool2";
        range ${P}.2.70 ${P}.2.95;
    }
}
EOF
/root/apply_dhcp.sh
```

Dalam step ini kami membuat class berdasarkan MAC, lalu membagi Network 2 menjadi dua pool.

	class "division-a-pool1" {
	    match hardware;
	}
	subclass "division-a-pool1" 1:$ITADORI_MAC;

Berfungsi untuk mengelompokkan client berdasarkan MAC. `1:` di depan MAC menandakan tipe hardware Ethernet. Class `division-b-short` dan `division-b-long` juga dibuat di sini untuk Soal 5.

	pool {
	    allow members of "division-a-pool1";
	    range ${P}.2.10 ${P}.2.35;
	}

Berfungsi agar pool ini hanya melayani anggota class `division-a-pool1`. Kalau hanya menulis dua `range`, client tidak akan dipisah ke pool yang benar.

##### `itadori`, `fushiguro`, `zenin`, `okkotsu` : perbarui lease

Ulangi perintah `udhcpc` dari Soal 3 di keempat client. Lease lama bisa ditolak karena tidak cocok lagi dengan aturan baru, jadi yang dicatat adalah ACK dan IP yang baru.

| Client | Rentang yang diizinkan | Gateway |
| --- | --- | --- |
| `itadori` | 192.216.2.10–35 | 192.216.2.1 |
| `fushiguro` | 192.216.2.10–35 | 192.216.2.1 |
| `zenin` | 192.216.2.70–95 | 192.216.2.1 |
| `okkotsu` | 192.216.2.70–95 | 192.216.2.1 |

### Soal 5
#### Brief Overview

Memberi lease 1800 detik dari 192.216.3.10–89 untuk `ino` dan `kusakabe`, serta lease 43200 detik dari 192.216.3.100–199 untuk `miwa`, lengkap dengan dua DNS dan search domain.

#### Step by Step

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

Dalam step ini kami menulis ulang scope Network 3 menjadi dua pool dengan lama lease berbeda.

	min-lease-time 1800;
	default-lease-time 1800;
	max-lease-time 1800;

Berfungsi untuk mengunci lama lease di 1800 detik. Karena nilai minimum, default, dan maksimum sama, lama lease tetap 1800 detik walaupun client meminta durasi lain. Pool `miwa` memakai pola yang sama dengan 43200 detik.

Option DNS dan search domain dari Soal 2 berlaku untuk kedua scope:

```text
option domain-name-servers 192.216.1.2, 192.216.1.3;
option domain-search "k10.com";
```

##### `ino`, `kusakabe`, `miwa` : perbarui dan cek

Jalankan perintah `udhcpc` dari Soal 3, lalu cek paket ACK di Soal 8: option 51 (lama lease), option 6 (DNS), dan option 119 (search domain). `ip addr` saja tidak cukup untuk membuktikan lama lease.

### Soal 6
#### Brief Overview

Memesan IP 192.216.3.99 untuk `kugisaki` berdasarkan MAC, di luar kedua pool dinamis.

#### Step by Step

##### `tengen` : host fixed-address

```bash
. /root/lab.env
. /root/client-macs.env
cat > /root/dhcp/hosts.conf <<EOF
host kugisaki {
    hardware ethernet $KUGISAKI_MAC;
    fixed-address ${P}.3.99;
}
EOF
/root/apply_dhcp.sh
```

Dalam step ini kami menambahkan deklarasi host untuk `kugisaki`.

	hardware ethernet $KUGISAKI_MAC;
	fixed-address ${P}.3.99;

Berfungsi agar client dengan MAC tersebut selalu mendapat 192.216.3.99.

##### `kugisaki` : perbarui dan cek

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
ip -4 addr show eth0
ip route
cat /etc/resolv.conf
```

Dalam step ini kami meminta lease ulang di `kugisaki`. Hasil yang benar: 192.216.3.99/24 dengan gateway 192.216.3.1, serta DNS dan search domain yang sama. Jalankan dua kali untuk membuktikan IP-nya tetap.

Reservasi fixed-address bisa tidak muncul di `dhcpd.leases`, jadi buktinya adalah deklarasi host dan paket ACK.

### Soal 7
#### Brief Overview

Mengecek konfigurasi semua client: alamat, mask, broadcast, gateway, DNS, search domain, kesesuaian pool, dan tidak ada IP yang bentrok.

#### Step by Step

##### Kedelapan client : catat nilai asli

```bash
hostname
cat /sys/class/net/eth0/address
ip -4 addr show eth0
ip route
cat /etc/resolv.conf
```

Dalam step ini kami mencatat hostname, MAC, alamat, route, dan resolver di setiap client. Nilai yang benar:

| Client | Prefix/mask | Broadcast | Gateway | Lease |
| --- | --- | --- | --- | --- |
| `itadori`, `fushiguro` | 192.216.2.10–35 /24 | 192.216.2.255 | 192.216.2.1 | 3600 detik (default) |
| `zenin`, `okkotsu` | 192.216.2.70–95 /24 | 192.216.2.255 | 192.216.2.1 | 3600 detik (default) |
| `ino`, `kusakabe` | 192.216.3.10–89 /24 | 192.216.3.255 | 192.216.3.1 | 1800 detik |
| `miwa` | 192.216.3.100–199 /24 | 192.216.3.255 | 192.216.3.1 | 43200 detik |
| `kugisaki` | 192.216.3.99 /24 | 192.216.3.255 | 192.216.3.1 | Fixed address, lease global |

Kedua nameserver harus 192.216.1.2 dan 192.216.1.3, dengan search `k10.com`. Kalau search domain tidak tertulis di `resolv.conf`, cek option 119 di paket ACK. Jangan menambahkannya manual lalu mengklaim itu dari DHCP.

##### `tengen` : cocokkan lease dan reservasi

```bash
cat /var/lib/dhcp/dhcpd.leases
cat /etc/dhcp/lab/hosts.conf
```

Dalam step ini kami membaca file lease dan deklarasi host.

	cat /var/lib/dhcp/dhcpd.leases

Berfungsi untuk melihat lease yang sudah diberikan. File ini berupa jurnal, jadi satu IP bisa punya beberapa catatan. Yang dipakai adalah catatan terakhir untuk IP tersebut. `kugisaki` dicek dari `hosts.conf`, karena reservasinya bisa tidak tercatat sebagai lease biasa.

Hasilnya kami rangkum dalam tabel per client (hostname, MAC, IP, gateway, broadcast, DNS, search domain, waktu kedaluwarsa). Tidak boleh ada dua client aktif dengan IP yang sama.

### Soal 8
#### Brief Overview

Menangkap pertukaran DHCP DORA (DISCOVER, OFFER, REQUEST, ACK), lengkap dengan IP server, IP yang ditawarkan, lama lease, dan `giaddr`.

#### Step by Step

##### `tengen` : mulai capture

```bash
mkdir -p /root/captures
tcpdump -ni eth0 -s 0 -w /root/captures/dora.pcap \
    'udp port 67 or udp port 68'
```

Dalam step ini kami menjalankan `tcpdump` sebelum client meminta IP.

	tcpdump -ni eth0 -s 0 -w /root/captures/dora.pcap \

Berfungsi untuk menyimpan semua paket DHCP (port 67 dan 68) di `eth0` ke file `dora.pcap`. `-s 0` menyimpan paket secara utuh.

##### Satu client Network 2 atau 3 : mulai DORA baru

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
```

Dalam step ini client meminta IP dari awal, tanpa IP lama. Setelah client dapat IP, hentikan `tcpdump` dengan Ctrl+C. Kalau yang tertangkap hanya REQUEST dan ACK, ulangi supaya keempat pesan muncul.

##### `tengen` : baca hasil capture

```bash
tcpdump -nn -vvv -r /root/captures/dora.pcap
```

Dalam step ini kami membaca isi capture dengan detail penuh (`-vvv`). Yang dicek:

| Field | Arti |
| --- | --- |
| DHCP message type (53) | DISCOVER, OFFER, REQUEST, lalu ACK |
| Transaction ID / MAC client | Menghubungkan keempat pesan untuk client yang sama |
| Server identifier (54) | `tengen` 192.216.4.10 |
| Your-IP / yiaddr | IP yang diberikan dari pool client tersebut |
| Lease time (51) | Lama lease sesuai config client |
| Gateway-IP / giaddr | 192.216.2.1 untuk Network 2, atau 192.216.3.1 untuk Network 3 |
| Routers (3) | Gateway subnet client |
| DNS (6) dan search (119) | 192.216.1.2, 192.216.1.3, dan `k10.com` |

Field `Gateway-IP` di output tcpdump adalah `giaddr`, berbeda dengan option router (option 3). Di sisi `tengen`, traffic UDP 67 ke 67 adalah normal karena paket datang dari relay.

### Soal 9
#### Brief Overview

Mematikan relay, membuktikan client gagal mendapat IP, lalu menyalakan relay lagi dan membuktikan client pulih.

#### Step by Step

##### `jichi` : hentikan relay

```bash
service isc-dhcp-relay stop
pgrep -a dhcrelay || true
```

Dalam step ini kami menghentikan relay. `pgrep -a dhcrelay` harus kosong.

##### `itadori` atau `ino` : minta IP tanpa IP lama

```bash
killall udhcpc 2>/dev/null || true
ip -4 addr flush dev eth0 scope global
udhcpc -i eth0 -n -q -t 3 -T 2 -O 119
printf 'DHCP command exit status: %s\n' "$?"
ip -4 addr show eth0
ip route
```

Dalam step ini IP lama dihapus dulu, lalu client meminta IP baru.

	printf 'DHCP command exit status: %s\n' "$?"

Berfungsi untuk menampilkan exit status `udhcpc`. Hasil yang benar: client mencoba berulang kali, tidak dapat IP, dan exit status bukan 0. IP lama sengaja dihapus, karena client yang masih memegang lease lama bisa terlihat tetap berjalan.

##### `jichi` : nyalakan relay

```bash
service isc-dhcp-relay start
service isc-dhcp-relay status
```

Dalam step ini relay dinyalakan kembali dan dicek statusnya.

##### Client yang sama : minta IP lagi

```bash
udhcpc -i eth0 -n -q -t 5 -T 2 -O 119
ip -4 addr show eth0
ip route
```

Dalam step ini client meminta IP lagi. Hasil yang benar: IP dari pool yang sama dengan gateway subnet yang sama. IP-nya tidak harus sama dengan lease sebelumnya.

### Soal 10
#### Brief Overview

Membuat DHCP server di `tengen` serta routing dan relay di `jichi` tetap berjalan setelah restart.

#### Step by Step

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

Dalam step ini kami membuat `/root/start_lab.sh` yang mengecek config DHCP lalu menyalakan service.

	update-rc.d isc-dhcp-server defaults

Berfungsi untuk mendaftarkan service agar start otomatis. Di container Docker, ini saja belum cukup, jadi script juga harus dipanggil dari boot hook.

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

Dalam step ini `start_lab.sh` di `jichi` menjalankan `start_router.sh` dari Soal 1, lalu menyalakan relay.

Node ini adalah container Docker, jadi `update-rc.d` saja tidak menjamin service jalan saat node menyala. Cek boot hook image, misalnya `/root/init.sh`. Kalau itu boot hook-nya, sisipkan `/root/start_lab.sh` sebelum baris `exec` atau `exit` terakhir, tanpa menghapus inisialisasi interface yang sudah ada. Kalau start command node bisa diatur sendiri, bisa memakai:

```bash
/bin/sh -c '/root/start_lab.sh; exec /bin/bash'
```

Atur ini lewat startup setting node di GNS3, bukan hanya diketik sekali di console. Interface client dibiarkan memakai DHCP supaya lease diperbarui setelah restart.

##### Restart dan uji

1. Stop lalu start `tengen` dan `jichi` di GNS3.
2. Di `tengen`, cek `service isc-dhcp-server status` dan `ip -br addr`.
3. Di `jichi`, cek `service isc-dhcp-relay status`, forwarding, dan NAT dari Soal 1.
4. Perbarui lease kedelapan client dan ulangi tabel Soal 7.
5. Pastikan kedua pool Network 2, lease pendek/panjang, dan IP tetap `kugisaki` (192.216.3.99) masih benar.

### Soal 11
#### Brief Overview

Melayani zona `jujutsu.k10.com` dari `utahime` (master) dan `shoko` (slave), membatasi zone transfer, dan membuktikan sinkronisasi otomatis setelah serial dinaikkan.

#### Step by Step

##### `utahime` & `shoko` : install BIND

```bash
apt-get update
apt-get install -y bind9 bind9-utils dnsutils
mkdir -p /root/bind
```

Sama seperti Modul 2, blok ini dijalankan di kedua server.

##### `utahime` & `shoko` : recursion untuk client internal

```bash
. /root/lab.env
cat > /root/bind/named.conf.options <<EOF
acl lab_clients {
    127.0.0.1;
    ${P}.1.0/24;
    ${P}.2.0/24;
    ${P}.3.0/24;
    ${P}.4.0/24;
};
options {
    directory "/var/cache/bind";
    listen-on { any; };
    listen-on-v6 { none; };
    allow-query { lab_clients; };
    recursion yes;
    allow-recursion { lab_clients; };
    forwarders { 192.168.122.1; };
    forward only;
    dnssec-validation auto;
};
EOF
cp /root/bind/named.conf.options /etc/bind/named.conf.options
```

Dalam step ini kami menulis `named.conf.options` di kedua server.

	acl lab_clients {

Berfungsi untuk mendefinisikan daftar jaringan yang boleh bertanya ke DNS ini, yaitu localhost dan keempat network 192.216.1.0/24 sampai 192.216.4.0/24.

	allow-query { lab_clients; };
	allow-recursion { lab_clients; };

Berfungsi untuk membatasi query dan recursion hanya untuk jaringan lab. Sedangkan:

	forwarders { 192.168.122.1; };
	forward only;

Berfungsi untuk meneruskan nama di luar zona ke NAT. Forwarder dipasang di kedua server supaya client tetap bisa resolve nama luar walaupun memakai `shoko`.

Kalau query ke domain luar dijawab `SERVFAIL`, ganti `dnssec-validation auto;` menjadi `dnssec-validation no;` seperti di Modul 2.

##### `utahime` : zona primary dan record SOA/NS
![](utahime%20-%20zona%20primary.png)

```bash
. /root/lab.env
cat > /root/bind/named.conf.local <<EOF
zone "$ZONE" {
    type primary;
    file "/etc/bind/db.$ZONE";
    notify yes;
    also-notify { $SHOKO; };
    allow-transfer { $SHOKO; };
};
EOF
cat > "/root/bind/db.$ZONE" <<EOF
\$TTL 300
@ IN SOA utahime.$ZONE. admin.$ZONE. (
    2026101501 300 120 604800 300 )
@       IN NS utahime.$ZONE.
@       IN NS shoko.$ZONE.
utahime IN A $UTAHIME
shoko   IN A $SHOKO
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
cp "/root/bind/db.$ZONE" "/etc/bind/db.$ZONE"
named-checkconf
named-checkzone "$ZONE" "/etc/bind/db.$ZONE"
service named restart
service named status
```

Dalam step ini kami mendaftarkan zona `jujutsu.k10.com` sebagai primary dan menulis isi zona awal.

	notify yes;
	also-notify { $SHOKO; };
	allow-transfer { $SHOKO; };

Berfungsi agar `utahime` memberi tahu `shoko` setiap ada perubahan, dan hanya `shoko` yang boleh menyalin zona.

	@ IN SOA utahime.$ZONE. admin.$ZONE. (
	    2026101501 300 120 604800 300 )

Berfungsi sebagai identitas zona dengan serial `2026101501`. TTL, refresh, dan retry dibuat pendek (300 dan 120 detik) supaya perubahan cepat terlihat saat pengujian.

##### `shoko` : zona secondary
![](shoko%20-%20zona%20secondary.png)
```bash
. /root/lab.env
cat > /root/bind/named.conf.local <<EOF
zone "$ZONE" {
    type secondary;
    primaries { $UTAHIME; };
    file "/var/cache/bind/db.$ZONE";
    allow-transfer { none; };
};
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
named-checkconf
service named restart
service named status
```

Dalam step ini `shoko` didaftarkan sebagai secondary dengan sumber `utahime`. Isi zonanya tidak ditulis manual: BIND mengambilnya lewat TCP 53 dan menyimpannya di `/var/cache/bind/`. `allow-transfer { none; }` mencegah pihak lain menyalin zona dari `shoko`. Kalau service-nya bernama `bind9`, pakai nama itu secara konsisten.

##### Satu client Alpine : install dig dan bandingkan
![](itadori%20-%20dig%20soa%20utahime.png)
![](itadori%20-%20dig%20soa%20ns%20shoko.png)

```bash
apk update
apk add bind-tools curl
. /root/lab.env
dig @"$UTAHIME" "$ZONE" SOA +norecurse
dig @"$SHOKO" "$ZONE" SOA +norecurse
dig @"$UTAHIME" "$ZONE" NS +short
dig @"$SHOKO" "$ZONE" NS +short
```

Dalam step ini kami menginstal `dig` di client, lalu menanyakan SOA dan NS ke kedua server. Saat menginstal paket, resolver masih `192.168.122.1`. Setelah itu, perbarui DHCP supaya DNS kembali ke `utahime` dan `shoko`.

Kedua jawaban SOA harus punya flag `aa` dan serial yang sama.

##### `utahime` : uji transfer otomatis
![](utahime%20-%20uji%20transfer%20otomatis.png)
```bash
. /root/lab.env
# Run this change once; use a higher serial if already advanced.
sed -i 's/2026101501/2026101502/' "/root/bind/db.$ZONE"
printf 'sync-test IN A %s\n' "$GOJO" >> "/root/bind/db.$ZONE"
cp "/root/bind/db.$ZONE" "/etc/bind/db.$ZONE"
named-checkzone "$ZONE" "/etc/bind/db.$ZONE" &&
    rndc reload "$ZONE"
```

Dalam step ini kami menaikkan serial ke `2026101502` dan menambahkan record `sync-test`.

	rndc reload "$ZONE"

Berfungsi untuk memuat ulang zona tanpa me-restart `named`. Setelah itu, tanyakan `sync-test.jujutsu.k10.com` ke kedua server dari client. Record baru harus muncul di `shoko` tanpa disalin manual.

Jalankan step ini sekali saja. Kalau serial sudah lebih tinggi, pakai angka yang lebih besar.

#### Kendala
##### `named` mati setelah node dinyalakan ulang

Setelah `utahime` dan `shoko` sempat mati, `named` tidak menyala sendiri, sehingga client gagal resolve nama (`Could not contact DNS servers`). Container Docker di GNS3 tidak menjalankan service secara otomatis, jadi kami menyalakannya manual di kedua server:

```bash
service named status || service named start
```

Sama seperti di Modul 2, pesan `rndc: connect failed: 127.0.0.1#953: connection refused` saat restart pertama hanya berarti `named` belum pernah jalan sebelumnya.

### Soal 12
#### Brief Overview

Mengarahkan `jujutsu.k10.com` ke `gojo`, membuat `www` sebagai CNAME, membuat PTR untuk `gojo`, dan membuktikan forwarding ke internet.

#### Step by Step

##### `utahime` : update zona forward dan reverse
![](utahime%20-%20zona%20forward%20dan%20reverse.png)
```bash
. /root/lab.env
cat > "/root/bind/db.$ZONE" <<EOF
\$TTL 300
@ IN SOA utahime.$ZONE. admin.$ZONE. (
    2026101503 300 120 604800 300 )
@         IN NS utahime.$ZONE.
@         IN NS shoko.$ZONE.
@         IN A $GOJO
www       IN CNAME $ZONE.
utahime   IN A $UTAHIME
shoko     IN A $SHOKO
gojo      IN A $GOJO
sync-test IN A $GOJO
api       IN A $GETO
maki      IN A $MAKI
panda     IN A $PANDA
nanami    IN A $NANAMI
todo      IN A $TODO
kirara    IN A $KIRARA
EOF
cat > "/root/bind/db.reverse-net1" <<EOF
\$TTL 300
@ IN SOA utahime.$ZONE. admin.$ZONE. (
    2026101503 300 120 604800 300 )
@ IN NS utahime.$ZONE.
@ IN NS shoko.$ZONE.
$GOJO_PTR IN PTR $ZONE.
EOF
cat > /root/bind/named.conf.local <<EOF
zone "$ZONE" {
    type primary;
    file "/etc/bind/db.$ZONE";
    notify yes;
    also-notify { $SHOKO; };
    allow-transfer { $SHOKO; };
};
zone "$REV_ZONE" {
    type primary;
    file "/etc/bind/db.reverse-net1";
    notify yes;
    also-notify { $SHOKO; };
    allow-transfer { $SHOKO; };
};
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
cp "/root/bind/db.$ZONE" "/etc/bind/db.$ZONE"
cp /root/bind/db.reverse-net1 /etc/bind/db.reverse-net1
named-checkconf
named-checkzone "$ZONE" "/etc/bind/db.$ZONE"
named-checkzone "$REV_ZONE" /etc/bind/db.reverse-net1
service named reload
```

Dalam step ini kami menulis ulang zona forward, membuat reverse zone Network 1, lalu mendaftarkan keduanya.

	@         IN A $GOJO
	www       IN CNAME $ZONE.

Berfungsi agar `jujutsu.k10.com` mengarah ke `gojo`, dan `www.jujutsu.k10.com` menjadi alias dari nama tersebut. Record `api`, `maki`, `panda`, dan seterusnya ditambahkan untuk soal-soal berikutnya. Sedangkan:

	$GOJO_PTR IN PTR $ZONE.

Berfungsi untuk menerjemahkan IP `gojo` kembali ke `jujutsu.k10.com`. Label PTR cukup oktet terakhir IP `gojo` (`4`), dan nama reverse zone-nya `1.216.192.in-addr.arpa`.

##### `shoko` : reverse zone secondary
![](shoko%20-%20reverse%20zone%20secondary.png)
```bash
. /root/lab.env
cat >> /root/bind/named.conf.local <<EOF
zone "$REV_ZONE" {
    type secondary;
    primaries { $UTAHIME; };
    file "/var/cache/bind/db.reverse-net1";
    allow-transfer { none; };
};
EOF
cp /root/bind/named.conf.local /etc/bind/named.conf.local
named-checkconf && service named reload
```

Dalam step ini reverse zone ditambahkan ke `shoko` sebagai secondary. Blok ini cukup dijalankan sekali, karena memakai `>>`.

##### Client : cek kedua server dan forwarding
![](itadori%20-%20cek%20dns%20dan%20forwarding.png)
```bash
. /root/lab.env
for dns in "$UTAHIME" "$SHOKO"; do
    dig @"$dns" "$ZONE" A +short
    dig @"$dns" "www.$ZONE" CNAME +short
    dig @"$dns" -x "$GOJO" +short
    dig @"$dns" google.com A +short
done
cat /etc/resolv.conf
ping -c 3 google.com
```

Dalam step ini kami menanyakan A, CNAME, PTR, dan `google.com` ke kedua server. Hasil yang benar: apex mengarah ke `gojo`, `www` ke apex, IP `gojo` ke apex, dan `google.com` berhasil di-resolve lewat forwarder.

Kalau seluruh lab harus bertahan setelah restart, tambahkan `service named start` ke boot hook kedua server dengan cara dari Soal 10.

### Soal 13
#### Brief Overview

Membuat database log request di `yaga` (master) dengan binary log aktif, mengisi `inumaki` (slave) dari snapshot yang konsisten, dan menyambungkan replikasi dengan `CHANGE MASTER TO`.

#### Step by Step

##### `yaga` & `inumaki` : install MariaDB

```bash
apt-get update
apt-get install -y mariadb-server mariadb-client openssl
mkdir -p /root/mariadb
```

Blok ini dijalankan di kedua node.

##### `yaga` : konfigurasi master
![](yaga%20-%20konfigurasi%20master.png)
```bash
cat > /root/mariadb/60-shibuya.cnf <<'EOF'
[mariadb]
server-id=1
bind-address=0.0.0.0
log-bin=mariadb-bin
binlog-format=ROW
EOF
cp /root/mariadb/60-shibuya.cnf /etc/mysql/mariadb.conf.d/
service mariadb restart
mariadb -e "SHOW VARIABLES WHERE Variable_name IN ('server_id','log_bin','bind_address');"
```

Dalam step ini kami membuat config master.

	server-id=1
	log-bin=mariadb-bin
	binlog-format=ROW

Berfungsi untuk memberi ID unik pada server dan menyalakan binary log, yaitu catatan semua perubahan data yang nanti dibaca oleh slave. `bind-address=0.0.0.0` membuat MariaDB bisa diakses dari node lain.

##### `inumaki` : konfigurasi slave
![](inumaki%20-%20konfigurasi%20slave.png)
```bash
cat > /root/mariadb/60-shibuya.cnf <<'EOF'
[mariadb]
server-id=2
bind-address=0.0.0.0
relay-log=inumaki-relay-bin
read-only=ON
EOF
cp /root/mariadb/60-shibuya.cnf /etc/mysql/mariadb.conf.d/
service mariadb restart
```

Dalam step ini `inumaki` diberi `server-id=2` dan `read-only=ON`, supaya data di slave hanya berubah lewat replikasi.

##### `yaga` : buat kredensial, database, dan tabel
![](yaga%20-%20buat%20database%20dan%20user.png)
```bash
umask 077
if [ ! -f /root/db-secrets.env ]; then
    APP_DB_PASS=$(openssl rand -hex 20)
    REPL_PASS=$(openssl rand -hex 20)
    printf "APP_DB_PASS='%s'\nREPL_PASS='%s'\n" \
        "$APP_DB_PASS" "$REPL_PASS" > /root/db-secrets.env
fi
. /root/lab.env
. /root/db-secrets.env
cat > /root/mariadb/create_shibuya.sql <<EOF
CREATE DATABASE IF NOT EXISTS ${DB_NAME};
CREATE TABLE IF NOT EXISTS ${DB_NAME}.request_logs (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    worker VARCHAR(32) NOT NULL,
    client_ip VARCHAR(45) NOT NULL,
    request_uri VARCHAR(2048) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
CREATE USER IF NOT EXISTS 'replicator'@'${INUMAKI}'
    IDENTIFIED BY '${REPL_PASS}';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'${INUMAKI}';
CREATE USER IF NOT EXISTS 'shibuya_app'@'${P}.4.%'
    IDENTIFIED BY '${APP_DB_PASS}';
GRANT SELECT, INSERT ON ${DB_NAME}.* TO 'shibuya_app'@'${P}.4.%';
EOF
mariadb < /root/mariadb/create_shibuya.sql
mariadb -e "SHOW MASTER STATUS; SHOW CREATE TABLE ${DB_NAME}.request_logs\G"
```

Dalam step ini kami membuat password acak sekali saja, lalu membuat database, tabel, dan dua user.

	APP_DB_PASS=$(openssl rand -hex 20)

Berfungsi untuk membuat password acak. Password disimpan di `/root/db-secrets.env` dengan `umask 077`, jadi hanya root yang bisa membacanya.

	GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'${INUMAKI}';

Berfungsi untuk memberi hak replikasi khusus untuk `inumaki`. Sedangkan:

	GRANT SELECT, INSERT ON ${DB_NAME}.* TO 'shibuya_app'@'${P}.4.%';

Berfungsi untuk memberi API worker di Network 4 hak membaca dan menambah data saja.

Replikasi ini asinkron: data biasanya cepat sampai, tapi ini bukan failover database otomatis. Jangan tampilkan password di screenshot laporan.

##### `yaga` : buat snapshot dan koordinat binlog
![](yaga%20-%20snapshot%20dan%20binlog.png)
```bash
. /root/lab.env
mariadb-dump --single-transaction --master-data=2 \
    --databases "$DB_NAME" > /root/mariadb/shibuya-seed.sql
sed -n '/^-- CHANGE MASTER TO/p' /root/mariadb/shibuya-seed.sql
sha256sum /root/mariadb/shibuya-seed.sql
base64 /root/mariadb/shibuya-seed.sql
```

Dalam step ini kami membuat dump database yang konsisten.

	mariadb-dump --single-transaction --master-data=2 \

Berfungsi untuk membuat dump tanpa mengunci tabel, sekaligus mencatat posisi binlog saat dump dibuat sebagai baris komentar `CHANGE MASTER TO`. `sha256sum` dipakai untuk memastikan file tidak berubah saat dipindah, dan `base64` mengubahnya menjadi teks supaya bisa disalin ke `inumaki`.

##### `inumaki` : import snapshot
![](inumaki%20-%20import%20snapshot.png)
```bash
cat > /root/mariadb/shibuya-seed.b64 <<'EOF'
LypNITk5OTk5OVwtIGVuYWJsZSB0aGUgc2FuZGJveCBtb2RlICovIAotLSBNYXJpYURCIGR1bXAg
MTAuMTktMTEuOC42LU1hcmlhREIsIGZvciBkZWJpYW4tbGludXgtZ251ICh4ODZfNjQpCi0tCi0t
IEhvc3Q6IGxvY2FsaG9zdCAgICBEYXRhYmFzZTogZGJfc2hpYnV5YV9LMTAKLS0gLS0tLS0tLS0t
LS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0tCi0tIFNlcnZlciB2
ZXJzaW9uCTExLjguNi1NYXJpYURCLTArZGViMTN1MSBmcm9tIERlYmlhbi1sb2cKCi8qITQwMTAx
IFNFVCBAT0xEX0NIQVJBQ1RFUl9TRVRfQ0xJRU5UPUBAQ0hBUkFDVEVSX1NFVF9DTElFTlQgKi87
Ci8qITQwMTAxIFNFVCBAT0xEX0NIQVJBQ1RFUl9TRVRfUkVTVUxUUz1AQENIQVJBQ1RFUl9TRVRf
UkVTVUxUUyAqLzsKLyohNDAxMDEgU0VUIEBPTERfQ09MTEFUSU9OX0NPTk5FQ1RJT049QEBDT0xM
QVRJT05fQ09OTkVDVElPTiAqLzsKLyohNDAxMDEgU0VUIE5BTUVTIHV0ZjhtYjQgKi87Ci8qITQw
MTAzIFNFVCBAT0xEX1RJTUVfWk9ORT1AQFRJTUVfWk9ORSAqLzsKLyohNDAxMDMgU0VUIFRJTUVf
Wk9ORT0nKzAwOjAwJyAqLzsKLyohNDAwMTQgU0VUIEBPTERfVU5JUVVFX0NIRUNLUz1AQFVOSVFV
RV9DSEVDS1MsIFVOSVFVRV9DSEVDS1M9MCAqLzsKLyohNDAwMTQgU0VUIEBPTERfRk9SRUlHTl9L
RVlfQ0hFQ0tTPUBARk9SRUlHTl9LRVlfQ0hFQ0tTLCBGT1JFSUdOX0tFWV9DSEVDS1M9MCAqLzsK
LyohNDAxMDEgU0VUIEBPTERfU1FMX01PREU9QEBTUUxfTU9ERSwgU1FMX01PREU9J05PX0FVVE9f
VkFMVUVfT05fWkVSTycgKi87Ci8qTSExMDA2MTYgU0VUIEBPTERfTk9URV9WRVJCT1NJVFk9QEBO
T1RFX1ZFUkJPU0lUWSwgTk9URV9WRVJCT1NJVFk9MCAqLzsKCi0tCi0tIEFsdGVybmF0ZWx5LCBm
b2xsb3dpbmcgaXMgdGhlIHBvc2l0aW9uIG9mIHRoZSBiaW5hcnkgbG9nZ2luZyBmcm9tIFNIT1cg
TUFTVEVSIFNUQVRVUyBhdCBwb2ludCBvZiBiYWNrdXAuCi0tIFVzZSB0aGlzIHdoZW4gY3JlYXRp
bmcgYSByZXBsaWNhIG9mIHRoZSBwcmltYXJ5IHNlcnZlciB3aGVyZSB0aGUgYmFja3VwIHdhcyBt
YWRlLgotLSBUaGUgbmV3IHNlcnZlciB3aWxsIGJlIGNvbm5lY3RpbmcgdG8gdGhlIHByaW1hcnkg
c2VydmVyIHdoZXJlIHRoZSBiYWNrdXAgd2FzIHRha2VuLgotLQoKLS0gQ0hBTkdFIE1BU1RFUiBU
TyBNQVNURVJfTE9HX0ZJTEU9J21hcmlhZGItYmluLjAwMDAwMScsIE1BU1RFUl9MT0dfUE9TPTE3
MjA7CgotLSBBIGNvcnJlc3BvbmRpbmcgdG8gdGhlIGFib3ZlIG1hc3Rlci1kYXRhIENIQU5HRS1N
QVNURVIgc2V0dGluZ3MgdG8gdGhlIHNsYXZlIGd0aWQgc3RhdGUgaXMgcHJpbnRlZCBhcyBjb21t
ZW50cyBsYXRlciBpbiB0aGUgZmlsZS4KCi0tCi0tIEN1cnJlbnQgRGF0YWJhc2U6IGBkYl9zaGli
dXlhX0sxMGAKLS0KCkNSRUFURSBEQVRBQkFTRSAvKiEzMjMxMiBJRiBOT1QgRVhJU1RTKi8gYGRi
X3NoaWJ1eWFfSzEwYCAvKiE0MDEwMCBERUZBVUxUIENIQVJBQ1RFUiBTRVQgdXRmOG1iNCBDT0xM
QVRFIHV0ZjhtYjRfdWNhMTQwMF9haV9jaSAqLzsKClVTRSBgZGJfc2hpYnV5YV9LMTBgOwoKLS0K
LS0gVGFibGUgc3RydWN0dXJlIGZvciB0YWJsZSBgcmVxdWVzdF9sb2dzYAotLQoKRFJPUCBUQUJM
RSBJRiBFWElTVFMgYHJlcXVlc3RfbG9nc2A7Ci8qITQwMTAxIFNFVCBAc2F2ZWRfY3NfY2xpZW50
ICAgICA9IEBAY2hhcmFjdGVyX3NldF9jbGllbnQgKi87Ci8qITQwMTAxIFNFVCBjaGFyYWN0ZXJf
c2V0X2NsaWVudCA9IHV0ZjhtYjQgKi87CkNSRUFURSBUQUJMRSBgcmVxdWVzdF9sb2dzYCAoCiAg
YGlkYCBiaWdpbnQoMjApIHVuc2lnbmVkIE5PVCBOVUxMIEFVVE9fSU5DUkVNRU5ULAogIGB3b3Jr
ZXJgIHZhcmNoYXIoMzIpIE5PVCBOVUxMLAogIGBjbGllbnRfaXBgIHZhcmNoYXIoNDUpIE5PVCBO
VUxMLAogIGByZXF1ZXN0X3VyaWAgdmFyY2hhcigyMDQ4KSBOT1QgTlVMTCwKICBgY3JlYXRlZF9h
dGAgdGltZXN0YW1wIE5PVCBOVUxMIERFRkFVTFQgY3VycmVudF90aW1lc3RhbXAoKSwKICBQUklN
QVJZIEtFWSAoYGlkYCkKKSBFTkdJTkU9SW5ub0RCIERFRkFVTFQgQ0hBUlNFVD11dGY4bWI0IENP
TExBVEU9dXRmOG1iNF91Y2ExNDAwX2FpX2NpOwovKiE0MDEwMSBTRVQgY2hhcmFjdGVyX3NldF9j
bGllbnQgPSBAc2F2ZWRfY3NfY2xpZW50ICovOwoKLS0KLS0gRHVtcGluZyBkYXRhIGZvciB0YWJs
ZSBgcmVxdWVzdF9sb2dzYAotLQoKU0VUIEBPTERfQVVUT0NPTU1JVD1AQEFVVE9DT01NSVQsIEBA
QVVUT0NPTU1JVD0wOwpMT0NLIFRBQkxFUyBgcmVxdWVzdF9sb2dzYCBXUklURTsKLyohNDAwMDAg
QUxURVIgVEFCTEUgYHJlcXVlc3RfbG9nc2AgRElTQUJMRSBLRVlTICovOwovKiE0MDAwMCBBTFRF
UiBUQUJMRSBgcmVxdWVzdF9sb2dzYCBFTkFCTEUgS0VZUyAqLzsKVU5MT0NLIFRBQkxFUzsKQ09N
TUlUOwpTRVQgQVVUT0NPTU1JVD1AT0xEX0FVVE9DT01NSVQ7CgotLSBUaGUgZGVmZXJyZWQgZ3Rp
ZCBzZXR0aW5nIGZvciBzbGF2ZSBjb3JyZXNwb25kaW5nIHRvIHRoZSBtYXN0ZXItZGF0YSBDSEFO
R0UtTUFTVEVSIGZvbGxvd3MKCi0tIFByZWZlcmFibHkgdXNlIEdUSUQgdG8gc3RhcnQgcmVwbGlj
YXRpb24gZnJvbSBHVElEIHBvc2l0aW9uOgoKLS0gU0VUIEdMT0JBTCBndGlkX3NsYXZlX3Bvcz0n
MC0xLTYnOwovKiE0MDEwMyBTRVQgVElNRV9aT05FPUBPTERfVElNRV9aT05FICovOwoKLyohNDAx
MDEgU0VUIFNRTF9NT0RFPUBPTERfU1FMX01PREUgKi87Ci8qITQwMDE0IFNFVCBGT1JFSUdOX0tF
WV9DSEVDS1M9QE9MRF9GT1JFSUdOX0tFWV9DSEVDS1MgKi87Ci8qITQwMDE0IFNFVCBVTklRVUVf
Q0hFQ0tTPUBPTERfVU5JUVVFX0NIRUNLUyAqLzsKLyohNDAxMDEgU0VUIENIQVJBQ1RFUl9TRVRf
Q0xJRU5UPUBPTERfQ0hBUkFDVEVSX1NFVF9DTElFTlQgKi87Ci8qITQwMTAxIFNFVCBDSEFSQUNU
RVJfU0VUX1JFU1VMVFM9QE9MRF9DSEFSQUNURVJfU0VUX1JFU1VMVFMgKi87Ci8qITQwMTAxIFNF
VCBDT0xMQVRJT05fQ09OTkVDVElPTj1AT0xEX0NPTExBVElPTl9DT05ORUNUSU9OICovOwovKk0h
MTAwNjE2IFNFVCBOT1RFX1ZFUkJPU0lUWT1AT0xEX05PVEVfVkVSQk9TSVRZICovOwoKLS0gRHVt
cCBjb21wbGV0ZWQgb24gMjAyNi0xMC0xNSAxNToyNjoxNAo=
EOF
base64 -d /root/mariadb/shibuya-seed.b64 > /root/mariadb/shibuya-seed.sql
sha256sum /root/mariadb/shibuya-seed.sql
mariadb < /root/mariadb/shibuya-seed.sql
sed -n '/^-- CHANGE MASTER TO/p' /root/mariadb/shibuya-seed.sql
```

Dalam step ini output base64 dari `yaga` ditempel ke `shibuya-seed.b64`, lalu di-decode dan di-import. Nilai SHA-256 di kedua node harus sama.

##### `inumaki` : sambungkan replikasi
```bash
. /root/lab.env
REPL_PASS='PASTE_REPLICATION_PASSWORD_FROM_YAGA'
BINLOG_FILE='mariadb-bin.000001'
BINLOG_POS='1720'
cat > /root/mariadb/connect_replica.sql <<EOF
CHANGE MASTER TO
    MASTER_HOST='${YAGA}',
    MASTER_USER='replicator',
    MASTER_PASSWORD='${REPL_PASS}',
    MASTER_PORT=3306,
    MASTER_LOG_FILE='${BINLOG_FILE}',
    MASTER_LOG_POS=${BINLOG_POS},
    MASTER_CONNECT_RETRY=5;
START SLAVE;
SHOW SLAVE STATUS\G
EOF
mariadb < /root/mariadb/connect_replica.sql
```

![](inumaki%20-%20replikasi%20berhasil.png)

Dalam step ini kami menyambungkan `inumaki` ke `yaga` dengan password replikasi dan posisi binlog dari baris `CHANGE MASTER TO` di dump. Jangan memakai posisi dari `SHOW MASTER STATUS` yang diambil belakangan, karena posisinya bisa sudah berbeda dari snapshot. Kalau slave sudah pernah dikonfigurasi, hentikan replikasinya dulu sebelum mengganti parameter.

##### `yaga` : tambah data uji
![](yaga%20-%20tambah%20data%20uji.png)
```bash
. /root/lab.env
mariadb "$DB_NAME" -e "INSERT INTO request_logs (worker,client_ip,request_uri) VALUES ('replication-test','127.0.0.1','/replication-check');"
mariadb "$DB_NAME" -e "SELECT * FROM request_logs ORDER BY id DESC LIMIT 5;"
```

Dalam step ini kami menambah satu baris uji di `yaga`.

##### `inumaki` : cek replikasi

```bash
. /root/lab.env
mariadb -e 'SHOW SLAVE STATUS\G'
mariadb "$DB_NAME" -e "SELECT * FROM request_logs ORDER BY id DESC LIMIT 5;"
```

Dalam step ini kami mengecek status slave dan isi tabel di `inumaki`. Hasil yang benar: `Slave_IO_Running: Yes`, `Slave_SQL_Running: Yes`, tidak ada error, dan baris uji dari `yaga` muncul di `inumaki`. Kalau `Seconds_Behind_Master` bernilai `NULL` atau ada field error yang terisi, replikasi bermasalah.

Untuk persistensi, tambahkan `service mariadb start` ke boot hook kedua node.
![](yaga%20-%20cek%20data%20uji.png)
![](inumaki%20-%20cek%20data%20tereplikasi.png)
#### Kendala
##### `inumaki` tidak bisa menjangkau jaringan

Awalnya `inumaki` tidak bisa ping gateway `192.216.4.1`, bahkan `yaga` dan `tengen` yang satu switch (`Destination Host Unreachable`), padahal `jichi` bisa ping `tengen`. Setelah kabelnya disambung ulang, `eth0` sempat hilang (`Cannot find device "eth0"`). Kami membuat ulang node `inumaki` dari template `debinet` di posisi dan switch yang sama (`Switch10`). Node baru mendapat MAC baru dan langsung bisa menjangkau gateway.

##### IP `yaga` dan `inumaki` berbeda dengan tabel

Config awal memakai `192.216.4.8` untuk `yaga` dan `192.216.4.9` untuk `inumaki`, sedangkan `lab.env` memakai `.30` dan `.31`. Replikasi gagal dengan `Can't connect to server on '192.216.4.30' (113 "No route to host")`. Kami mengganti config interface kedua node menjadi `.30` dan `.31` sesuai tabel.

##### Paket MariaDB hilang setelah node diubah

Setelah IP `yaga` diganti dan node di-restart, paket MariaDB hilang (`mariadb: unrecognized service`), tetapi isi `/root` tetap ada. Karena config, script SQL, dan password tersimpan di `/root`, kami cukup menginstal ulang dan menjalankan ulang script-nya:

```bash
apt-get update
apt-get install -y mariadb-server mariadb-client openssl
cp /root/mariadb/60-shibuya.cnf /etc/mysql/mariadb.conf.d/
service mariadb restart
mariadb < /root/mariadb/create_shibuya.sql
mariadb -e "SHOW MASTER STATUS;"
```

Posisi binlog kembali `mariadb-bin.000001` / `1720`. Karena `yaga` mulai dari data kosong, tabel di `inumaki` kami kosongkan dulu supaya ID barisnya tidak bentrok, lalu replikasi disambungkan ulang:

```bash
. /root/lab.env
mariadb -e "STOP SLAVE; TRUNCATE TABLE ${DB_NAME}.request_logs; CHANGE MASTER TO MASTER_HOST='192.216.4.30', MASTER_USER='replicator', MASTER_PASSWORD='ISI_REPL_PASS_DARI_YAGA', MASTER_PORT=3306, MASTER_LOG_FILE='mariadb-bin.000001', MASTER_LOG_POS=1720, MASTER_CONNECT_RETRY=5; START SLAVE;"
```

##### `Access denied` karena password replikasi masih placeholder

Blok sambungkan replikasi sempat dijalankan dengan `REPL_PASS` yang masih placeholder, sehingga `Slave_IO_Running` tertahan di `Connecting` lalu gagal login (`Access denied for user 'replicator'@'192.216.4.31'`).

![](inumaki%20-%20replikasi%20gagal%20password%20placeholder.png)
![](inumaki%20-%20status%20replikasi%20connecting.png)

Kami mengambil password asli di `yaga` dengan `grep REPL_PASS /root/db-secrets.env`, lalu menjalankan ulang `CHANGE MASTER TO` dengan password tersebut. Setelah itu `Slave_IO_Running` dan `Slave_SQL_Running` sama-sama `Yes`.

### Soal 14
#### Brief Overview

Menjalankan halaman frontend di `maki` dan `panda`, serta API yang mencatat request ke database di `nanami`, `todo`, dan `kirara`. Frontend hanya boleh diakses dari `gojo`, dan API hanya dari `geto`.

#### Step by Step

##### Kelima worker : install nginx dan PHP-FPM

```bash
apt-get update
apt-get install -y nginx php-fpm php-mysql php-cli curl
mkdir -p /root/nginx /root/web /var/www/shibuya
PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
service "php${PHPV}-fpm" start
ls -l "/run/php/php${PHPV}-fpm.sock"
```

Dalam step ini kami menginstal nginx, PHP-FPM, dan driver MySQL untuk PHP.

	PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')

Berfungsi untuk mendeteksi versi PHP saat script dijalankan, supaya nama socket PHP-FPM tidak perlu ditulis manual.

##### `maki` & `panda` : halaman frontend
![](maki%20-%20konfigurasi%20frontend.png)
![](panda%20-%20konfigurasi%20frontend.png)
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

Dalam step ini kami membuat `index.php` yang menampilkan hostname worker, lalu menulis config nginx. Pastikan `hostname` sudah benar sebelum menjalankan blok ini.

	$name = htmlspecialchars(sprintf('%-16s', gethostname()), ENT_QUOTES, 'UTF-8');

Berfungsi untuk membuat panjang nama worker selalu 16 karakter, supaya `ab` di Soal 20 tidak menganggap perbedaan panjang halaman sebagai kegagalan. Sedangkan:

	allow $GOJO;
	deny all;

Berfungsi agar hanya `gojo` yang boleh mengakses frontend. nginx mengecek IP koneksi langsung, jadi header `X-Forwarded-For` palsu tidak bisa melewati aturan ini.

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

![](nanami%20-%20akses%20database.png)
![](todo%20-%20akses%20database.png)
![](kirara%20-%20akses%20database.png)

Dalam step ini kami menyimpan kredensial database di `/etc/shibuya-db.ini`. Tempel password **aplikasi** dari `yaga`, bukan password replikasi.

	install -o root -g www-data -m 0640 /root/shibuya-db.ini /etc/shibuya-db.ini

Berfungsi agar file hanya bisa dibaca oleh root dan grup `www-data` (PHP-FPM).

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

Dalam step ini kami membuat API PHP yang mencatat setiap request ke tabel `request_logs` di `yaga`.

	if (!$cfg || $peer !== $cfg['trusted_proxy']) {

Berfungsi untuk menolak request yang tidak datang langsung dari `geto`, dengan status HTTP 403.

	$client = $_SERVER['HTTP_X_REAL_IP'] ?? '';

Berfungsi untuk mengambil IP client asli dari header `X-Real-IP` yang diisi `geto`. Header ini hanya dipercaya karena koneksinya sudah dipastikan berasal dari `geto`. Sedangkan:

	$stmt->execute([$worker, $client, $uri]);

Berfungsi untuk menyimpan nama worker, IP client, dan URI ke database memakai prepared statement. Respons JSON baru dikirim setelah insert berhasil, dan password database tidak pernah ikut di respons.

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

![](nanami%20-%20virtual%20host%20api.png)
![](todo%20-%20virtual%20host%20api.png)
![](kirara%20-%20virtual%20host%20api.png)

Dalam step ini config nginx API dibuat dengan pola yang sama seperti frontend, tapi yang diizinkan hanya `geto` (`allow $GETO; deny all;`).

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

![](itadori%20-%20akses%20langsung%20ditolak.png)

Dalam step ini client Network 2 atau 3 mencoba mengakses worker secara langsung. Hasil yang benar: HTTP 403 untuk semua worker, walaupun header `Host` sudah benar.

##### `geto` : uji akses API yang diizinkan

```bash
. /root/lab.env
curl -i -H "Host: api.$ZONE" -H "X-Real-IP: $GETO" "http://$NANAMI/api"
```

![](geto%20-%20uji%20akses%20api%20nanami.png)

Dalam step ini kami menguji API dari `geto` sendiri. Karena request dibuat oleh `geto`, IP client yang tercatat adalah IP `geto`. Untuk frontend, uji dengan `curl -H "Host: $ZONE" http://$MAKI/` dari `gojo`. Jalur lengkap dari client dibuktikan di Soal 15–17.

##### Semua worker : script startup

```bash
#!/bin/sh
set -eu
PHPV=$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')
service "php${PHPV}-fpm" start
nginx -t
service nginx start
```

Dalam step ini kami menyiapkan isi `/root/start_workers.sh` untuk menyalakan PHP-FPM dan nginx. Daftarkan script ini ke boot hook setiap worker seperti di Soal 10.
#### Kendala
##### IP worker berbeda dengan tabel

Config awal memakai `192.216.4.2` sampai `.7` untuk `geto`, `maki`, `panda`, `nanami`, `todo`, dan `kirara`. Karena aturan `allow` dan `lab.env` memakai IP tabel, kami mengganti config interface keenam node sesuai tabel (`.11`, `.20`–`.24`) sebelum menginstal paket apa pun.

##### `nginx -t` gagal: `invalid number of arguments in "allow" directive`

`lab.env` belum dibuat di worker, sehingga `$GOJO` dan `$GETO` kosong dan config menjadi `allow ;`. Setelah `lab.env` dibuat, blok config dijalankan ulang dan `nginx -t` berhasil.

##### API menjawab `503 Database logging unavailable`

Log nginx di `nanami` (`/var/log/nginx/error.log`) menunjukkan `SQLSTATE[HY000] [2002] Connection refused`. Penyebabnya dua: MariaDB di `yaga` sedang tidak berjalan karena paketnya hilang (lihat Kendala Soal 13), dan `db_password` di `/etc/shibuya-db.ini` masih placeholder. Setelah MariaDB di `yaga` dipasang ulang, password aplikasi diisi di ketiga API worker:

```bash
sed -i "s/^db_password=.*/db_password=\"ISI_APP_DB_PASS_DARI_YAGA\"/" /etc/shibuya-db.ini /root/shibuya-db.ini
```

##### Client tidak mendapat IP dari DHCP

Saat uji akses langsung, `itadori` tidak punya IP (`Network unreachable`) dan `udhcpc` gagal (`no lease`). DHCP server di `tengen` berjalan, tetapi relay di `jichi` mati setelah node dinyalakan ulang. Kami menyalakannya kembali di `jichi`:

```bash
service isc-dhcp-relay start
ps -ef | grep '[d]hcrelay'
```

Setelah itu `itadori` mendapat `192.216.2.11` dari `tengen`.

### Soal 15
#### Brief Overview

Membagi request API di `geto` ke `nanami` dan `todo` dengan bobot 3:1, lalu membuktikan pembagiannya dari Network 4.

#### Step by Step

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

Dalam step ini kami membuat `/root/write_geto.sh`, yang menulis ulang `nginx.conf` sesuai tahap. Tahap `basic` dipakai di soal ini, `trusted` di Soal 17, dan `backup` di Soal 18.

	server $NANAMI:80 weight=3 $PRIMARY_FLAGS;
	server $TODO:80 weight=1 $PRIMARY_FLAGS;

Berfungsi untuk membagi request dengan perbandingan 3:1. Di tahap `basic`, `max_fails=0` membuat worker tidak pernah ditandai gagal selama uji rasio.

	worker_processes 1;

Berfungsi agar urutan pembagian request lebih mudah dibaca, karena hanya ada satu proses worker nginx. Sedangkan:

	geo $PEER_VAR \$allowed_api_peer {

Berfungsi untuk mengizinkan koneksi hanya dari `gojo` dan Network 4. Client Divisi A dan B tidak bisa mengakses `geto` secara langsung.

##### `tengen` : 40 request berurutan dari Network 4
![](tengen%20-%2040%20request%20berurutan.png)
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

Dalam step ini `tengen` mengirim 40 request ke API lewat `geto`, lalu Python menghitung jumlah respons per worker. Hasil yang benar: mendekati 30 `nanami` dan 10 `todo`, dan `kirara` belum menerima request. Yang dilaporkan adalah hitungan asli, bukan rasio di config. Log akses `geto` dan tabel `request_logs` di `yaga` bisa dipakai sebagai bukti tambahan.

### Soal 16
#### Brief Overview

Menjadikan `gojo` pintu masuk publik: `/` dibagi round robin ke `maki` dan `panda`, sedangkan `/api` diteruskan ke `geto`, untuk nama apex maupun `www`.

#### Step by Step

##### `gojo` : install nginx dan script penulis config
![](gojo%20-%20write%20gojo%20baseline.png)
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

Dalam step ini kami membuat `/root/write_gojo.sh`. Tahap `baseline` dipakai di soal ini, dan tahap `cached` di Soal 19. File rate limit sengaja dibuat kosong sampai Soal 19.

	location = /api {
	location ^~ /api/ {

Berfungsi untuk menangkap `/api` dan `/api/...` lalu meneruskannya ke `geto`. Path lain seperti `/apix` tetap masuk ke frontend.

	proxy_pass http://$GETO;

Berfungsi untuk meneruskan request ke `geto` tanpa mengubah URI, jadi `/api` dan query string-nya tetap utuh.

	upstream frontend_cluster {

Berfungsi untuk membagi request frontend secara round robin ke `maki` dan `panda`. Sedangkan:

	log_format audit escape=json

Berfungsi untuk mencatat setiap request dalam format JSON, termasuk upstream, status cache, dan status rate limit. Log ini dipakai di Soal 19 dan 20.

##### `itadori` atau `ino` : cek kedua jalur dan kedua nama
![](itadori%20-%20uji%20frontend%20dan%20api.png)
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

Dalam step ini kami mengakses frontend dan API masing-masing 8 kali, lalu mengecek nama `www`. Hasil yang benar: halaman frontend bergantian menampilkan `maki` dan `panda`, dan JSON API menampilkan `nanami` atau `todo` beserta ID log database. Sebelum Soal 17, IP client yang tercatat bisa masih berupa IP `gojo`.

Uji pembagian frontend ini sebelum cache dinyalakan di Soal 19.

#### Kendala
##### Baris `server_name` rusak saat di-copy

Saat command di-copy dari tampilan markdown yang sudah dirender, teks `www.$ZONE` berubah menjadi link markdown, sehingga `write_gojo.sh` berisi `server_name $ZONE [www.$ZONE](https://www.$ZONE);`. Kami memperbaikinya tanpa menulis ulang script:

```bash
sed -i 's/\[\(w\{3\}\.\$ZONE\)\]([^)]*)/\1/' /root/write_gojo.sh
grep -n 'server_name' /root/write_gojo.sh
```

Selain itu, resolver di `gojo` belum diisi sehingga `apt-get` gagal resolve `deb.debian.org`. Setelah resolver diisi seperti di Persiapan, nginx bisa diinstal.

### Soal 17
#### Brief Overview

Meneruskan IP asli client lewat `X-Real-IP` dan `X-Forwarded-For` di `gojo` dan `geto`, hanya mempercayai header dari `gojo`, dan menyimpan IP client yang sudah diverifikasi di `yaga`.

#### Step by Step

##### `gojo` : cek perilaku header
![](gojo%20-%20cek%20header.png)
```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $remote_addr;
```

Dalam step ini tidak ada yang diubah, karena config dari Soal 16 sudah berisi dua baris di atas. Dua baris itu hanya isi config nginx, bukan command shell. Kalau diketik di terminal, hasilnya `proxy_set_header: command not found`. `gojo` mengisi kedua header dengan IP koneksi client (`$remote_addr`), jadi header palsu dari client tertimpa. `jichi` tidak memakai NAT untuk traffic internal, jadi IP yang dilihat `gojo` adalah IP client asli.

##### `geto` : aktifkan trusted real-IP
![](geto%20-%20trusted%20real%20ip.png)
```bash
nginx -V 2>&1 | grep -- --with-http_realip_module
/root/write_geto.sh trusted
```

Dalam step ini kami mengecek modul realip, lalu menjalankan tahap `trusted`. Config yang ditambahkan:

```nginx
set_real_ip_from GOJO_IP;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

	set_real_ip_from GOJO_IP;

Berfungsi agar `geto` hanya mempercayai header `X-Forwarded-For` dari `gojo`. `GOJO_IP` diisi IP `gojo` oleh script. Untuk koneksi dari `gojo`, `$remote_addr` berubah menjadi IP client asli, sedangkan `$realip_remote_addr` tetap berisi IP `gojo`. Aturan akses memakai `$realip_remote_addr`, jadi request client yang diteruskan `gojo` tetap diizinkan.

##### `itadori` : kirim request bertanda
![](itadori%20-%20request%20bertanda%20network2.png)
```bash
. /root/lab.env
ip -4 addr show eth0
curl -fsS "http://$ZONE/api?case=network2"
```

##### `ino` : kirim request bertanda
![](ino%20-%20request%20bertanda%20network3.png)
```bash
. /root/lab.env
ip -4 addr show eth0
curl -fsS "http://$ZONE/api?case=network3"
```

Dalam step ini setiap client mengirim request dengan penanda `case=network2` atau `case=network3`, supaya barisnya mudah dicari di database.

##### Client : coba header palsu
![](ino%20-%20uji%20header%20palsu.png)
```bash
. /root/lab.env
curl -fsS -H 'X-Real-IP: 203.0.113.123' \
    -H 'X-Forwarded-For: 203.0.113.123' \
    "http://$ZONE/api?case=spoof-check"
```

Dalam step ini client mengirim `X-Real-IP` dan `X-Forwarded-For` palsu (`203.0.113.123`). Hasil yang benar: JSON tetap menampilkan IP asli client (192.216.2.x atau 192.216.3.x).

##### `yaga` : cek database
![](yaga%20-%20cek%20client%20ip%20di%20database.png)
```bash
. /root/lab.env
mariadb "$DB_NAME" -e "SELECT id,worker,client_ip,request_uri,created_at FROM request_logs WHERE request_uri LIKE '%case=%' ORDER BY id DESC LIMIT 12;"
```

Dalam step ini kami mengambil baris yang punya penanda `case=`. Hasil yang benar: baris `network2` sesuai IP `itadori`, baris `network3` sesuai IP `ino`, dan `spoof-check` tetap berisi IP asli. Baris dari sebelum Soal 17 bisa masih berisi IP proxy, jadi bedakan dari penanda dan waktunya.

### Soal 18
#### Brief Overview

Menjadikan `kirara` sebagai backup, memakai `max_fails=2` dan `fail_timeout=5s` untuk `nanami` dan `todo`, lalu mematikan kedua worker utama dan membuktikan respons tetap datang dari `kirara`.

#### Step by Step

##### `geto` : config upstream akhir
![](geto%20-%20upstream%20backup.png)
```bash
/root/write_geto.sh backup
cat /root/nginx/geto.conf
```

Dalam step ini kami menjalankan tahap `backup`. Upstream sekarang berisi:

```nginx
upstream api_cluster {
    zone api_cluster 64k;
    server NANAMI_IP:80 weight=3 max_fails=2 fail_timeout=5s;
    server TODO_IP:80 weight=1 max_fails=2 fail_timeout=5s;
    server KIRARA_IP:80 backup;
}
```

	max_fails=2 fail_timeout=5s

Berfungsi agar worker dianggap gagal setelah 2 kegagalan dalam 5 detik, lalu tidak dipakai selama 5 detik. Ini pengecekan pasif: kegagalan baru terdeteksi dari request sungguhan, bukan dari probe berkala. Sedangkan:

	server KIRARA_IP:80 backup;

Berfungsi agar `kirara` hanya menerima request saat kedua worker utama tidak tersedia. `proxy_next_upstream` dan `proxy_next_upstream_tries 3` dari Soal 15 membuat request yang gagal dicoba ulang ke worker lain sebelum respons dikirim ke client.

##### Client : kondisi normal
![](ino%20-%20kondisi%20normal.png)
```bash
. /root/lab.env
for n in $(seq 1 8); do
    curl -fsS "http://$ZONE/api?case=healthy"
done
```

Dalam step ini kami mengirim 8 request saat semua worker sehat. `kirara` tidak boleh menerima request.

##### `nanami` & `todo` : matikan nginx
![](nanami%20-%20matikan%20nginx.png)
![](todo%20-%20matikan%20nginx.png)
```bash
service nginx stop
ss -ltnp | grep ':80' || true
```

Dalam step ini nginx dimatikan di kedua worker utama. `ss` memastikan tidak ada lagi yang mendengarkan di port 80.

##### Client : lihat respons dari backup
![](ino%20-%20respons%20dari%20backup.png)
```bash
. /root/lab.env
for n in $(seq 1 12); do
    curl -sS -w ' HTTP=%{http_code}\n' \
        "http://$ZONE/api?case=failover"
    sleep 1
done
```

Dalam step ini kami mengirim 12 request dengan jeda 1 detik. Hasil yang benar: JSON dengan `worker: kirara`, ID log database, dan HTTP 200 tanpa 502. Catat semua status, termasuk yang gagal. Log `/var/log/nginx/geto_access.log` bisa menunjukkan beberapa percobaan upstream untuk satu request yang akhirnya berhasil.

##### `nanami` & `todo` : nyalakan lagi
![](nanami%20-%20nyalakan%20nginx.png)
![](todo%20-%20nyalakan%20nginx.png)
```bash
service nginx start
```

Dalam step ini nginx dinyalakan kembali. Setelah lewat 5 detik, ulangi request dari client: `nanami` dan `todo` harus kembali menerima request, dan `kirara` berhenti. Biarkan kedua worker utama tetap menyala untuk soal berikutnya.

### Soal 19
#### Brief Overview

Menyimpan cache halaman frontend selama 10 menit dengan header `X-Cache-Status`, dan membatasi `/api` maksimal 10 request per detik per IP client.

#### Step by Step

##### `gojo` : aktifkan cache frontend
![](gojo%20-%20aktifkan%20cache.png)
```bash
/root/write_gojo.sh cached
```

Dalam step ini kami menjalankan tahap `cached` dari `write_gojo.sh`.

	proxy_cache_valid 200 10m;

Berfungsi untuk menyimpan respons HTTP 200 selama 10 menit.

	proxy_cache_key "$scheme|$host|$request_uri";

Berfungsi agar setiap kombinasi scheme, hostname, dan URI punya cache sendiri. Sedangkan:

	proxy_cache_bypass $http_authorization $http_cookie;

Berfungsi agar request yang membawa `Authorization` atau `Cookie` tidak memakai dan tidak disimpan ke cache. Respons API tetap tidak di-cache (`proxy_cache off`).

##### Client : MISS lalu HIT di URL yang sama
![](ino%20-%20cache%20miss%20lalu%20hit.png)
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

Dalam step ini kami membuka URL yang sama dua kali. URL diberi token waktu supaya cache-nya masih kosong. Hasil yang benar: request pertama `X-Cache-Status: MISS`, request kedua `HIT`, dan isi halamannya sama (`cmp` tidak menampilkan apa-apa).

Selama cache berlaku, nama worker bisa tetap `maki` atau `panda` saja, karena request HIT tidak diteruskan ke worker.

##### `gojo` : aktifkan rate limit API
![](gojo%20-%20aktifkan%20rate%20limit.png)
```bash
cat > /root/nginx/api-rate.conf <<'EOF'
limit_req zone=api_per_ip;
limit_req_status 429;
limit_req_log_level notice;
EOF
cp /root/nginx/api-rate.conf /etc/nginx/snippets/shibuya-rate.conf
nginx -t && service nginx reload
```

Dalam step ini kami mengisi file rate limit yang sebelumnya kosong.

	limit_req zone=api_per_ip;

Berfungsi untuk memakai zona `api_per_ip` yang sudah dibuat di Soal 16, dengan `rate=10r/s` dan kunci IP client (`$binary_remote_addr`). Tidak ada `burst`, jadi request yang berlebihan langsung ditolak. Sedangkan:

	limit_req_status 429;

Berfungsi agar request yang ditolak mendapat status 429.

##### Client : kirim request API bersamaan
![](ino%20-%20request%20api%20bersamaan.png)
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

Dalam step ini client mengirim 40 request sekaligus di background, lalu menghitung status yang diterima. Hasil yang benar: campuran 200 dan 429, dengan jumlah yang tergantung waktu. Status 502 atau 503 berarti ada masalah di upstream, bukan rate limit.

##### `gojo` : cek alasan di access log
![](gojo%20-%20log%20rate%20limit.png)
```bash
tail -n 60 /var/log/nginx/gojo_access.jsonl
```

Dalam step ini kami membaca log JSON di `gojo`. Request yang ditolak rate limit punya `status` 429 dan `limit` bernilai `REJECTED`, tanpa diproses oleh upstream.

### Soal 20
#### Brief Overview

Menjalankan ApacheBench dari `kusakabe` dengan `-n 600 -c 30`, mencatat RPS, latency, dan kegagalan, lalu membandingkan API tanpa dan dengan rate limit.

#### Step by Step

##### `kusakabe` : install ab

```bash
apk update
apk add apache2-utils curl
mkdir -p /root/benchmark
. /root/lab.env
```

Dalam step ini kami menginstal `ab` dan menyiapkan folder hasil benchmark. Gunakan client, jumlah request, concurrency, dan URL yang sama untuk semua percobaan, dan jangan menjalankan benchmark lain bersamaan.

##### `gojo` : matikan rate limit untuk baseline

```bash
printf '# API limiting disabled for controlled baseline\n' \
    > /root/nginx/api-rate.conf
cp /root/nginx/api-rate.conf /etc/nginx/snippets/shibuya-rate.conf
nginx -t && service nginx reload
```

Dalam step ini hanya rate limit API yang dimatikan. Cache frontend tetap aktif.

##### `kusakabe` : baseline
![](kusakabe%20-%20benchmark%20baseline.png)
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

Dalam step ini kami memanaskan cache halaman utama, lalu menjalankan `ab` ke frontend dan API.

	ab -l -n 600 -c 30 "http://$ZONE/api?bench=baseline" \

Berfungsi untuk mengirim 600 request dengan 30 koneksi bersamaan ke API. `-l` dipakai karena panjang JSON API berbeda-beda (nama worker dan ID log), sehingga `ab` tidak menghitungnya sebagai gagal. Opsi ini tidak menyembunyikan status non-2xx.

##### `gojo` : tandai log dan nyalakan rate limit
![](gojo%20-%20tandai%20log%20rate%20limit.png)
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

Dalam step ini kami mencatat jumlah baris log saat ini, lalu menyalakan kembali rate limit. Jumlah baris ini dipakai untuk mengambil log percobaan berikutnya saja.

##### `kusakabe` : ulangi dengan rate limit aktif
![](kusakabe%20-%20benchmark%20rate%20limit%20aktif.png)
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

Dalam step ini benchmark yang sama diulang dengan penanda `bench=active`.

##### `gojo` : ringkas request API
![](gojo%20-%20ringkasan%20request%20api.png)
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

Dalam step ini kami mengambil baris log yang baru, lalu Python menghitung status HTTP, status rate limit, jumlah penolakan 429, dan jumlah 5xx.

##### Tabel hasil

Isi tabel ini dari output `ab` dan log `gojo` yang asli:

| Metrik | Frontend baseline | Frontend saat rate limit API aktif | API baseline | API dengan rate limit |
| --- | --- | --- | --- | --- |
| Request / concurrency | 600 / 30 | 600 / 30 | 600 / 30 | 600 / 30 |
| Status cache | Catat HIT/MISS | Kondisi cache sama | Off | Off |
| Requests per second | | | | |
| Rata-rata latency (ms) | | | | |
| p95 latency (ms) | | | | |
| Failed requests | | | | |
| Non-2xx responses | | | | |
| 429 dengan `REJECTED` | | | 0 | |
| HTTP 5xx | | | | |
| Connection errors | | | | |

`ab` menampilkan dua nilai `Time per request`. Nilai pertama adalah rata-rata latency per request dengan memperhitungkan concurrency, sedangkan nilai kedua adalah rata-rata waktu per request untuk semua request bersamaan. Tulis nilai mana yang dipakai, dan ambil p95 dari baris 95% di tabel persentase.

Cara membaca hasil:

- Rate limit hanya berlaku untuk `/api`. Perbedaan hasil frontend dengan cache yang sama adalah variasi pengukuran, bukan efek rate limit.
- Respons 429 yang cepat bisa menaikkan total RPS walaupun request API yang benar-benar diproses berkurang. Laporkan jumlah respons 200 secara terpisah.
- `Failed requests` dan `Non-2xx responses` di `ab` mengukur hal yang berbeda.
- Status 429 dengan `limit=REJECTED` berarti ditolak rate limit. Status 5xx dari upstream berarti masalah di backend. Status 502/504 dari `gojo` bisa berarti gagal koneksi atau timeout, jadi cek field upstream dan `error.log`.
- Kegagalan koneksi di sisi client bisa tidak tercatat di log `gojo`, jadi cek juga rincian connect/receive/exception di output `ab`.
