<h1 align="center">Nois Testnet Kurulum</h1>

![NOIS - TESTNET](https://user-images.githubusercontent.com/107190154/192220441-07b68638-67a6-4df5-9fe7-306fbccd8c21.gif)

### Gereksinimler

|      CPU        |   RAM    |  Disk    | 
|-----------------|----------|----------|
|4CPU|   8GB    | 200GB    |

## [Explorer](https://explorer.nodestake.top/nois-testnet/)

## Kuruluma Başlayalım.

### Sistemi Güncelleyelim
```
sudo apt-get update -y
sudo apt install -y build-essential
sudo apt install make clang pkg-config libssl-dev build-essential git jq ncdu bsdmainutils -y < "/dev/null"
```

### Go Kurulumu
```
cd $HOME
wget -O go1.18.2.linux-amd64.tar.gz https://go.dev/dl/go1.18.2.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.2.linux-amd64.tar.gz && rm go1.18.2.linux-amd64.tar.gz
echo 'export GOROOT=/usr/local/go' >> $HOME/.bashrc
echo 'export GOPATH=$HOME/go' >> $HOME/.bashrc
echo 'export GO111MODULE=on' >> $HOME/.bashrc
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bashrc && . $HOME/.bashrc
go version
```

### Nois Full Node Reposunu Klonlayalım.
```
git clone https://github.com/noislabs/full-node.git 
cd full-node/full-node/
```

### Binary Dosyayını Yükleyelim
```
./setup.sh
mkdir -p $HOME/go/bin
mv out/noisd $HOME/go/bin/
export PATH=$HOME/go/bin:$PATH
```

### Initialize İşlemi yapıyoruz
```
noisd init nodeisminiz --chain-id nois-testnet-002
```

### Genesis Dosyamızı indiriyoruz
```
wget https://raw.githubusercontent.com/noislabs/testnets/main/nois-testnet-002/genesis.json -O $HOME/.noisd/config/genesis.json
```

### Seed ve Peer ayarı
```
SEEDS=""
PEERS="a1222dfb8641e0cb55615b75e0122d5695be1f35@node-0.noislabs.com:26656,2df500525826199afc25665ee7cc45ceb86d68d7@node-1.noislabs.com:26656,61be6aa87471196757ea0f7b1d7897e97b4e09c2@node-2.noislabs.com:26656,cf16671c00eec9a9a047a5c6aa8510cb681b64b8@node-3.noislabs.com:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.noisd/config/config.toml
```

### Minimum gas değeri ayarlaması
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.00125unois\"/" $HOME/.noisd/config/app.toml
```

### Prometheus ayarı
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.noisd/config/config.toml
```

### Pruning 
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.noisd/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.noisd/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.noisd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.noisd/config/app.toml
```

### İndexer
```
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.noisd/config/config.toml
```

### Servisi Başlatalım
```
sudo tee /etc/systemd/system/noisd.service > /dev/null <<EOF
[Unit]
Description=noisd
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which noisd) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
### Başlatalım
```                                                          
sudo systemctl daemon-reload
sudo systemctl enable noisd
sudo systemctl restart noisd
systemctl restart systemd-journald
```

### Snapshot
>Dileyen Kullanabilir.
```
sudo systemctl stop noisd
noisd tendermint unsafe-reset-all --home $HOME/.noisd --keep-addr-book

SNAP_RPC="https://nois-testnet-rpc.polkachu.com:443"

LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

peers="de14a239b0a942a6053bb5dc51606f568716bbaf@65.109.28.219:17356"
sed -i 's|^persistent_peers *=.*|persistent_peers = "'$peers'"|' $HOME/.noisd/config/config.toml

sed -i -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.noisd/config/config.toml

sudo systemctl restart noisd
sudo journalctl -fu noisd -o cat
```

### Log Kontrol
``` 
journalctl -fu noisd -o cat
```  

### Sync Durumu
``` 
noisd status 2>&1 | jq .SyncInfo
``` 

### Cüzdan Oluşturma
``` 
noisd keys add cüzdanismi
``` 

### Faucet için
> Noisadresiniz yazan yere kendi cüzdan adresinizi yazın.
```
curl --header "Content-Type: application/json" \
  --request POST \
  --data '{"denom":"unois","address":"noisadresiniz"}' \
  http://faucet.noislabs.com/credit
  ```
  
### Validator OLuşturma Komutu
```
noisd tx staking create-validator \
--amount 99900000unois \
--from cüzdanisminiz \
--commission-max-change-rate "0.01" \
--commission-max-rate "0.2" \
--commission-rate "0.09" \
--min-self-delegation "1" \
--pubkey  $(noisd tendermint show-validator) \
--moniker nodeisminiz \
--chain-id nois-testnet-002 \
--fees 500unois
```
  
### Silme Komutları
```
sudo systemctl stop noisd
sudo systemctl disable noisd
sudo rm /etc/systemd/system/nois* -rf
sudo rm $(which noisd) -rf
sudo rm $HOME/.nois* -rf
sudo rm $HOME/full-node -rf
sed -i '/NOIS_/d' ~/.bash_profile
```

### Hepinize Kolay Gelsin.
### Sorunuz olursa Notitia Telegram Kanalımıza Bekleriz: https://t.me/NotitiaGroup
