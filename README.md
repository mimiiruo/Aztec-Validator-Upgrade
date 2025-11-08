# Aztec Sequencer Node Update Guide - Version 2.1.2

A comprehensive guide to update your Aztec sequencer node to version 2.1.2.

## Overview

This guide will walk you through updating your Aztec sequencer node from a previous version to version 2.1.2. The process involves generating new validator keys, registering them on-chain, and updating your node configuration.

## Prerequisites

Before starting, ensure you have:

- Root access to your server
- Your old sequencer private key
- Ethereum Sepolia RPC URL
- At least 1 Sepolia ETH for gas fees
- Basic familiarity with command line operations

**Estimated Time:** 30-45 minutes (plus sync time)

## Table of Contents

1. [Stop and Clean Old Node](#step-1-stop-and-clean-old-node)
2. [Update Aztec CLI](#step-2-update-aztec-to-212)
3. [Install Foundry](#step-3-install-foundry)
4. [Approve Stake Token](#step-4-approve-stake-token)
5. [Generate New Validator Keys](#step-5-generate-new-validator-keys)
6. [Extract and Save Keys](#step-6-extract-and-save-keys)
7. [Register Validator](#step-7-register-validator)
8. [Update Docker Compose](#step-8-update-docker-compose)
9. [Update Environment Variables](#step-9-update-environment-variables)
10. [Start Updated Node](#step-10-start-updated-node)

---

## Step 1: Stop and Clean Old Node

First, stop your existing sequencer node and clean the old data directory:

```bash
docker stop aztec-sequencer && docker rm aztec-sequencer
sudo rm -rf /root/.aztec/testnet/data/
```

---

## Step 2: Update Aztec to 2.1.2

Update the Aztec CLI to version 2.1.2:

```bash
aztec-up -v 2.1.2
```

---

## Step 3: Install Foundry

Install Foundry (required for interacting with Ethereum contracts):

```bash
curl -L https://foundry.paradigm.xyz | bash
source /root/.bashrc
foundryup
```

---

## Step 4: Approve Stake Token

Approve the rollup contract to spend your stake tokens:

```bash
cast send 0x139d2a7a0881e16332d7D1F8DB383A4507E1Ea7A \
  "approve(address,uint256)" \
  0xebd99ff0ff6677205509ae73f93d0ca52ac85d67 \
  200000ether \
  --private-key "YOUR_OLD_SEQUENCER_PRIVATE_KEY" \
  --rpc-url YOUR_ETH_RPC_URL
```

**Replace:**
- `YOUR_OLD_SEQUENCER_PRIVATE_KEY` with your old sequencer private key
- `YOUR_ETH_RPC_URL` with your Ethereum Sepolia RPC endpoint

---

## Step 5: Generate New Validator Keys

Generate new validator keys for version 2.1.2:

```bash
aztec validator-keys new \
  --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000 \
  --data-dir ~/.aztec/keystore \
  --file key1.json
```

---

## Step 6: Extract and Save Keys

Extract and display your new validator keys:

```bash
ETH_PK=$(jq -r '.validators[0].attester.eth' ~/.aztec/keystore/key1.json); \
BLS_KEY=$(jq -r '.validators[0].attester.bls' ~/.aztec/keystore/key1.json); \
ETH_ADDR=$(cast wallet address --private-key "$ETH_PK"); \
printf "================================================\n"; \
printf "NEW VALIDATOR KEYS - SAVE THESE SECURELY!\n"; \
printf "================================================\n"; \
printf "ETH Address     : %s\n" "$ETH_ADDR"; \
printf "ETH Private Key : %s\n" "$ETH_PK"; \
printf "BLS Key         : %s\n" "$BLS_KEY"; \
printf "================================================\n"
```

### ⚠️ CRITICAL: Save Your Keys

**You must save all three values securely:**
- ETH Address
- ETH Private Key
- BLS Key

**Fund the ETH Address with at least 1 Sepolia ETH before continuing.**

---

## Step 7: Register Validator

Register your new validator on the L1 rollup contract:

```bash
aztec add-l1-validator \
  --l1-rpc-urls "YOUR_ETH_RPC_URL" \
  --network testnet \
  --private-key "YOUR_OLD_SEQUENCER_PRIVATE_KEY" \
  --attester "YOUR_NEW_ETH_ADDRESS" \
  --withdrawer "YOUR_NEW_ETH_ADDRESS" \
  --bls-secret-key "YOUR_BLS_KEY" \
  --rollup 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67
```

**Replace all placeholder values:**
- `YOUR_ETH_RPC_URL` with your RPC endpoint
- `YOUR_OLD_SEQUENCER_PRIVATE_KEY` with your old sequencer key
- `YOUR_NEW_ETH_ADDRESS` with the ETH Address from Step 6
- `YOUR_BLS_KEY` with the BLS Key from Step 6

---

## Step 8: Update Docker Compose

Update your Docker Compose configuration to use version 2.1.2 and full sync mode:

```bash
cd ~/aztec && \
sed -i 's|aztecprotocol/aztec:2.0.4|aztecprotocol/aztec:2.1.2|g' docker-compose.yml && \
sed -i 's|--snapshots-url https://snapshots.aztec.graphops.xyz/files|--sync-mode full|' docker-compose.yml
```

---

## Step 9: Update Environment Variables

Update your environment configuration with the new validator keys:

```bash
cd ~/aztec && \
cp .env .env.backup && \
ETH_PK=$(jq -r '.validators[0].attester.eth' ~/.aztec/keystore/key1.json) && \
ETH_ADDR=$(cast wallet address --private-key "$ETH_PK") && \
sed -i "s|^VALIDATOR_PRIVATE_KEYS=.*|VALIDATOR_PRIVATE_KEYS=$ETH_PK|" .env && \
sed -i "s|^COINBASE=.*|COINBASE=$ETH_ADDR|" .env && \
echo "✅ .env updated!" && cat .env
```

This command automatically:
- Creates a backup of your existing `.env` file
- Updates the `VALIDATOR_PRIVATE_KEYS` variable
- Updates the `COINBASE` variable
- Displays the updated configuration

---

## Step 10: Start Updated Node

Start your updated node:

```bash
cd ~/aztec && docker compose up -d
```

### View Logs

Monitor your node's logs to ensure it's running correctly:

```bash
docker logs -f aztec-sequencer
```

Press `Ctrl+C` to exit the log viewer.

---

## Verification

Your node is now running version 2.1.2 and will begin syncing with the network. You can verify the update by checking:

1. The Docker container is running: `docker ps`
2. The logs show no errors: `docker logs aztec-sequencer`
3. The node is syncing blocks

---

## Troubleshooting

### Node Won't Start

- Check Docker logs for error messages
- Verify all environment variables are set correctly
- Ensure your ETH address has sufficient Sepolia ETH

### Sync Issues

- Verify your RPC endpoint is working
- Check network connectivity
- Ensure you're using full sync mode

### Key Generation Errors

- Verify `jq` is installed: `sudo apt-get install jq`
- Check that `key1.json` exists in `~/.aztec/keystore/`

---

## Important Notes

- **Keep your validator keys secure** - never share them publicly
- **Backup your `.env` file** - a backup is automatically created as `.env.backup`
- **Monitor your node** - regularly check logs for any issues
- **Maintain sufficient ETH** - ensure your validator address always has enough Sepolia ETH for gas

---

## Support

For additional help:
- [Aztec Documentation](https://docs.aztec.network/)
- [Aztec Discord](https://discord.gg/aztec)
- [GitHub Issues](https://github.com/AztecProtocol/aztec-packages/issues)

---

## License

This guide is provided as-is for informational purposes. Always refer to official Aztec documentation for the most up-to-date information.
