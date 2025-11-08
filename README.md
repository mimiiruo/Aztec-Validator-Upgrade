# Aztec Validator Setup Guide v2.1.2

This guide walks you through setting up an Aztec validator node on the testnet using Docker.

## Quick Start: Automated Installation (Recommended for New Nodes)

If you're setting up a brand new validator node, you can use the automated Docker installer:

```bash
rm -rf autoinstall-aztec-docker.sh && \
wget https://raw.githubusercontent.com/ezlabsnodes/autoinstall/main/autoinstall-aztec-docker.sh && \
chmod +x autoinstall-aztec-docker.sh && \
sudo ./autoinstall-aztec-docker.sh
```

The automated installer will prompt you for:

- **ETHEREUM_RPC_URL**: Your L1 Execution Layer RPC endpoint
- **CONSENSUS_BEACON_URL**: Your L1 Consensus Layer Beacon RPC endpoint
- **VALIDATOR_PRIVATE_KEYS**: Your validator's private key (comma-separated if multiple)
- **COINBASE**: Your wallet address (Ethereum address for rewards)

**When to use the automated installer:**
- Fresh VPS setup with no existing Aztec installation
- Migrating to a new server
- Quick deployment without manual configuration

**When to use the manual process below:**
- Upgrading an existing validator node
- Migrating from an old version (e.g., 2.0.4 to 2.1.2)
- Custom configuration requirements
- Troubleshooting specific issues

---

## Manual Setup Process

If you need to upgrade an existing node or prefer manual control, follow the steps below.

### Prerequisites

- A Linux server with Docker and Docker Compose installed
- Root access or sudo privileges
- An existing Ethereum wallet with testnet tokens
- RPC endpoint access (both Aztec and Ethereum)

## Step 1: Clean Up Existing Installation

Navigate to your Aztec directory and reset the environment:

```bash
cd ~/aztec
docker compose down
rm -rf /root/.aztec/testnet/data
```

## Step 2: Update Docker Compose Configuration

Update the Aztec version and network settings:

```bash
sed -i 's|^ *image: aztecprotocol/aztec:.*|    image: aztecprotocol/aztec:2.1.2|' docker-compose.yml
sed -i 's|--network alpha-testnet|--network testnet|g' docker-compose.yml
docker compose pull
```

## Step 3: Install Aztec CLI Tools

Install the Aztec toolchain:

```bash
bash -i <(curl -s https://install.aztec.network)
```

Add Aztec to your PATH:

```bash
echo 'export PATH=$PATH:/root/.aztec/bin' >> ~/.bashrc
source ~/.bashrc
```

Verify installation and set version:

```bash
aztec-up -v 2.1.2
```

## Step 4: Install Foundry (Ethereum Development Tools)

Install Foundry for interacting with Ethereum contracts:

```bash
curl -L https://foundry.paradigm.xyz | bash
source /root/.bashrc
foundryup
```

## Step 5: Approve Token Allowance

Approve the Rollup contract to spend tokens from your existing wallet:

```bash
cast send 0x139d2a7a0881e16332d7D1F8DB383A4507E1Ea7A \
  "approve(address,uint256)" \
  0xebd99ff0ff6677205509ae73f93d0ca52ac85d67 \
  200000ether \
  --private-key YOUR_OLD_PRIVATE_KEY \
  --rpc-url YOUR_ETH_RPC_URL
```

**Replace:**
- `YOUR_OLD_PRIVATE_KEY` with your existing wallet's private key
- `YOUR_ETH_RPC_URL` with your Ethereum RPC endpoint

## Step 6: Verify Token Allowance

Confirm the approval was successful:

```bash
cast call 0x139d2a7a0881e16332d7D1F8DB383A4507E1Ea7A \
  "allowance(address,address)(uint256)" \
  YOUR_OLD_WALLET_ADDRESS \
  0xebd99ff0ff6677205509ae73f93d0ca52ac85d67 \
  --rpc-url YOUR_ETH_RPC_URL
```

**Replace:**
- `YOUR_OLD_WALLET_ADDRESS` with your Ethereum address
- `YOUR_ETH_RPC_URL` with your RPC endpoint

## Step 7: Generate New Validator Keys

Create a new set of validator keys:

```bash
aztec validator keys new \
  --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000 \
  --data-dir ~/.aztec/keystore \
  --file key1.json
```

## Step 8: Extract and Save Your Keys

Display your newly generated keys:

```bash
ETH_PK=$(jq -r '.validators[0].attester.eth' ~/.aztec/keystore/key1.json)
BLS_KEY=$(jq -r '.validators[0].attester.bls' ~/.aztec/keystore/key1.json)
ETH_ADDR=$(cast wallet address --private-key "$ETH_PK")

printf "================================================\n"
printf "NEW VALIDATOR KEYS - SAVE THESE SECURELY!\n"
printf "================================================\n"
printf "ETH Address     : %s\n" "$ETH_ADDR"
printf "ETH Private Key : %s\n" "$ETH_PK"
printf "BLS Key         : %s\n" "$BLS_KEY"
printf "================================================\n"
```

**⚠️ IMPORTANT:** Save these keys securely! You'll need them for the next step.

## Step 9: Register Validator on L1

Register your new validator keys with the Aztec rollup contract:

```bash
aztec add-l1-validator \
  --l1-rpc-urls YOUR_ETH_RPC_URL \
  --network testnet \
  --private-key YOUR_OLD_PRIVATE_KEY \
  --attester YOUR_NEW_ETH_ADDRESS \
  --withdrawer YOUR_NEW_ETH_ADDRESS \
  --bls-secret-key YOUR_BLS_SECRET_KEY \
  --rollup 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67
```

**Replace:**
- `YOUR_ETH_RPC_URL` with your Ethereum RPC endpoint
- `YOUR_OLD_PRIVATE_KEY` with your existing wallet's private key (used to pay gas)
- `YOUR_NEW_ETH_ADDRESS` with the ETH Address from Step 8
- `YOUR_BLS_SECRET_KEY` with the BLS Key from Step 8

## Step 10: Update Docker Compose for Full Sync

Modify the Docker Compose configuration to use full sync mode:

```bash
cd ~/aztec
sed -i 's|aztecprotocol/aztec:2.0.4|aztecprotocol/aztec:2.1.2|g' docker-compose.yml
sed -i 's|--snapshots-url https://snapshots.aztec.graphops.xyz/files|--sync-mode full|' docker-compose.yml
```

## Step 11: Update Environment Variables

Configure your validator credentials in the `.env` file:

```bash
cd ~/aztec
cp .env .env.backup

ETH_PK=$(jq -r '.validators[0].attester.eth' ~/.aztec/keystore/key1.json)
ETH_ADDR=$(cast wallet address --private-key "$ETH_PK")

sed -i "s|^VALIDATOR_PRIVATE_KEYS=.*|VALIDATOR_PRIVATE_KEYS=$ETH_PK|" .env
sed -i "s|^COINBASE=.*|COINBASE=$ETH_ADDR|" .env

echo "✅ .env updated!"
cat .env
```

This backs up your existing `.env` file and updates it with your new validator credentials.

## Step 12: Start Your Validator Node

Launch the validator node:

```bash
cd ~/aztec
docker compose up -d
```

## Step 13: Monitor Your Validator

Check the logs to ensure everything is running correctly:

```bash
docker compose logs -f
```

Press `Ctrl+C` to exit the logs view.
