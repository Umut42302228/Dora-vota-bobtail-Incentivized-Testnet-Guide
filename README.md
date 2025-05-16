# Dora Vota Bobtail Testnet Full Node and Validator Setup

This guide provides a complete setup for a Dora Vota Bobtail testnet (`vota-bobtail`) node, including dependencies, building `dorad`, State Sync, wallet creation, validator setup, and reward withdrawal. Copy the entire code block below to run all steps sequentially.


# Step 1: Install Dependencies
````
sudo apt update && sudo apt upgrade -y
sudo apt install -y git build-essential curl jq wget liblz4-tool
sudo apt install golang jq git cmake gcc make
````
# Step 2: Install Go
````
wget https://go.dev/dl/go1.23.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.23.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
go version
````

# Step 3: Build dorad (write your own "MONIKER" name)
````
export VERSION=0.4.2
export CHAIN_ID="vota-bobtail"
export MONIKER="Umut|nodes"
git clone https://github.com/DoraFactory/doravota.git
cd doravota
git checkout <version tag>
make install
sudo cp ~/go/bin/dorad /usr/local/bin
````

# Step 4: Initialize Node
````
dorad init my-node --chain-id vota-bobtail --home ~/.dora
````
# Step 5: Configure Genesis File
````
wget -O ~/.dora/config/genesis.json https://df-node-dump.s3.ap-southeast-1.amazonaws.com/vota-bobtail-genesis.json
````
# Step 6: Set Persistent Peers
````
# Replace "peer1,peer2" with valid peers from https://t.me/DoraVotaOfficial
sed -i 's|^persistent_peers =.*|persistent_peers = "peer1,peer2"|' ~/.dora/config/config.toml
````
# Step 7: Create Wallet
````
dorad keys add my-wallet --home ~/.dora
# Save the mnemonic phrase securely
dorad keys show my-wallet -a --home ~/.dora
````
# Step 8: Fund Wallet
# Check balance (replace <WALLET_ADDRESS> with your wallet address)
````
dorad query bank balances <WALLET_ADDRESS> --node https://rpc-vota-bobtail.dorafactory.org:443
````
# Step 9: Create Systemd Service
````
sudo tee /etc/systemd/system/dorad.service > /dev/null <<EOF
[Unit]
Description=Dorad-service
After=network-online.target

[Service]
User=root
ExecStart=/usr/local/bin/dorad start --home /root/.dora
Restart=always
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable dorad
sudo systemctl daemon-reload
````
# Step 10: Configure State Sync
````
sudo systemctl stop dorad
dorad tendermint unsafe-reset-all --home ~/.dora --keep-addr-book
SNAP_RPC="https://rpc-vota-bobtail.dorafactory.org:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,https://rpc-vota-bobtail.dorafactory.org:443\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(trust_period[[:space:]]+=[[:space:]]+).*$|\1\"168h0m0s\"| ; \
s|^(discovery_time[[:space:]]+=[[:space:]]+).*$|\1\"5s\"|" $HOME/.dora/config/config.toml
grep -E 'rpc_servers|trust_height|trust_hash|trust_period|discovery_time' $HOME/.dora/config/config.toml
sudo systemctl start dorad
curl -s http://localhost:26657/status | jq .result.sync_info
````
# Wait until catching_up: false


# Step 11: Create Validator
# Ensure node is synced before running
````
dorad tx staking create-validator \
  --from=my-wallet \
  --chain-id=vota-bobtail \
  --amount=1000000udora \
  --pubkey=$(dorad tendermint show-validator --home ~/.dora) \
  --moniker=my-validator \
  --commission-rate=0.10 \
  --commission-max-rate=0.20 \
  --commission-max-change-rate=0.01 \
  --min-self-delegation=1 \
  --gas-prices=100000peaka \
  --gas=auto \
  --gas-adjustment=1.4 \
  --node=https://rpc-vota-bobtail.dorafactory.org:443 \
  -y
````
# Step 12: Verify Validator
# Replace <VALIDATOR_ADDRESS> with your doravaloper address
````
dorad query staking validator <VALIDATOR_ADDRESS> --node https://rpc-vota-bobtail.dorafactory.org:443
````
# Step 13: Check Rewards
# Replace <WALLET_ADDRESS> and <VALIDATOR_ADDRESS>
````
dorad query distribution rewards <WALLET_ADDRESS> <VALIDATOR_ADDRESS> --node https://rpc-vota-bobtail.dorafactory.org:443
````
# Step 14: Withdraw Rewards
# Replace <VALIDATOR_ADDRESS>
````
dorad tx distribution withdraw-rewards <VALIDATOR_ADDRESS> \
  --from=my-wallet \
  --commission \
  --chain-id=vota-bobtail \
  --gas-prices=100000peaka \
  --gas=auto \
  --gas-adjustment=1.4 \
  --node=https://rpc-vota-bobtail.dorafactory.org:443 \
  -y
````
# Step 15: Verify Balance
# Replace <WALLET_ADDRESS>
````
dorad query bank balances <WALLET_ADDRESS> --node https://rpc-vota-bobtail.dorafactory.org:443
````
