Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.20.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export PRYZM_CHAIN_ID="indigo-1"" >> $HOME/.bash_profile
echo "export PRYZM_PORT="41"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget https://storage.googleapis.com/pryzm-zone/core/0.18.0/pryzmd-0.18.0-linux-amd64.tar.gz
tar -xzvf $HOME/pryzmd-0.18.0-linux-amd64.tar.gz
mv pryzmd $HOME/go/bin
```

**config and init app**
```
pryzmd config node tcp://localhost:${PRYZM_PORT}657
pryzmd config keyring-backend os
pryzmd config chain-id indigo-1
pryzmd init "test" --chain-id indigo-1
```

**download genesis and addrbook**
```
wget -O $HOME/.pryzm/config/genesis.json https://server-4.itrocket.net/testnet/pryzm/genesis.json
wget -O $HOME/.pryzm/config/addrbook.json  https://server-4.itrocket.net/testnet/pryzm/addrbook.json
```

**set seeds and peers**
```
SEEDS="fbfd48af73cd1f6de7f9102a0086ac63f46fb911@pryzm-testnet-seed.itrocket.net:41656"
PEERS="713307ce72306d9e86b436fc69a03a0ab96b678f@pryzm-testnet-peer.itrocket.net:41656,089fc2d0d012c2417f9eb8e176f4a7811027050c@213.199.45.120:41656,486c8e5c2f128cc6424773891b8bfa2b02890495@194.163.137.83:23256,b5cbda80baeb1f9ad91e5246dc3715ef48caeff6@185.119.116.228:26656,a9c9f21f4519fd1cce4b4ddd356b5c47e6c24386@81.1.13.238:23256,013efc1bb66c696aada395019e8cdf57f5ccc106@85.10.211.215:27722,5d00ca94af3b6bde01f5684b81b9fd9a03fa0eeb@84.247.190.189:656,7cdc65a90e252709b562556f7625830e9599f33c@65.21.203.242:14656,db0e0cff276b3292804474eb8beb83538acf77f5@195.14.6.192:26656,cdcd86ca01858275d0e78ee66b82109ee06df454@65.108.72.253:40656,0642040be904101264ffea9c2af861b37d2be335@173.249.50.178:41656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.pryzm/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${PRYZM_PORT}317%g;
s%:8080%:${PRYZM_PORT}080%g;
s%:9090%:${PRYZM_PORT}090%g;
s%:9091%:${PRYZM_PORT}091%g;
s%:8545%:${PRYZM_PORT}545%g;
s%:8546%:${PRYZM_PORT}546%g;
s%:6065%:${PRYZM_PORT}065%g" $HOME/.pryzm/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${PRYZM_PORT}658%g;
s%:26657%:${PRYZM_PORT}657%g;
s%:6060%:${PRYZM_PORT}060%g;
s%:26656%:${PRYZM_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${PRYZM_PORT}656\"%;
s%:26660%:${PRYZM_PORT}660%g" $HOME/.pryzm/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.pryzm/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.pryzm/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.pryzm/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.015upryzm"|g' $HOME/.pryzm/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.pryzm/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.pryzm/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/pryzmd.service > /dev/null <<EOF
[Unit]
Description=Pryzm node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.pryzm
ExecStart=$(which pryzmd) start --home $HOME/.pryzm
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
pryzmd tendermint unsafe-reset-all --home $HOME/.pryzm
if curl -s --head curl https://server-4.itrocket.net/testnet/pryzm/pryzm_2024-09-01_4101046_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/pryzm/pryzm_2024-09-01_4101046_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.pryzm
    else
  echo "no snapshot founded"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable pryzmd
sudo systemctl restart pryzmd && sudo journalctl -u pryzmd -f
Automatic Installation
pruning: nothing: 100/0/10 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/pryzm/autoinstall/)
```

**Create wallet**
```
pryzmd keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
pryzmd keys add $WALLET --recover
```

# save wallet and validator address
WALLET_ADDRESS=$(pryzmd keys show $WALLET -a)
VALOPER_ADDRESS=$(pryzmd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
pryzmd status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
pryzmd query bank balances $WALLET_ADDRESS 
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, upryzm
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
pryzmd tx staking create-validator \
--amount 1000000upryzm \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(pryzmd tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id indigo-1 \
--fees 3000upryzm \
-y
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${PRYZM_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop pryzmd
sudo systemctl disable pryzmd
sudo rm -rf /etc/systemd/system/pryzmd.service
sudo rm $(which pryzmd)
sudo rm -rf $HOME/.pryzm
sed -i "/PRYZM_/d" $HOME/.bash_profile
