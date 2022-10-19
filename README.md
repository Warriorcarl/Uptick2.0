<p style="font-size:14px" align="right">
<a href="https://t.me/+-tFu9biVYCdiYTJl" target="_blank">Join our telegram <img src="https://user-images.githubusercontent.com/50621007/183283867-56b4d69f-bc6e-4939-b00a-72aa019d1aea.png" width="30"/></a>
<a href="https://discord.gg/WuVV4VYcBr
" target="_blank">Join our discord <img src="https://user-images.githubusercontent.com/50621007/176236430-53b0f4de-41ff-41f7-92a1-4233890a90c8.png" width="30"/></a>
<a href="https://nodewarrior.dev/" target="_blank">Visit our website <img src="https://avatars.githubusercontent.com/u/100796465?s=400&u=e7a1fab25433b68f07a989fe0fefff1644170af5&v=4" width="30"/></a>
</p>

<p align="center">
  <img height="100" height="auto" src="https://user-images.githubusercontent.com/50621007/171044333-016e348d-1d96-4d00-8dce-f7de45aa9f84.png">
</p>

# Uptick node setup for testnet — uptick_7000-1

Official documentation:
>- https://docs.uptick.network/testnet/

Explorer:
>- https://explorer.testnet.uptick.network/uptick-network-testnet

