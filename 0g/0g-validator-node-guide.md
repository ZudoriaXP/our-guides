Update server:

```bash
sudo apt update
sudo apt install curl git jq build-essential gcc unzip wget lz4 -y
```

Install GO:

```bash
cd $HOME && \
ver="1.21.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

Install evmosd binary:

```bash
git clone -b v0.1.0 https://github.com/0glabs/0g-chain.git
./0g-chain/networks/testnet/install.sh
source .profile
```

Set variables (you can personalize them):

```bash
echo 'export MONIKER="type_your_moniker_nodebrand"' >> ~/.bash_profile
echo 'export CHAIN_ID="zgtendermint_16600-1"' >> ~/.bash_profile
echo 'export WALLET_NAME="wallet"' >> ~/.bash_profile
echo 'export RPC_PORT="26657"' >> ~/.bash_profile
source $HOME/.bash_profile
```

Initialize the node:

```bash
cd $HOME
0gchaind config chain-id $CHAIN_ID
0gchaind init $MONIKER --chain-id $CHAIN_ID
0gchaind config node tcp://localhost:$RPC_PORT
0gchaind config keyring-backend os
```

Install current release:

```bash
wget -P ~/.0gchain/config https://github.com/0glabs/0g-chain/releases/download/v0.1.0/genesis.json
```

Configure **config.toml** file:

```bash
PEERS="" && \
SEEDS="c4d619f6088cb0b24b4ab43a0510bf9251ab5d7f@54.241.167.190:26656,44d11d4ba92a01b520923f51632d2450984d5886@54.176.175.48:26656,f2693dd86766b5bf8fd6ab87e2e970d564d20aff@54.193.250.204:26656,f878d40c538c8c23653a5b70f615f8dccec6fb9f@54.215.187.94:26656" && \
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.0gchain/config/config.toml
```

Set the config:

```bash
EXTERNAL_IP=$(wget -qO- eth0.me) \
PROXY_APP_PORT=26658 \
P2P_PORT=26656 \
PPROF_PORT=6060 \
API_PORT=1317 \
GRPC_PORT=9090 \
GRPC_WEB_PORT=9091
```

Set the port pruning and price:

```bash
sed -i \
    -e "s/\(proxy_app = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$PROXY_APP_PORT\"/" \
    -e "s/\(laddr = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$RPC_PORT\"/" \
    -e "s/\(pprof_laddr = \"\)\([^:]*\):\([0-9]*\).*/\1localhost:$PPROF_PORT\"/" \
    -e "/\[p2p\]/,/^\[/{s/\(laddr = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$P2P_PORT\"/}" \
    -e "/\[p2p\]/,/^\[/{s/\(external_address = \"\)\([^:]*\):\([0-9]*\).*/\1${EXTERNAL_IP}:$P2P_PORT\"/; t; s/\(external_address = \"\).*/\1${EXTERNAL_IP}:$P2P_PORT\"/}" \
    $HOME/.0gchain/config/config.toml
```

```bash
sed -i \
    -e "/\[api\]/,/^\[/{s/\(address = \"tcp:\/\/\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$API_PORT\4/}" \
    -e "/\[grpc\]/,/^\[/{s/\(address = \"\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$GRPC_PORT\4/}" \
    -e "/\[grpc-web\]/,/^\[/{s/\(address = \"\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$GRPC_WEB_PORT\4/}" $HOME/.0gchain/config/app.toml
```

```bash
sed -i.bak -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.0gchain/config/app.toml
sed -i.bak -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.0gchain/config/app.toml
sed -i.bak -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" $HOME/.0gchain/config/app.toml
```

```bash
sed -i "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0ua0gi\"/" $HOME/.0gchain/config/app.toml
```

```bash
sed -i "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.0gchain/config/config.toml
```

Create a service file:

```bash
sudo tee /etc/systemd/system/ogd.service > /dev/null <<EOF
[Unit]
Description=OG Node
After=network.target

[Service]
User=$USER
Type=simple
ExecStart=$(which 0gchaind) start --home $HOME/.0gchain
Restart=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

Start node:

```bash
sudo systemctl daemon-reload && \
sudo systemctl enable ogd && \
sudo systemctl restart ogd && sudo journalctl -u ogd -f -o cat
```

Wait a bit until you get the logs. To get out of them press **Cntr+C.**

Install Spanshot, addrbook, update peers:

```bash
curl -Ls https://snapshots.zudoriaxp.net/snapshots/testnet/zero-gravity/addrbook.json > $HOME/.0gchain/config/addrbook.json
```

```bash
PEERS=$(curl -s --max-time 3 --retry 2 --retry-connrefused "https://snapshots.zudoriaxp.net/snapshots/testnet/zero-gravity/peers.txt")
if [ -z "$PEERS" ]; then
    echo "No peers were retrieved from the URL."
else
    echo -e "\nPEERS: "$PEERS""
    sed -i "s/^persistent_peers *=.*/persistent_peers = "$PEERS"/" "$HOME/.0gchain/config/config.toml"
    echo -e "\nConfiguration file updated successfully.\n"
fi
```

```bash
sudo systemctl stop ogd
cp $HOME/.0gchain/data/priv_validator_state.json $HOME/.0gchain/priv_validator_state.json.backup
rm -rf $HOME/.0gchain/data
curl -L http://snapshots.zudoriaxp.net/snapshots/testnet/zero-gravity/zgtendermint_16600-1_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.0gchain
mv $HOME/.0gchain/priv_validator_state.json.backup $HOME/.0gchain/data/priv_validator_state.json
sudo systemctl restart ogd && sudo journalctl -u ogd -f -o cat
```

Check node synchronization:

```bash
0gchaind status | jq .sync_info
```

If you get **false,** you can install the validator. If it is **true**, you have to wait. You can also compare the height of your node with explorer.

# **Create the validator:**

Create a wallet for our validator:

```bash
0gchaind keys add $WALLET_NAME --eth
```

You may also import an existing wallet:

```bash
0gchaind keys add --recover $WALLET_NAME --eth
```

Paste the password

We will be given our wallet with a seed — we save it in a safe place:

Request private key of the EVM address:

```bash
0gchaind keys unsafe-export-eth-key $WALLET_NAME
```

Copy address and import in Metamask.

Now, we go to the [faucet](https://faucet.0g.ai/) and request test tokens

Checking the balance in the terminal:

```bash
0gchaind q bank balances $(0gchaind keys show $WALLET_NAME -a)
```

### The faucet gives you 1000000000000000000aevmos. For a validator to join an active set, you need at least 10000000000000000000aevmos (10 times more)

- Create the validator (you may change identity, website and details):

```bash
0gchaind tx staking create-validator \
  --amount=1000000ua0gi \
  --pubkey=$(0gchaind tendermint show-validator) \
  --moniker="$MONIKER" \
  --chain-id=zgtendermint_16600-1 \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --details="Your Details" \
  --min-self-delegation="1" \
  --from=$WALLET_NAME \
  --gas=auto \
  --gas-adjustment=1.4
```

- Copy 0gvaloper address

We delegate tokens to ourselves:

```bash
0gchaind q staking validator $(0gchaind keys show $WALLET_NAME --bech val -a)
```

Delegating to another validator:

```bash
0gchaind tx staking delegate <validator address> --from <wallet> <amount>ua0gi --gas=auto --gas-adjustment=1.4 -y
```

Check transactions in Explorer.

# **More commands:**

Check logs:

```bash
sudo journalctl -u ogd -f -o cat
```

Check synchronization status:

```bash
0gchaind status | jq .sync_info
```

Restart the node:

```bash
sudo systemctl restart ogd
```
