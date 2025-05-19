# ⚙️ Geth + Prysm Sepolia Node Setup (Google Cloud, Mounted Disk) — by cryptoprofezor

This step-by-step guide will take you from zero to a fully functional **Ethereum Sepolia Node** (Execution + Consensus Layer) using **Geth** and **Prysm**, with mounted disk setup on Google Cloud. All commands are **fully tested**, clean, and copy-paste ready from your **home directory**.

---

## 🖥️ System Requirements (Recommended for Smooth Syncing)

| Component | Recommended Specs     |
| --------- | --------------------- |
| CPU       | 4 vCPU or higher      |
| RAM       | 16 GB                 |
| SSD       | 1 TB (Mounted Disk)   |
| Bandwidth | 20–50 Mbps up/down    |
| OS        | Ubuntu 20.04 or 22.04 |
| Access    | Root / sudo access    |

---

## 🔧 Pre-Requirements

* ✅ Google Cloud VM with mounted disk (e.g., `/mnt/eth-data`)
* ✅ Docker + Docker Compose installed
* ✅ 1TB attached disk formatted & mounted

---

## 🛠 Step-by-Step Installation

### 1. 📦 Install Dependencies (All Required Packages)

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano \
  automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev \
  libleveldb-dev tar clang bsdmainutils ncdu unzip -y
```

### 2. 🐳 Install Docker and Docker Compose

```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo systemctl enable docker && sudo systemctl start docker
```

---

### 3. 💽 Format and Mount Your Attached Disk (Optional if already mounted)

```bash
# Format the new disk (replace /dev/sdb with your actual disk if different)
sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

# Create a mount point
dudo mkdir -p /mnt/eth-data

# Mount the disk
sudo mount -o discard,defaults /dev/sdb /mnt/eth-data

# Add to /etc/fstab for auto-mount on reboot
echo '/dev/sdb /mnt/eth-data ext4 discard,defaults,nofail 0 2' | sudo tee -a /etc/fstab

# Confirm mount
lsblk
```

> ⚠️ If unsure about disk name, run `lsblk` before starting.

> ⚠️ Update `/etc/fstab` to auto-mount on reboot.

---

### 4. 📁 Create Directories and JWT Token

```bash
sudo mkdir -p /mnt/eth-data/ethereum/execution
sudo mkdir -p /mnt/eth-data/ethereum/consensus
cd ~
openssl rand -hex 32 > jwt.hex
```

---

### 5. 📝 Create docker-compose.yml

```bash
nano docker-compose.yml
```

Paste this config:

```yaml
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    network_mode: host
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /mnt/eth-data/ethereum/execution:/data
      - ./jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.api=eth,net,web3
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain
    container_name: prysm
    network_mode: host
    restart: unless-stopped
    volumes:
      - /mnt/eth-data/ethereum/consensus:/data
      - ./jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
    command:
      - --sepolia
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=0.0.0.0
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --min-sync-peers=3
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

### 6. 🚀 Start the Node

```bash
sudo docker-compose up -d
```

Verify containers:

```bash
docker ps
```

---

### 7. 🔍 Check Syncing Status

**Geth (Execution Layer)**

```bash
curl http://localhost:8545 -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
```

**Prysm (Consensus Layer)**

```bash
curl http://localhost:3500/eth/v1/node/syncing | jq
```

---

### 8. 🌐 RPC Endpoints

* **Execution:** `http://<your-vm-ip>:8545`
* **Beacon API:** `http://<your-vm-ip>:3500`

Use these in Aztec, DApps, dashboards, etc.

---

### 9. 🔐 Open Firewall Ports (GCP)

> 📌 **No need to install or configure UFW/firewalld inside VPS**. Just set these from your **Google Cloud Console or CLI**.

Use the following commands to allow all necessary ports via `gcloud` CLI:

```bash
gcloud compute firewall-rules create allow-geth-p2p \
  --direction=INGRESS --priority=1000 --network=default --action=ALLOW \
  --rules=tcp:30303,udp:30303 --source-ranges=0.0.0.0/0 --target-tags=geth-node

gcloud compute firewall-rules create allow-geth-p2p-egress \
  --direction=EGRESS --priority=1000 --network=default --action=ALLOW \
  --rules=tcp:30303,udp:30303 --destination-ranges=0.0.0.0/0 --target-tags=geth-node

gcloud compute firewall-rules create allow-prysm-p2p \
  --direction=INGRESS --priority=1000 --network=default --action=ALLOW \
  --rules=tcp:13000,udp:13000 --source-ranges=0.0.0.0/0 --target-tags=geth-node

gcloud compute firewall-rules create allow-prysm-p2p-egress \
  --direction=EGRESS --priority=1000 --network=default --action=ALLOW \
  --rules=tcp:13000,udp:13000 --destination-ranges=0.0.0.0/0 --target-tags=geth-node

gcloud compute firewall-rules create allow-geth-rpc \
  --direction=INGRESS --priority=1000 --network=default --action=ALLOW \
  --rules=tcp:8545,tcp:8551 --source-ranges=0.0.0.0/0 --target-tags=geth-node

gcloud compute firewall-rules create allow-prysm-rest \
  --direction=INGRESS --priority=1000 --network=default --action=ALLOW \
  --rules=tcp:3500 --source-ranges=0.0.0.0/0 --target-tags=geth-node
```

> 🏷️ Don't forget to apply the tag `geth-node` to your VM instance in GCP.

---

### 10. 🔁 Manage / Restart / Logs

```bash
sudo docker-compose down        # Stop
sudo docker-compose up -d       # Restart

docker logs -f geth              # Geth Logs
docker logs -f prysm             # Prysm Logs
```

---

### ✅ Final Words

Congrats! You now have a fully synced, persistent Sepolia node using your mounted SSD storage — perfect for Aztec testnet, validator setups, and custom infra! 🚀
