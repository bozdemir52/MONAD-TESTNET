## Monad Mainnet/Testnet Node Setup & Optimization Guide
This guide covers the installation steps and system optimizations for a Monad Node on Ubuntu 24.04 using a high-performance NVMe SSD (TrieDB).

# 🛠️ System Requirements
OS: Ubuntu 24.04 LTS (Kernel v6.8.0.60+ is Mandatory)

CPU: 16 Core+ (4.5GHz+ frequency recommended)

RAM: 32 GB+ (64GB-128GB recommended for Mainnet)

Disk: 2 x 2TB NVMe SSD (INDEPENDENT/NO RAID IS MANDATORY. One for OS/BFT, the other for TrieDB)

Network: 1 Gbps+ (Capable of 70,000 PPS)

# 🚨 STEP 0: Critical System Check (Fatal Kernel Bug)
Linux kernel versions between v6.8.0.56 and v6.8.0.59 contain a fatal bug that puts the Monad node into deep sleep and locks it. Check your version before proceeding:

```Bash
uname -r
```
If your version is 56, 57, 58, or 59, you must update the system and reboot before continuing:

```Bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```
(Verify that the version is v6.8.0.60 or higher after rebooting.)

# 🚀 STEP 1: Basic Packages & Preparation
```Bash
# Install system tools
sudo apt install -y curl nvme-cli aria2 jq parted ufw linux-tools-common linux-tools-$(uname -r)

# Add Monad Repository and GPG Key
sudo curl -fsSL https://pkg.category.xyz/keys/public-key.asc | sudo gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg

sudo cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

# Install Monad
sudo apt update
sudo apt install -y monad
sudo apt-mark hold monad

# Create User and Directory Structure
sudo useradd -m -s /bin/bash monad
sudo mkdir -p /home/monad/monad-bft/config /home/monad/monad-bft/ledger /home/monad/monad-bft/config/forkpoint /home/monad/monad-bft/config/validators
```
# ⚙️ STEP 2: Hardware Optimization (CPU & Performance)
To ensure Monad runs without latency, we take the processor out of powersave mode and lock it into "Performance" mode:

```Bash
# Lock the processor to maximum performance mode
sudo cpupower frequency-set -g performance

# Check (The Governor section should display "performance")
cpupower frequency-info | grep "current policy"
```
# 💾 STEP 3: Disk Configuration (TrieDB - Independent NVMe)
Important: Ensure your disks are NOT configured in RAID. We will use a separate physical disk (/dev/nvme1n1) exclusively for TrieDB.

```Bash
# Define Disk Variable (Check according to your own disk: lsblk)
export TRIEDB_DRIVE=/dev/nvme1n1

# Partition the Disk (GPT)
sudo parted -s $TRIEDB_DRIVE mklabel gpt
sudo parted -s $TRIEDB_DRIVE mkpart triedb 0% 100%

# Create Udev Rule for I/O Performance
PARTUUID=$(lsblk -o PARTUUID $TRIEDB_DRIVE | tail -n 1)
echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" | sudo tee /etc/udev/rules.d/99-triedb.rules

# Apply Rules
sudo udevadm trigger
sudo udevadm control --reload
sudo udevadm settle

# Start TrieDB Service
sudo systemctl start monad-mpt
```
# 🔑 STEP 4: Configuration & Wallet Creation
```Bash
# Download configuration files
MF_BUCKET=https://bucket.monadinfra.com
curl -o /home/monad/.env $MF_BUCKET/config/testnet/latest/.env.example
curl -o /home/monad/monad-bft/config/node.toml $MF_BUCKET/config/testnet/latest/full-node-node.toml

# Generate and Backup Password
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env
sudo mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup

# Create Keystore (Wallet)
monad-keystore create --key-type secp --keystore-path /home/monad/monad-bft/config/id-secp --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup
monad-keystore create --key-type bls --keystore-path /home/monad/monad-bft/config/id-bls --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup
```
# 🛡️ STEP 5: Firewall & Network Settings (RaptorCast Warning)
⚠️ ATTENTION: Monad generates massive UDP traffic (up to ~70,000 PPS). You MUST disable or highly relax the standard anti-DDoS protections on your server provider's external control panel (e.g., Hetzner, Latitude). Otherwise, they may mistake legitimate Monad traffic for an attack and null-route your server.

```Bash
# Allow SSH and Monad Ports
sudo ufw allow ssh
sudo ufw allow 8000/tcp
sudo ufw allow 8000/udp
sudo ufw --force enable

# Adjust Ownership Permissions
sudo chown -R monad:monad /home/monad/

# Start Services
sudo systemctl enable monad-bft monad-execution monad-rpc
sudo systemctl start monad-bft monad-execution monad-rpc
```
# 📊 Monitoring
To check the node status:

```Bash
# Follow logs
journalctl -u monad-bft -f

# Official status script
curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh -o /usr/local/bin/monad-status && chmod +x /usr/local/bin/monad-status
monad-status
```
## 🔄 Troubleshooting: Manual Soft Reset (Sync Issues)

A soft reset is highly recommended if your node is out of sync, the `blockDifference` keeps increasing, or `statesync` is permanently stuck at `0.0000%`. 

This process fetches the latest `forkpoint` and `validators` from the network, allowing your node to skip ahead and sync instantly without losing your keys or wiping your database.

**1. Stop Monad Services**
First, stop the running services to safely update the configuration files.
```bash
sudo systemctl stop monad-bft monad-execution monad-rpc
```
2. Fetch the Latest Testnet Configurations
Run the following block to download the latest forkpoint.toml and validators.toml files from the official Monad infrastructure bucket.

```Bash
MF_BUCKET=[https://bucket.monadinfra.com](https://bucket.monadinfra.com)
VALIDATORS_FILE=/home/monad/monad-bft/config/validators/validators.toml

curl -sSL $MF_BUCKET/scripts/testnet/download-forkpoint.sh | sudo bash
sudo curl $MF_BUCKET/validators/testnet/validators.toml -o $VALIDATORS_FILE
sudo chown monad:monad $VALIDATORS_FILE
```
3. Restart Monad Services
Once the fresh configuration files are in place, start the services again.

```Bash
sudo systemctl start monad-bft monad-execution monad-rpc
```
4. Verify Node Status
Check if all systemd services are actively running:

```Bash
sudo systemctl list-units --type=service monad-bft.service monad-execution.service monad-rpc.service
```
Finally, monitor your node's status. It might take a minute, but your node should transition to status: in-sync and blockDifference: 0.

```Bash
watch -n 2 monad-status
```
