#Monad Testnet Full Node Setup Guide (NVMe Optimized)
📘 Monad Testnet Full Node Setup Guide
This guide covers the installation steps for a Monad Testnet Full Node on Ubuntu 24.04, optimized for high-performance NVMe SSDs (specifically configured for TrieDB).

Note: This guide assumes the TrieDB disk path is /dev/nvme1n1. Please adjust the path according to your own server configuration.

🛠️ System Requirements
OS: Ubuntu 24.04 LTS

CPU: 16 Core+ (Recommended)

RAM: 32 GB+

Storage: High-speed NVMe SSD (Dedicated drive/partition for TrieDB is highly recommended)

Network: 1 Gbps+ (100 Mbps upload minimum)

🚀 Step 1: Preparation & Dependencies
Update the system and install necessary tools:

Bash

# Update System
apt update && apt upgrade -y
apt install -y curl nvme-cli aria2 jq parted ufw

# Add Monad Repository
cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

# Add GPG Key
curl -fsSL https://pkg.category.xyz/keys/public-key.asc \
  | gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg

# Install Monad Package
apt update
apt install -y monad
apt-mark hold monad

# Create User and Directories
useradd -m -s /bin/bash monad
mkdir -p /home/monad/monad-bft/config \
         /home/monad/monad-bft/ledger \
         /home/monad/monad-bft/config/forkpoint \
         /home/monad/monad-bft/config/validators
💾 Step 2: Disk Configuration (TrieDB)
Optimizing the NVMe drive for Monad's high-throughput requirements. (Replace /dev/nvme1n1 with your actual disk path)

Bash

# Define Disk Variable
export TRIEDB_DRIVE=/dev/nvme1n1

# Partition the Disk (GPT & 100% Usage)
parted -s $TRIEDB_DRIVE mklabel gpt
parted -s $TRIEDB_DRIVE mkpart triedb 0% 100%

# Set Permissions (Udev Rule)
PARTUUID=$(lsblk -o PARTUUID $TRIEDB_DRIVE | tail -n 1)

echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" \
  | tee /etc/udev/rules.d/99-triedb.rules

# Apply Rules
udevadm trigger
udevadm control --reload
udevadm settle

# Start Database Service
systemctl start monad-mpt
⚙️ Step 3: Configuration & Keys
Download testnet configs and generate node identity.

Bash

# Download Config Files
MF_BUCKET=https://bucket.monadinfra.com
curl -o /home/monad/.env $MF_BUCKET/config/testnet/latest/.env.example
curl -o /home/monad/monad-bft/config/node.toml $MF_BUCKET/config/testnet/latest/full-node-node.toml

# Generate Password
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env

# Backup Password (IMPORTANT)
mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup

# Create Keys (SECP & BLS)
bash <<'EOF'
set -e
source /home/monad/.env
monad-keystore create --key-type secp --keystore-path /home/monad/monad-bft/config/id-secp --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup
monad-keystore create --key-type bls --keystore-path /home/monad/monad-bft/config/id-bls --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup
# Display Public Address
grep "public key" /opt/monad/backup/secp-backup /opt/monad/backup/bls-backup | tee /home/monad/pubkey-secp-bls
EOF
📝 Step 4: Node Identity & Peer Discovery
Set your node name and generate the network signature.

1. Set Node Name: Replace YOUR_MONIKER with your desired node name.

Bash

sed -i 's/node_name = .*/node_name = "YOUR_MONIKER"/' /home/monad/monad-bft/config/node.toml
2. Generate Signature: Run the following command and add the output to node.toml under [peer_discovery].

Bash

source /home/monad/.env
monad-sign-name-record \
  --address $(curl -s4 ifconfig.me):8000 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num 0
🔥 Step 5: Launch & Security
Finalize permissions and start services.

Bash

# Fix Ownership
chown -R monad:monad /home/monad/

# Configure Firewall (UFW)
ufw allow ssh
ufw allow 8000/tcp
ufw allow 8000/udp
ufw --force enable

# Enable & Start Services
systemctl enable monad-bft monad-execution monad-rpc
systemctl start monad-bft monad-execution monad-rpc
📊 Monitoring
You can use the community status script to check sync status.

Bash

# Install Status Tool
curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh -o /usr/local/bin/monad-status && chmod +x /usr/local/bin/monad-status

# Check Status
monad-status

# View Logs
journalctl -u monad-bft -f
Author: Bahadır Özdemir

GitHub: bozdemir52
