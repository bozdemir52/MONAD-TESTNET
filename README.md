## Monad Testnet Node Setup Guide (High Performance & NVMe Optimized)
This guide covers the advanced installation steps for a Monad Testnet Node on Ubuntu 24.04. It is strictly optimized for high performance NVMe SSDs (TrieDB) and includes OS-level CPU & Network optimizations to prevent sync lags and missed blocks.

Note: This guide assumes the TrieDB disk path is /dev/nvme1n1. Please adjust the path according to your own server configuration (lsblk).

## 🛠️ System Requirements
OS: Ubuntu 24.04 LTS

CPU: 16 Core+ (Physical cores preferred over logical threads)

RAM: 32 GB+

Storage: High-speed NVMe SSD (Dedicated drive/partition for TrieDB is highly recommended)

Network: 1 Gbps+ (100 Mbps upload minimum)

## 🚀 Step 1: Preparation & Dependencies
Update the system and install necessary tools:

```Bash
apt update && apt upgrade -y
apt install -y curl nvme-cli aria2 jq parted ufw cpufrequtils python3-pip
Add Monad Repository & GPG Key:
```
Bash
cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

curl -fsSL https://pkg.category.xyz/keys/public-key.asc \
  | gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg
Install Monad Package:

Bash
apt update
apt install -y monad
apt-mark hold monad
Create User and Directories:

Bash
useradd -m -s /bin/bash monad
mkdir -p /home/monad/monad-bft/config \
         /home/monad/monad-bft/ledger \
         /home/monad/monad-bft/config/forkpoint \
         /home/monad/monad-bft/config/validators
⚡ Step 2: System & CPU Optimizations (Crucial)
High-performance chains like Monad require physical CPU cores to run without context-switching interruptions.

1. Disable SMT (Hyper-Threading):
Disabling logical threads ensures that execution processes get dedicated physical cores.

Bash
echo off > /sys/devices/system/cpu/smt/control
# To make this persistent across reboots, we add it to cron:
(crontab -l 2>/dev/null; echo "@reboot echo off > /sys/devices/system/cpu/smt/control") | crontab -
2. Set CPU Governor to Performance:

Bash
echo 'GOVERNOR="performance"' > /etc/default/cpufrequtils
systemctl restart cpufrequtils
3. Increase Open File Limits (ulimit):
Nodes open thousands of P2P connections and database files.

Bash
cat <<EOF >> /etc/security/limits.conf
* soft nofile 1048576
* hard nofile 1048576
root soft nofile 1048576
root hard nofile 1048576
monad soft nofile 1048576
monad hard nofile 1048576
EOF
4. Network Tuning (sysctl):

Bash
cat <<EOF > /etc/sysctl.d/99-monad.conf
net.core.rmem_max=26214400
net.core.wmem_max=26214400
net.ipv4.tcp_rmem=4096 87380 26214400
net.ipv4.tcp_wmem=4096 65536 26214400
net.ipv4.tcp_tw_reuse=1
fs.file-max=1048576
EOF
sysctl --system
💾 Step 3: Disk Configuration (TrieDB)
Optimizing the NVMe drive for Monad's high-throughput requirements. (Replace /dev/nvme1n1 with your actual disk path)

Bash
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
⚙️ Step 4: Configuration & Keys
Download testnet configs and generate your node's identity.

Bash
MF_BUCKET=https://bucket.monadinfra.com
curl -o /home/monad/.env $MF_BUCKET/config/testnet/latest/.env.example
curl -o /home/monad/monad-bft/config/node.toml $MF_BUCKET/config/testnet/latest/full-node-node.toml
Generate & Backup Password (IMPORTANT):

Bash
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env

mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup
Create Keys (SECP & BLS):

Bash
bash <<'EOF'
set -e
source /home/monad/.env
monad-keystore create --key-type secp --keystore-path /home/monad/monad-bft/config/id-secp --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup
monad-keystore create --key-type bls --keystore-path /home/monad/monad-bft/config/id-bls --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup

# Display Public Address
grep "public key" /opt/monad/backup/secp-backup /opt/monad/backup/bls-backup | tee /home/monad/pubkey-secp-bls
EOF
📝 Step 5: Node Identity & Peer Discovery
Set your node name and generate the network signature.

1. Set Node Name: Replace YOUR_MONIKER with your desired node name.

Bash
sed -i 's/node_name = .*/node_name = "YOUR_MONIKER"/' /home/monad/monad-bft/config/node.toml
2. Generate Signature: Run the command below and add the output to node.toml under the [peer_discovery] section.

Bash
source /home/monad/.env
monad-sign-name-record \
  --address $(curl -s4 ifconfig.me):8000 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num 0
🔥 Step 6: Launch & Security
Finalize permissions, open firewall ports, and start services.

Bash
# Fix Ownership
chown -R monad:monad /home/monad/

# Configure Firewall (UFW)
ufw allow ssh
ufw allow 8000/tcp
ufw allow 8000/udp
# Optional: Allow 8080 if you are querying the EVM RPC externally
# ufw allow 8080/tcp 
ufw --force enable

# Enable & Start Services
systemctl enable monad-bft monad-execution monad-rpc
systemctl start monad-bft monad-execution monad-rpc
📊 Step 7: Monitoring & Watchdog
1. CLI Status Tool:
You can use the community status script to check sync status.

Bash
curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh -o /usr/local/bin/monad-status && chmod +x /usr/local/bin/monad-status
monad-status
2. View Logs:

Bash
journalctl -u monad-bft -f
# Or check execution logs:
journalctl -u monad-execution -f
