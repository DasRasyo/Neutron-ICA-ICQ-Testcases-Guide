# Neutron-ICA-ICQ-Testcases-Guide


First, make sure you open your Neutron node to external connections. You don't need to do this if you are going to run IBC-ICQ on vps where you have node. This guide assumes that you already open your neutron node to the external connections and do this setup on a different vps.

## We are going to start with updates
```
sudo apt update && apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential \
git make ncdu -y && reboot
```
after this step, connect to your vps again.

```
ver="1.18.2"
cd $HOME
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
```
cd $HOME
git clone -b v0.1.1 https://github.com/neutron-org/neutron.git
cd neutron
make install
```

## We need to create wallet
```
cd
neutrond keys add ibc-relayer
neutrond keys add icq-relayer
```
Make sure you saved your mnemonics!
Fund with some ntrn token these wallets!

Also we need Keplr wallet. You need to import your ibc-relayer wallet to Keplr. After that go to https://jsfiddle.net/kht96uvo/1/ and approve via Keplr and our target chain "theta-testnet-001" will be added to Keplr. You also need fund this wallet with some ATOM. You should go to Cosmos Discord. You will see faucet channel there. https://discord.gg/cosmosnetwork

```
wget https://raw.githubusercontent.com/neutron-org/neutron-contracts/neutron_audit_oak_19_09_2022_fixes/validator_test_upload_contract.sh
chmod +x validator_test_upload_contract.sh
```
we need to edit our node url in this validator_test_upload_contract.sh

```
nano validator_test_upload_contract.sh
```
write your node url like;
http://xxx.xxx.xxx.xxx:26657/

then save and quit. (ctrl x, y > enter)

```
wget https://github.com/neutron-org/neutron-contracts/raw/neutron_audit_oak_19_09_2022_fixes/artifacts/neutron_validators_test.wasm
chmod +x neutron_validators_test.wasm
```

```
./validator_test_upload_contract.sh /root/neutron_validators_test.wasm
```
this will create wallet and contract address. Make sure you save them. We will need these!

## We continue with hermes
```
wget https://github.com/informalsystems/hermes/releases/download/v1.1.0/hermes-v1.1.0-x86_64-unknown-linux-gnu.tar.gz
tar -C ./ -vxzf hermes.tar.gz
rm -f hermes.tar.gz
sudo mv ./hermes /usr/local/bin/
sudo chgrp root /usr/local/bin/hermes
sudo chown root /usr/local/bin/hermes 
```
```
nano /etc/systemd/system/neutron-ibc-cosmoshub-relayer.service

##paste these and save

[Unit]
Description=Neutron IBC CosmosHub Relayer
After=network.target

[Service]
User=ibc-cosmoshub-rly
ExecStart=/usr/local/bin/hermes start

[Install]
WantedBy=multi-user.target
```

```
cd ~/.hermes
wget https://raw.githubusercontent.com/neutron-org/testnets/main/quark/ibc-relayer/config.toml
nano config.toml
```
You need to edit this config with your informations (your own information instead of todo). then save it.
if everything right `health-check` should be okey.

```
export NEUTRON_MNEMONIC="TODO"  #write your ibc-relayer wallet mnemonic
export TARGET_CHAIN_MNEMONIC="TODO" #write your ibc-relayer wallet mnemonic
export TARGET_CHAIN_ID="theta-testnet-001"
export TARGET_KEY_NAME="TODO" # e.g. "cosmoshub-ibc-relayer"
hermes keys add --chain quark-1 --mnemonic-file <(echo "$NEUTRON_MNEMONIC") --key-name ibc-relayer
hermes keys add --chain $TARGET_CHAIN_ID --mnemonic-file <(echo "$TARGET_CHAIN_MNEMONIC") --key-name $TARGET_KEY_NAME
```
Make sure you funded all your wallets.

## Start Service
```
sudo systemctl enable neutron-ibc-cosmoshub-relayer.service
sudo systemctl start neutron-ibc-cosmoshub-relayer.service
sudo systemctl status neutron-ibc-cosmoshub-relayer.service
sudo journalctl -u neutron-ibc-cosmoshub-relayer.service -f
```

## We will create connection between chains
```
hermes create connection --a-chain quark-1 --b-chain theta-testnet-001
```
This may take some time. Save connection-id! We will use this in next steps.

## ICQ Relayer

```
cd
git clone -b v0.1.1 https://github.com/neutron-org/neutron-query-relayer.git
cd neutron-query-relayer
make install
```
Check your version with `neutron_query_relayer version`
it should be 
```
Version: 0.1.1
Commit: 166b2d573b1ec6ea538e158624a1330a7d8eeb93
```

We will import our `icq_wallet` which we created in first steps.
```
neutrond keys add icq_wallet --recover --home /root --keyring-backend test
```
write your `icq-relayer` wallet mnemonics.

```
wget https://raw.githubusercontent.com/neutron-org/testnets/main/quark/icq-relayer/.env
```
edit .env with your informations.
```
RELAYER_NEUTRON_CHAIN_CONNECTION_ID= write our connection id which we created earlier step.
RELAYER_REGISTRY_ADDRESSES= write contract address which we created earlier step.
RELAYER_NEUTRON_CHAIN_RPC_ADDR=http://xxx.xxx.xxx.xxx:26657
RELAYER_NEUTRON_CHAIN_REST_ADDR=http://xxx.xxx.xxx.xxx:1317
```
### NOTE: Delete everything which starts with `#`

```
tmux
export $(grep -v '^#' .env | xargs) && neutron_query_relayer start
```
You can quit from tmux screen with `ctrl +b > d`

## Test Script

``` wget https://raw.githubusercontent.com/neutron-org/neutron-contracts/neutron_audit_oak_19_09_2022_fixes/validator_test.sh
chmod +x validator_test.sh
```

## Before running this script make sure you funded your wallets.
```
We need to edit script
`nano validator_test.sh`
edit node_url and key name like this;
NODE_URL="${NODE_URL:-http://xxx.xxx.xxx.xxx:26657}" 
NEUTRON_KEY_NAME=icq-relayer
```

We are ready to run test script. We will use our connection id again, which we created earlier.

```
./validator_test.sh  [connection-id]
```

follow the script instructions. in the end of the script, you will get TXs. Do not forget send them with form.!
https://docs.google.com/forms/d/1JrjHcdMhMdNJHF8U6Xr3BJbNFRJ_TZJU3eAgZj8wVRc/viewform?pli=1%5D(https://forms.gle/JN7R4dvCNj1PZhz6A&pli=1%5D(https://forms.gle/JN7R4dvCNj1PZhz6A&edit_requested=true
