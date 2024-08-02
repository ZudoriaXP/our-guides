### **Recommended Hardware:**

4 Cores, 8GB RAM, 200GB of storage (NVME)

# Installation

```bash
# install dependencies, if needed
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y

# install go, if needed
cd $HOME
VER="1.21.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
 
# set vars
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export WARDEN_CHAIN_ID="buenavista-1"" >> $HOME/.bash_profile
echo "export WARDEN_PORT="18"" >> $HOME/.bash_profile
source $HOME/.bash_profile
 
# download binary
cd $HOME
rm -rf wardenprotocol
git clone --depth 1 --branch v0.3.0 https://github.com/warden-protocol/wardenprotocol/
cd wardenprotocol
make install
 
# config and init app
wardend init $MONIKER
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${WARDEN_PORT}657\"|" $HOME/.warden/config/client.toml
 
# download genesis and addrbook
wget -O $HOME/.warden/config/genesis.json https://snapshot.zudoriaxp.net/warden-testnet/genesis.json
wget -O $HOME/.warden/config/addrbook.json https://snapshot.zudoriaxp.net/warden-testnet/addrbook.json
 
# set seeds and peers
SEEDS="bda08962882048fea4331fcf96ad02789671700e@warden-testnet-peer.zudoriaxp.net:35656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.warden/config/config.toml
 
# set custom ports in app.toml
sed -i.bak -e "s|:1317|:${WARDEN_PORT}317|g;
s|:8080|:${WARDEN_PORT}080|g;
s|:9090|:${WARDEN_PORT}090|g;
s|:9091|:${WARDEN_PORT}091|g;
s|:8545|:${WARDEN_PORT}545|g;
s|:8546|:${WARDEN_PORT}546|g;
s|:6065|:${WARDEN_PORT}065|g" $HOME/.warden/config/app.toml
 
# set custom ports in config.toml file
sed -i.bak -e "s|:26658|:${WARDEN_PORT}658|g;
s|:26657|:${WARDEN_PORT}657|g;
s|:6060|:${WARDEN_PORT}060|g;
s|:26656|:${WARDEN_PORT}656|g;
s|^external_address = \"\"|external_address = \"$(wget -qO- eth0.me):${WARDEN_PORT}656\"|g;
s|:26660|:${WARDEN_PORT}660|g" $HOME/.warden/config/config.toml
 
# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.warden/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.warden/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.warden/config/app.toml
 
# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0025uward"|g' $HOME/.warden/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.warden/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.warden/config/config.toml
 
# create service file
sudo tee /etc/systemd/system/wardend.service > /dev/null <<EOF
[Unit]
Description=Warden node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.warden
ExecStart=$(which wardend) start --home $HOME/.warden
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
 
# reset and download snapshot
wardend tendermint unsafe-reset-all --home $HOME/.warden
 
# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable wardend
sudo systemctl restart wardend && sudo journalctl -u wardend -f
```

# Wallet

### **Create a wallet**

```bash
wardend keys add $WALLET
```

### **To restore your keys you have created, do this**

```bash
wardend keys add $WALLET --recover
```

### **Save wallet and validator address**

```bash
WALLET_ADDRESS=$(wardend keys show $WALLET -a)
VALOPER_ADDRESS=$(wardend keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### **Check sync status, once your node is fully synced, the output from above will print "false"**

```bash
wardend status 2>&1 | jq
```

### **After creating a validator, make sure the fund has been funded sucessfully**

```bash
wardend query bank balances $WALLET_ADDRESS
```

# Validator

To create a validator, you should import the data of your validator to the **validator.json** file

first of all, you have to execute this to see `validator-pub-key`

```bash
wardendd comet show-validator
```

then, you can see the result such a following output below
```json
{"@type":"/cosmos.crypto.ed25519.PubKey","key":"xxxx"}
```

```bash
sudo tee $HOME/.warden/validator.json > /dev/null <<EOF
{
	"pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"xxxx"},
	"amount": "1000000uward",
	"moniker": "<validator-name>",
	"identity": "optional identity signature (ex. UPort or Keybase)",
	"website": "validator's (optional) website",
	"security": "validator's (optional) security contact email",
	"details": "validator's (optional) details",
	"commission-rate": "0.1",
	"commission-max-rate": "0.2",
	"commission-max-change-rate": "0.01",
	"min-self-delegation": "1"
}
EOF
 
# Create a validator using the JSON configuration
wardend tx staking create-validator $HOME/.warden/validator.json \
    --from $WALLET \
    --chain-id buenavista-1 \
	--gas auto --gas-adjustment 1.5 --fees 600uward
```
