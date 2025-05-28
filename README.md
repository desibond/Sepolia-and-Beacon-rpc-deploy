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
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev  -y
```

### 2. 🐳 Install Docker and Docker Compose

```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```

### Verify Installation

```bash
docker --version
```
```bash
docker compose version
```

---

### 3. 💽 Format and Mount Your Attached Disk (Only Once When You Create VM)

Follow these steps exactly to mount your external 1TB SSD in GCP:

```bash
# STEP 1: Find your external disk name (usually /dev/sdb)
lsblk
```

✅ You will see output like:

```
sda     10G  disk  (main system disk)
sdb    1000G disk  <-- This is your attached disk
```

If it's `/dev/sdb`, continue:

```bash
sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
sudo mkdir -p /mnt/eth-data
sudo mount -o discard,defaults /dev/sdb /mnt/eth-data
echo '/dev/sdb /mnt/eth-data ext4 discard,defaults,nofail 0 2' | sudo tee -a /etc/fstab
lsblk
```

> ✅ After reboot, your disk will always auto-mount to `/mnt/eth-data`
> ❌ DO NOT repeat this formatting step — it will erase all blockchain data!

> ⚠️ If unsure about disk name, run `lsblk` before starting.

> ⚠️ Update `/etc/fstab` to auto-mount on reboot.

---

### 4. 📁 Create Directories and JWT Token

```bash
sudo mkdir -p /mnt/eth-data/execution
sudo mkdir -p /mnt/eth-data/consensus
sudo openssl rand -hex 32 | sudo tee /mnt/eth-data/jwt.hex
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
      - /mnt/eth-data/execution:/data
      - /mnt/eth-data/jwt.hex:/data/jwt.hex
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
      - /mnt/eth-data/consensus:/data
      - /mnt/eth-data/jwt.hex:/data/jwt.hex
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
sudo docker compose up -d 
```

Check Logs

```bash
sudo docker compose logs -f
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
curl http://localhost:3500/eth/v1/node/syncing
```

---

### 8. 🌐 RPC Endpoints

## Sepolia RPC ( In Other VPS )
 
```bash
http://<your-vm-ip>:8545
```

## Beacon RPC 

```bash
http://<your-vm-ip>:3500
```

Use these in Aztec Sequencer Node ( Make Sure To Update Your RPC VPS IP On It Before Using It 

---

### 9. 🔐 Open Firewall Ports (GCP)

> 📌 **No need to install or configure UFW/firewalld inside VPS**. Just set these from your **Google Cloud Console or CLI**.

### Login gcloud in Terminal

```bash
gcloud auth login
```

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

### 10. 🔁 Stop / Restart 

```bash
sudo docker compose down
sudo docker compose up -d
```

### Check Logs

```bash
sudo docker compose logs -f
```

### ⚠️ Important Update (Must Read)

If you're running your own **Geth + Prysm node**, make sure to use **`http://`** (not `https://`) in your Aztec start command.

> 🛠 Use `https://` **only when using third-party RPC providers** (like Alchemy, Nodereal, etc.)  
> 🧑‍💻 Use `http://` **when connecting to your own local or VPS-hosted Geth/Prysm nodes**

---

### ✅ Correct Aztec Start Command for Self-Hosted RPC:

```bash
aztec start --node --archiver --sequencer \
  --network alpha-testnet \
  --l1-rpc-urls http://<your-vm-ip>:8545 \
  --l1-consensus-host-urls http://<your-vm-ip>:3500 \
  --sequencer.validatorPrivateKey 0xYOUR_PRIVATE_KEY \
  --sequencer.coinbase 0xYOUR_PUBLIC_ADDRESS \
  --p2p.p2pIp <your-vm-ip>
```

> ⚠️ Using `https://` with self-hosted nodes will silently fail and throw errors like `Rollup__InvalidManaBaseFee`, `Rollup__InvalidTimestamp`, `Rollup__InvalidProof`, etc.

---

### ✅ Final Words

Congrats! You now have your own fully synced Sepolia RPC running on Google Cloud 💻  
Perfect for Aztec testnet, validator setups, or your own infra needs 🚀

---

📌 **Follow for more updates:**

- 🔹 GitHub → [@cryptoprofezor](https://github.com/cryptoprofezor)  
- 🔹 Telegram → [@MrCryptoTamilan](https://t.me/MrCrypto_Tamilan)

No VPS? No time to run RPC node? Chill bro. I’ll give you Sepolia + Beacon RPC valid for 1 month – budget-friendly. DM if needed

---

More guides coming soon. Stay tuned 🔧🔥
