# Kichain IBC Challenge
The aim of this dicumentation is to show how to setup a relayer between the kichain-t-4 chain and sifchain-1 with Hermes.

### Prerequisites
* Go version 1.16+

* Synchronised node with kichain-t-4

* Synchronised node with sifchain-1

* Hermes

## Go Installation
Run the following commands:
* wget https://dl.google.com/go/go1.16.linux-amd64.tar.gz
* sudo tar -C /usr/local -xzf go1.16.linux-amd64.tar.gz
* GOPATH=/usr/local/go
* PATH=$GOPATH/bin:$PATH
* mkdir -p $HOME/go/bin
* echo "export PATH=$PATH:$(go env GOPATH)/bin" >> ~/.bash_profile
* source ~/.bash_profile
* go version

## Synchronized sifchain node
Run the following commands:
### Binary install
* wget https://github.com/Sifchain/sifnode/releases/download/betanet-0.9.5/sifnoded-betanet-0.9.5-linux-amd64.zip
* unzip sifnoded-betanet-0.9.5-linux-amd64.zip
* mv sifnoded /usr/bin
* sifnoded init YOUR_MONIKER
* sifnoded keys add YOUR_KEY_NAME
* wget https://github.com/Sifchain/networks/raw/master/mainnet/sifchain-1/genesis.json.gz
* gunzip -k genesis.json.gz
* mv genesis.json ~/.sifnoded/config

### Backup the blockchain
* rm -rf ~/.sifnoded/data
* mkdir -p ~/.sifnoded/data
* cd ~/.sifnoded/data
* SNAP_NAME=$(curl -s http://135.181.60.250:8081/sifchain/ |egrep -o ">sifchain.*tar"|tr -d ">");\
* wget -O - http://135.181.60.250:8081/sifchain/${SNAP_NAME}|tar xf -

### Systemctl sifnoded service
Setup a systemd service with these configuration:

``[Unit]

Description=Sifnode	

[Service]

Environment=SIFNODED_P2P_LADDR=tcp://0.0.0.0:2110

Environment=SIFNODED_RPC_LADDR=tcp://0.0.0.0:2111

Environment=SIFNODED_GRPC_ADDRESS=127.0.0.1:2112

Environment=SIFNODED_API_ADDRESS=tcp://127.0.0.1:2113

Environment=SIFNODED_NODE=tcp://127.0.0.1:2111

Environment=SIFNODED_P2P_SEEDS="8dc1863d1d23cf9ad7cbea215c19bcbe8bf39702@p2p.baaa7e56-cc71-4ae4-b4b3-c6a9d4a9596a.cryptodotorg.bison.run:26656,8a7922f3fb3fb4cfe8cb57281b9d159ca7fd29c6@p2p.aef59b2a-d77e-4922-817a-d1eea614aef4.cryptodotorg.bison.run:26656,494d860a2869b90c458b07d4da890539272785c9@p2p.fabc23d9-e0a1-4ced-8cd7-eb3efd6d9ef3.cryptodotorg.bison.run:26656,dc2540dabadb8302da988c95a3c872191061aed2@p2p.7d1b53c0-b86b-44c8-8c02-e3b0e88a4bf7.cryptodotorg.herd.run:26656,33b15c14f54f71a4a923ac264761eb3209784cf2@p2p.0d20d4b3-6890-4f00-b9f3-596ad3df6533.cryptodotorg.herd.run:26656,d2862ef8f86f9976daa0c6f59455b2b1452dc53b@p2p.a088961f-5dfd-4007-a15c-3a706d4be2c0.cryptodotorg.herd.run:26656,87c3adb7d8f649c51eebe0d3335d8f9e28c362f2@seed-0.crypto.org:26656,e1d7ff02b78044795371beb1cd5fb803f9389256@seed-1.crypto.org:26656,2c55809558a4e491e9995962e10c026eb9014655@seed-2.crypto.org:26656"

Environment=SIFNODED_P2P_MAX_NUM_INBOUND_PEERS=500

Environment=SIFNODED_SNAPSHOT_INTERVAL=1000

Environment=SIFNODED_P2P_MAX_NUM_OUTBOUND_PEERS=60

Environment=HOME=/root

LimitNOFILE=500000

ExecStart=/usr/bin/sifnoded start --pruning custom --pruning-keep-recent 362880 --pruning-keep-every 0 --pruning-interval 100

LogLevelMax=3

[Install]

WantedBy=multi-user.target``
* sudo systemctl daemon-reload
* sudo systemctl enable sifchaind
* sudo systemctl start sifchaind

## Hermes Setup
Run the following commands:
### Rustc installation
* curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
* Restart your terminal
* rustc --version
### Hermes installation
* mkdir -p $HOME/.hermes/bin
* wget https://github.com/informalsystems/ibc-rs/releases/download/v0.7.0/hermes-v0.7.0-x86_64-unknown-linux-gnu.tar.gz
* tar -C $HOME/.hermes/bin/ -vxzf hermes-v0.7.0-x86_64-unknown-linux-gnu.tar.gz
* export PATH="$HOME/.hermes/bin:$PATH"
* HERMESPATH=/root/.hermes/bin
* echo "export PATH=$PATH:$(hermes env HERMESPATH)/bin" >> ~/.bash_profile
* source ~/.bash_profile
* hermes version
### Hermes configuration
* touch $HOME/.hermes/config.toml
* vi $HOME/.hermes/config.toml
* Adapt your configuration with this https://github.com/notional-labs/notional/blob/master/hermes/config.toml
### Keys
You have to create keys for each blockchain and add them to your hermes configuration
* kid keys add KI_RELAYER_NAME
* sifnoded keys add SIF_RELAYER_NAME
Then add them to hermes
* hermes -c $HOME/.hermes/config.toml keys add kichain-t-4 -f /path/to/your/key
* hermes -c $HOME/.hermes/config.toml keys add sifchain -f /path/to/your/key
### Create client
Run this command to create client in both chains
* hermes tx raw create-client kichain-t-4 sifchain-1 **DEE2A2E3AA854E627CA07E0C821058E7E5C96BA54F895A6D56CAB6DA1F2B3CA3** our tx hash
* hermes tx raw create-client sifchain-1 kichain-t-4 **59F19A012897EAA049B002964CACD54D7F3C6342A48DF3F0F8FDBA1C1F04A129** 
### Create a channel
To create channel run this command
* hermes create channel kichain-t-4 sifchain-1  --port-a transfer --port-b transfer -o unordered **016A7312A0D38A0DC6C6A6909DAC7B167BA35548029ED15D87E60BB108C73805**
### Send packet into channel
hermes tx raw ft-transfer sifchain-1 kichain-t-4 transfer channel-317 9999 -o 1000 -n 2 -d utki **332515DF52D15C4306678B46752D7EFED4F255444A9D1CE2652AF449B908AC56**
* hermes tx raw packet-recv sifchain-1 kichain-t-4 transfer channel-317 **00FF2EAFF7B671C0A745DCE571D82C2CF75B79AB6E7BDA8C6A166F4EE3EDCA6E**
### Query packet
* hermes query packet unreceived-packets sifchain-1 transfer channel-13
