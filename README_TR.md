## Monad Mainnet/Testnet Node Kurulum & Optimizasyon Rehberi
Bu rehber, Ubuntu 24.04 üzerinde yüksek performanslı NVMe SSD (TrieDB) kullanarak Monad Node kurulum adımlarını ve sistem optimizasyonlarını içerir.

# 🛠️ Sistem Gereksinimleri
OS: Ubuntu 24.04 LTS (Kernel v6.8.0.60+ Şarttır)

CPU: 16 Core+ (4.5GHz+ frekans önerilir)

RAM: 32 GB+ (Mainnet için 64GB-128GB önerilir)

Disk: 2 x 2TB NVMe SSD (BAĞIMSIZ/NO RAID ŞART. Biri OS/BFT, diğeri TrieDB için)

Network: 1 Gbps+ (Saniyede 70.000 PPS kapasiteli)

🚨 ADIM 0: Kritik Sistem Kontrolü (Ölümcül Kernel Hatası)
Linux kernel sürümleri v6.8.0.56 ile v6.8.0.59 arasında Monad node'unu kilitleyen bir hata barındırır. Kuruluma başlamadan önce kontrol edin:

```Bash
uname -r
```
Eğer sürümünüz 56, 57, 58 veya 59 ise, kuruluma devam etmeden önce sistemi güncelleyip yeniden başlatın:

```Bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```
(Yeniden açıldığında sürümün v6.8.0.60 veya üstü olduğunu teyit edin.)

# 🚀 ADIM 1: Temel Paketler ve Hazırlık
```Bash
# Sistem araçlarını kurun
sudo apt install -y curl nvme-cli aria2 jq parted ufw linux-tools-common linux-tools-$(uname -r)

# Monad Deposunu ve GPG Anahtarını Ekle
sudo curl -fsSL https://pkg.category.xyz/keys/public-key.asc | sudo gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg

sudo cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

# Monad Kurulumu
sudo apt update
sudo apt install -y monad
sudo apt-mark hold monad

# Kullanıcı ve Klasör Yapısı
sudo useradd -m -s /bin/bash monad
sudo mkdir -p /home/monad/monad-bft/config /home/monad/monad-bft/ledger /home/monad/monad-bft/config/forkpoint /home/monad/monad-bft/config/validators
```
⚙️ ADIM 2: Donanım Optimizasyonu (CPU & Performans)
Monad'ın gecikmesiz çalışması için işlemciyi enerji tasarrufu modundan çıkarıp "Performans" moduna alıyoruz:

Bash
# İşlemciyi maksimum performans moduna sabitleyin
sudo cpupower frequency-set -g performance

# Kontrol (Governor kısmında "performance" yazmalıdır)
cpupower frequency-info | grep "current policy"
💾 ADIM 3: Disk Yapılandırması (TrieDB - Bağımsız NVMe)
Önemli: Sunucunuzda disklerin RAID yapılmadığından emin olun. TrieDB için ayrı bir fiziksel disk (/dev/nvme1n1) kullanacağız.

Bash
# Disk Değişkenini Tanımla (Kendi diskinize göre kontrol edin: lsblk)
export TRIEDB_DRIVE=/dev/nvme1n1

# Diski Bölümlendir (GPT)
sudo parted -s $TRIEDB_DRIVE mklabel gpt
sudo parted -s $TRIEDB_DRIVE mkpart triedb 0% 100%

# I/O Performansı İçin Udev Kuralı Oluştur
PARTUUID=$(lsblk -o PARTUUID $TRIEDB_DRIVE | tail -n 1)
echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" | sudo tee /etc/udev/rules.d/99-triedb.rules

# Kuralları Uygula
sudo udevadm trigger
sudo udevadm control --reload
sudo udevadm settle

# TrieDB Servisini Başlat
sudo systemctl start monad-mpt
🔑 ADIM 4: Yapılandırma ve Cüzdan Oluşturma
Bash
# Yapılandırma dosyalarını çekin
MF_BUCKET=https://bucket.monadinfra.com
curl -o /home/monad/.env $MF_BUCKET/config/testnet/latest/.env.example
curl -o /home/monad/monad-bft/config/node.toml $MF_BUCKET/config/testnet/latest/full-node-node.toml

# Şifre Üret ve Yedekle
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env
sudo mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup

# Keystore (Cüzdan) Oluşturma
monad-keystore create --key-type secp --keystore-path /home/monad/monad-bft/config/id-secp --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup
monad-keystore create --key-type bls --keystore-path /home/monad/monad-bft/config/id-bls --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup
🛡️ ADIM 5: Firewall ve Ağ Ayarları (RaptorCast Uyarısı)
⚠️ DİKKAT: Monad saniyede ~70.000 paket (PPS) trafik üretir. Sunucu sağlayıcınızın (Hetzner, Latitude vb.) dış panelindeki DDoS korumalarını kapatın veya en esnek hale getirin.

Bash
# SSH ve Monad Portlarına izin ver
sudo ufw allow ssh
sudo ufw allow 8000/tcp
sudo ufw allow 8000/udp
sudo ufw --force enable

# Sahiplik İzinlerini Düzenle
sudo chown -R monad:monad /home/monad/

# Servisleri Başlat
sudo systemctl enable monad-bft monad-execution monad-rpc
sudo systemctl start monad-bft monad-execution monad-rpc
📊 İzleme
Node durumunu kontrol etmek için:

Bash
# Logları izle
journalctl -u monad-bft -f

# Resmi durum scripti
curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh -o /usr/local/bin/monad-status && chmod +x /usr/local/bin/monad-status
monad-status
⚠️ Hatırlatma: /opt/monad/backup/ klasöründeki tüm dosyaları mutlaka bilgisayarınıza veya güvenli bir yere yedekleyin. Bu dosyalar node'unuzun kimliğidir.
