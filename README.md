#📘 Monad Testnet Full Node Kurulum Rehberi
Bu rehber, Ubuntu 24.04 üzerinde ve yüksek performanslı NVMe SSD (TrieDB) kullanarak Monad Testnet Full Node kurulum adımlarını içerir.

Not: Bu rehberde disk yolu /dev/nvme1n1 olarak baz alınmıştır. Kendi sunucunuzdaki disk yapısına göre bu yolu güncellemelisiniz.

🛠️ Sistem Gereksinimleri
OS: Ubuntu 24.04 LTS

CPU: 16 Core+ (Önerilen)

RAM: 32 GB+

Disk: Yüksek hızlı NVMe SSD (TrieDB için ayrı disk/bölüm önerilir)

Network: 1 Gbps+ (100 Mbps upload minimum)

🚀 Adım 1: Temel Paketler ve Hazırlık
Sistemi güncelleyin ve gerekli araçları kurun:

Bash

# Sistemi güncelle
```
apt update && apt upgrade -y
apt install -y curl nvme-cli aria2 jq parted ufw
```

# Monad Deposunu Ekle
cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

# GPG Anahtarını Ekle
curl -fsSL https://pkg.category.xyz/keys/public-key.asc \
  | gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg

# Monad Paketini Kur
apt update
apt install -y monad
apt-mark hold monad

# Monad Kullanıcısını ve Klasörleri Oluştur
useradd -m -s /bin/bash monad
mkdir -p /home/monad/monad-bft/config \
         /home/monad/monad-bft/ledger \
         /home/monad/monad-bft/config/forkpoint \
         /home/monad/monad-bft/config/validators
💾 Adım 2: Disk Yapılandırması (TrieDB)
Monad performansı için NVMe diski optimize ediyoruz. (Dikkat: TRIEDB_DRIVE değişkenini kendi diskinize göre ayarlayın, örn: /dev/nvme0n1)

Bash

# Disk Değişkenini Tanımla
export TRIEDB_DRIVE=/dev/nvme1n1

# Diski Bölümlendir (GPT ve %100 Alan)
parted -s $TRIEDB_DRIVE mklabel gpt
parted -s $TRIEDB_DRIVE mkpart triedb 0% 100%

# İzinleri Ayarla (Udev Rule)
PARTUUID=$(lsblk -o PARTUUID $TRIEDB_DRIVE | tail -n 1)

echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" \
  | tee /etc/udev/rules.d/99-triedb.rules

# Kuralları Uygula
udevadm trigger
udevadm control --reload
udevadm settle

# Veritabanını Başlat
systemctl start monad-mpt
⚙️ Adım 3: Yapılandırma ve Anahtarlar
Testnet yapılandırma dosyalarını indirin ve kimlik oluşturun.

Bash

# Testnet Dosyalarını İndir
MF_BUCKET=https://bucket.monadinfra.com
curl -o /home/monad/.env $MF_BUCKET/config/testnet/latest/.env.example
curl -o /home/monad/monad-bft/config/node.toml $MF_BUCKET/config/testnet/latest/full-node-node.toml

# Şifre Oluştur ve Kaydet
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env

# Şifreyi Yedek Klasörüne Yaz
mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup

# Cüzdan ve Node Anahtarlarını Üret
bash <<'EOF'
set -e
source /home/monad/.env
monad-keystore create --key-type secp --keystore-path /home/monad/monad-bft/config/id-secp --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup
monad-keystore create --key-type bls --keystore-path /home/monad/monad-bft/config/id-bls --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup
grep "public key" /opt/monad/backup/secp-backup /opt/monad/backup/bls-backup | tee /home/monad/pubkey-secp-bls
EOF
📝 Adım 4: Node İsmi ve Peer Discovery
Node isminizi ve ağ imzanızı ayarlayın.

1. İsim Ayarı: NODE_ISMINIZ kısmını kendi takma adınızla değiştirin.

Bash

sed -i 's/node_name = .*/node_name = "full_NODE_ISMINIZ"/' /home/monad/monad-bft/config/node.toml
sed -i 's/beneficiary = .*/beneficiary = "0x0000000000000000000000000000000000000000"/' /home/monad/monad-bft/config/node.toml
2. İmza Oluşturma: Aşağıdaki komutu çalıştırın, çıkan sonucu node.toml dosyasına eklemeniz gerekecek.

Bash

source /home/monad/.env
monad-sign-name-record \
  --address $(curl -s4 ifconfig.me):8000 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num 0
Çıktıdaki değerleri node.toml dosyasına kaydedin.

🔥 Adım 5: Başlatma
İzinleri verin ve servisleri başlatın.

Bash

# Dosya Sahipliğini Düzelt
chown -R monad:monad /home/monad/

# Firewall Ayarları
ufw allow ssh
ufw allow 8000/tcp
ufw allow 8000/udp
ufw --force enable

# Servisleri Başlat
systemctl enable monad-bft monad-execution monad-rpc
systemctl start monad-bft monad-execution monad-rpc
📊 İzleme ve Kontrol
Node durumunu kontrol etmek için resmi script'i kullanabilirsiniz.

Bash

# Script'i İndir
curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh -o /usr/local/bin/monad-status && chmod +x /usr/local/bin/monad-status

# Durumu Kontrol Et
monad-status

# Logları İzle
journalctl -u monad-bft -f
⚠️ Önemli Not: Yedekleme
Kurulum sonrası /opt/monad/backup/ klasöründeki dosyaları mutlaka güvenli bir yere yedekleyiniz. Bu dosyalar node kimliğinizdir.