## Usefull tools and references
> To set up monitoring for your validator node navigate to [Set up monitoring and alerting for uptick validator](https://github.com/Warriorcarl/Uptick2.0/tree/main/monitoring)
>
> To migrate your validator to another machine read [Migrate your validator to another machine](https://github.com/Warriorcarl/Uptick2.0/blob/main/migrate_validator.md)

## Recommended Hardware Requirements 
 - 2x CPUs; the faster clock speed the better
 - 16GB RAM
 - 200GB of storage (SSD or NVME)
 - Permanent Internet connection (traffic will be minimal during testnet; 10Mbps will be plenty - for production at least 100Mbps is expected)

## Set up your uptick fullnode
### Option 1 (automatic)
You can setup your uptick fullnode in few minutes by using automated script below. It will prompt you to input your validator node name!
```
wget -O uptick.sh https://raw.githubusercontent.com/Warriorcarl/Uptick2.0/main/uptick.sh && chmod +x uptick.sh && ./uptick.sh
```

### Option 2 (manual)
You can follow [manual guide](https://github.com/Warriorcarl/Uptick2.0/blob/main/uptick/manual_install.md) if you better prefer setting up node manually

## Post installation

When installation is finished please load variables into system
```
source $HOME/.bash_profile
```

Next you have to make sure your validator is syncing blocks. You can use command below to check synchronization status
```
uptickd status 2>&1 | jq .SyncInfo
```

### (OPTIONAL) Disable and cleanup indexing
```
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.uptickd/config/config.toml
sudo systemctl restart uptickd
sleep 3
sudo rm -rf $HOME/.uptickd/data/tx_index.db
```

### (OPTIONAL) State Sync
You can state sync your node in minutes by running commands below
```
peers="5aa25d2e0c3fdc00c0a9738ba0bb6cce2ef902a2@15.235.40.222:12656"; \
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.uptickd/config/config.toml

SNAP_RPC="https://uptick.t.rpc.nodewarrior.dev:443"; \
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash); \
echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

sed -i -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.uptickd/config/config.toml

sudo systemctl stop uptickd && \
uptickd tendermint unsafe-reset-all --home ~/.uptickd && \
sudo systemctl restart uptickd
```

### Create wallet
To create new wallet you can use command below. Don’t forget to save the mnemonic
```
uptickd keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase
```
uptickd keys add $WALLET --recover
```

To get current list of wallets
```
uptickd keys list
```

### Save wallet info
Add wallet and valoper address into variables 
```
UPTICK_WALLET_ADDRESS=$(uptickd keys show $WALLET -a)
UPTICK_VALOPER_ADDRESS=$(uptickd keys show $WALLET --bech val -a)
echo 'export UPTICK_WALLET_ADDRESS='${UPTICK_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export UPTICK_VALOPER_ADDRESS='${UPTICK_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Fund your wallet
In order to create validator first you need to fund your wallet with testnet tokens.
To top up your wallet join [uptick discord server](https://discord.gg/fYsw5VrXG8) and navigate to:
- **#faucet** to request test tokens

To request a faucet grant:
```
$request <YOUR_WALLET_ADDRESS>
```

To check wallet balance:
```
$balance <YOUR_WALLET_ADDRESS>
```

### Create validator
Before creating validator please make sure that you have at least 1 uptick (1 uptick is equal to 1000000000000000000 auptick) and your node is synchronized

To check your wallet balance:
```
uptickd query bank balances $UPTICK_WALLET_ADDRESS
```
> If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue 

To create your validator run command below
```
uptickd tx staking create-validator \
  --amount 1000000000000000000auptick \
  --from $WALLET \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.07" \
  --min-self-delegation "1" \
  --pubkey  $(uptickd tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $UPTICK_CHAIN_ID
```

## Security
To protect you keys please make sure you follow basic security rules

### Set up ssh keys for authentication
Good tutorial on how to set up ssh keys for authentication to your server can be found [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)

### Basic Firewall security
Start by checking the status of ufw.
```
sudo ufw status
```

Sets the default to allow outgoing connections, deny all incoming except ssh and 26656. Limit SSH login attempts
```
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh/tcp
sudo ufw limit ssh/tcp
sudo ufw allow ${UPTICK_PORT}656,${UPTICK_PORT}660/tcp
sudo ufw enable
```

## Monitoring
To monitor and get alerted about your validator health status you can use my guide on [Set up monitoring and alerting for uptick validator](https://github.com/Warriorcarl/Uptick2.0/blob/main/uptick/monitoring/README.md)

## Calculate synchronization time
This script will help you to estimate how much time it will take to fully synchronize your node\
It measures average blocks per minute that are being synchronized for period of 5 minutes and then gives you results
```
wget -O synctime.py https://raw.githubusercontent.com/Warriorcarl/Uptick2.0/main/uptick/tools/synctime.py && python3 ./synctime.py
```

### Check your validator key
```
[[ $(uptickd q staking validator $UPTICK_VALOPER_ADDRESS -oj | jq -r .consensus_pubkey.key) = $(uptickd status | jq -r .ValidatorInfo.PubKey.value) ]] && echo -e "\n\e[1m\e[32mTrue\e[0m\n" || echo -e "\n\e[1m\e[31mFalse\e[0m\n"
```

### Get list of validators
```
uptickd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

## Get currently connected peer list with ids
```
curl -sS http://localhost:${UPTICK_PORT}657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

## Usefull commands
### Service management
Check logs
```
journalctl -fu uptickd -o cat
```

Start service
```
sudo systemctl start uptickd
```

Stop service
```
sudo systemctl stop uptickd
```

Restart service
```
sudo systemctl restart uptickd
```

### Node info
Synchronization info
```
uptickd status 2>&1 | jq .SyncInfo
```

Validator info
```
uptickd status 2>&1 | jq .ValidatorInfo
```

Node info
```
uptickd status 2>&1 | jq .NodeInfo
```

Show node id
```
uptickd tendermint show-node-id
```

### Wallet operations
List of wallets
```
uptickd keys list
```

Recover wallet
```
uptickd keys add $WALLET --recover
```

Delete wallet
```
uptickd keys delete $WALLET
```

Get wallet balance
```
uptickd query bank balances $UPTICK_WALLET_ADDRESS
```

Transfer funds
```
uptickd tx bank send $UPTICK_WALLET_ADDRESS <TO_UPTICK_WALLET_ADDRESS> 1000000000000000000auptick
```

### Voting
```
uptickd tx gov vote 1 yes --from $WALLET --chain-id=$UPTICK_CHAIN_ID
```

### Staking, Delegation and Rewards
Delegate stake
```
uptickd tx staking delegate $UPTICK_VALOPER_ADDRESS 1000000000000000000auptick --from=$WALLET --chain-id=$UPTICK_CHAIN_ID --gas=auto
```

Redelegate stake from validator to another validator
```
uptickd tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 1000000000000000000auptick --from=$WALLET --chain-id=$UPTICK_CHAIN_ID --gas=auto
```

Withdraw all rewards
```
uptickd tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$UPTICK_CHAIN_ID --gas=auto
```

Withdraw rewards with commision
```
uptickd tx distribution withdraw-rewards $UPTICK_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=$UPTICK_CHAIN_ID
```

### Validator management
Edit validator
```
uptickd tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=$UPTICK_CHAIN_ID \
  --from=$WALLET
```

Unjail validator
```
uptickd tx slashing unjail \
  --broadcast-mode=block \
  --from=$WALLET \
  --chain-id=$UPTICK_CHAIN_ID \
  --gas=auto
```

### Delete node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop uptickd
sudo systemctl disable uptickd
sudo rm /etc/systemd/system/uptick* -rf
sudo rm $(which uptickd) -rf
sudo rm $HOME/.uptickd* -rf
sudo rm $HOME/uptick -rf
sed -i '/UPTICK_/d' ~/.bash_profile
```
