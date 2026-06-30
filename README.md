# SAFROCHAIN-MAINNET

<img width="1252" height="380" alt="image" src="https://github.com/user-attachments/assets/deb7c914-9658-4a2c-8de2-93d1ddb189f6" />



RPC: https://safrochain-rpc.zeycanode.com/

API: https://safrochain-api.zeycanode.com/

### Sistem Güncellemesi ve Bağımlılıklar
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```
### Go Kurulumu
```
cd $HOME
wget https://go.dev/dl/go1.25.8.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.25.8.linux-amd64.tar.gz
rm go1.25.8.linux-amd64.tar.gz
```
### Çevre değişkenlerini işletim sistemine tanımlama
```
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```
### Version Kontrolü (go version go1.25.8 çıktısı alınmalı)
```
go version
```
### Binary Derleme
```
cd $HOME
git clone https://github.com/Safrochain-Org/safrochain-node ~/safrochain-node
cd ~/safrochain-node
git fetch --tags
git checkout v0.2.2
make install
```
### Doğrulama (safrochaind v0.2.2 çıktısı alınmalı)
```
safrochaind version
```
### Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
mkdir -p $HOME/.safrochain/cosmovisor/genesis/bin
mkdir -p $HOME/.safrochain/cosmovisor/upgrades
cp $(which safrochaind) $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind
sudo ln -sfn $HOME/.safrochain/cosmovisor/genesis $HOME/.safrochain/cosmovisor/current
```
### İnit 
```
safrochaind init "MONIKER_ADINIZ" --chain-id safrochain-1 --home ~/.safrochain
```
### Genesis
```
curl -L https://raw.githubusercontent.com/Safrochain-Org/mainnet-genesis/main/genesis.json -o ~/.safrochain/config/genesis.json
```
### Kontrol (Beklenen çıktı: c05ac5aec1918df9edb257e8e0eea184d73edc51370eb4aa9f0b4f0aad615c4d)
```
sha256sum ~/.safrochain/config/genesis.json
```
### Seeds
```
SEEDS="bc772fdc9749e6dfd200a9428f07d86fe4fd34ec@seed.safrochain.network:26666,d323d296ba55e89fb6ce1a724f8da1740bd8cbb0@seed2.safrochain.network:26670"

sed -i -e "s|^seeds *=.*|seeds = \"$SEEDS\"|" ~/.safrochain/config/config.toml
```
### Gas ve Pruning Ayarları
```
sed -i \
    -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.05usaf"|' \
    -e 's|^pruning *=.*|pruning = "default"|' \
    ~/.safrochain/config/app.toml
```
### Servis Dosyasının Oluşturulması
```
sudo tee /etc/systemd/system/safrochaind.service > /dev/null << 'EOF'
[Unit]
Description=Safrochain Mainnet Node (Cosmovisor)
After=network-online.target

[Service]
Type=simple
User=root
ExecStart=/root/go/bin/cosmovisor run start --home /root/.safrochain
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576
TimeoutStopSec=30s
Environment="DAEMON_HOME=/root/.safrochain"
Environment="DAEMON_NAME=safrochaind"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF
```
### Başlatma
```
sudo systemctl daemon-reload
sudo systemctl enable safrochaind
sudo systemctl start safrochaind
```
### Cüzdan oluşturma
```
safrochaind keys add wallet --home ~/.safrochain
```
### Cüzdan Ekleme
```
safrochaind keys add wallet --recover --home ~/.safrochain
```
### Validatör Oluşturma
  Validatör Yapılandırma Dosyası Oluşturma

```
cat > $HOME/.safrochain/validator.json << EOF
{
  "pubkey": $(safrochaind tendermint show-validator --home ~/.safrochain),
  "amount": "1000000usaf",
  "moniker": "MONIKER_ADINIZ",
  "identity": "",
  "website": "",
  "security": "",
  "details": "Safrochain Mainnet Validator - CoreNode",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```
### Validatör Oluşturma 
```
safrochaind tx staking create-validator $HOME/.safrochain/validator.json \
    --from wallet \
    --chain-id safrochain-1 \
    --gas auto \
    --gas-adjustment 1.4 \
    --fees 300usaf \
    --home ~/.safrochain \
    -y
```
### ÖNEMLİ KOMUTLAR
### Nodu Başlatma
```
sudo systemctl start safrochaind
```
### Nodu Durdurma
```
sudo systemctl stop safrochaind
```
### Nodu Yeniden Başlatma
```
sudo systemctl restart safrochaind
```
### Log Kontrolü
```
sudo journalctl -fu safrochaind -o cat
```
### Sync Kontrolü
```
LOCAL=$(curl -s http://localhost:26657/status | jq -r '.result.sync_info.latest_block_height')
NETWORK=$(curl -s https://rpc.safrochain.network/status | jq -r '.result.sync_info.latest_block_height')
echo "Yerel Blok: $LOCAL | Güncel Ağ Bloğu: $NETWORK | Kalan Fark: $((NETWORK - LOCAL)) blok"
```

### Delegasyon (Kendi Kendine Stake Etme)
```
VAL_ADDR=$(safrochaind keys show wallet --bech val -a --home ~/.safrochain)

safrochaind tx staking delegate "$VAL_ADDR" 1000000usaf \
    --from wallet \
    --chain-id safrochain-1 \
    --gas auto \
    --gas-adjustment 1.5 \
    --gas-prices 0.05usaf \
    --home ~/.safrochain \
    -y
```
### Unjail
```
safrochaind tx slashing unjail \
    --from wallet \
    --chain-id safrochain-1 \
    --gas auto \
    --gas-adjustment 1.5 \
    --gas-prices 0.05usaf \
    --home ~/.safrochain \
    -y
```
### Nodu Silme
```
safrochaind tx slashing unjail \
    --from wallet \
    --chain-id safrochain-1 \
    --gas auto \
    --gas-adjustment 1.5 \
    --gas-prices 0.05usaf \
    --home ~/.safrochain \
    -y
```





