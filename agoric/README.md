### Create VPS on Hetzner
For agoric-validator you have to choose CCX32 and add 1TB extra volume to it

LOGIN as root

## Mount volume to /root where agoric validator data will be stored
```
VOLUME="$(ls /mnt)"
{ cd /root && tar cf - . ; } | { cd /mnt/$VOLUME/ && tar xvf -  ; echo EXIT=$? ; }
sed -i "s|/mnt/$VOLUME|/root|g" /etc/fstab
mv /root /root-
mkdir /root 
mount /root
```

## Mount Hetzner storage box to /mnt/backup-server
```
sb_username=<storagebox_username>
sb_password=<storagebox_username>

cat <<EOF > /etc/backup-credentials.txt
username=$sb_username
password=$sb_password
EOF
cat /etc/backup-credentials.txt

grep "$sb_username" /etc/fstab || 
printf "//$sb_username.your-storagebox.de/backup /mnt/backup-server cifs iocharset=utf8,rw,credentials=/etc/backup-credentials.txt,uid=root,gid=root,file_mode=0660,dir_mode=0770 0 0\n" >> /etc/fstab
```


## Run script bellow to prepare your RPC server
```
wget -O agoric_mainnet.sh https://raw.githubusercontent.com/kj89/testnet_manuals/main/agoric/agoric_mainnet.sh && chmod +x agoric_mainnet.sh && ./agoric_mainnet.sh
```

## Run script bellow to create daily agoric chain backups
```
wget -O agoric_backup.sh https://raw.githubusercontent.com/kj89/testnet_manuals/main/agoric/agoric_backup.sh && chmod +x agoric_backup.sh
(crontab -l; echo "0 0 * * * bash ./agoric_backup.sh >> /mnt/agoric_backups.log")|awk '!x[$0]++'|crontab -
```

### Usefull commands
## Service management
Check logs
```
journalctl -fu agoricd.service
```

Stop agoric service
```
service ag0 stop
```

Start agoric service
```
service ag0 start
```

Configuration file
```
vim ~/.ag0/config/config.toml
```

Check validator node status
```
curl -O https://gist.githubusercontent.com/michaelfig/2319d0573c0f6bc257de6a97e3d46b3e/raw/cf2b2b784c60359d8e49fb3fa198b5be13c816be/check-validator.js
node check-validator.js
```

## Node info
Synchronization info
```
ag0 status 2>&1 | jq .SyncInfo
```

Validator info
```
ag0 status 2>&1 | jq .ValidatorInfo
```

Node info
```
ag0 status 2>&1 | jq .NodeInfo
```

## Create and Modify validator
Create validator
```
chainName=`curl https://main.agoric.net/network-config | jq -r .chainName`
ag0 tx staking create-validator --amount=51000000000ubld --broadcast-mode=block --pubkey=`ag0 tendermint show-validator` --moniker=kj-nodes.xyz --website="http://kj-nodes.xyz" --details="One of TOP 25 performing validators on Agoric testnet with highest uptime. Uptime is important to me. Server is constantly being monitored and maintained. You can contact me at discord: kristaps#8455 or telegram: @janispaegle" --commission-rate="0.07" --commission-max-rate="0.20" --commission-max-change-rate="0.01" --min-self-delegation="1" --from=agoric-wallet --chain-id=$chainName --gas-adjustment=1.4 --fees=5001ubld
```

Modify validator
```
chainName=`curl https://main.agoric.net/network-config | jq -r .chainName`
ag0 tx staking edit-validator --moniker="kj-nodes.xyz" --website="http://kj-nodes.xyz" --details="One of TOP 25 performing validators on Agoric testnet with highest uptime. Uptime is important to me. Server is constantly being monitored and maintained. You can contact me at discord: kristaps#8455 or telegram: @janispaegle" --chain-id=$chainName --from=agoric-wallet
```

## Wallet operations
Recover wallet
```
ag0 keys add agoric-wallet --recover
```

Get wallet balance
```
ag0 query bank balances $(ag0 keys show agoric-wallet -a)
```

List of wallets
```
ag0 keys list
```

Delete wallet
```
ag0 keys delete agoric-wallet
```

## Configruation reset
Reset configs
```
ag0 unsafe-reset-all
```

## Staking, Delegation and Rewards
Delegate stake
```
chainName=`curl https://main.agoric.net/network-config | jq -r .chainName`
ag0 tx staking delegate $(ag0 keys show agoric-wallet --bech val -a) 470000000ubld --from=agoric-wallet --chain-id=$chainName --gas=auto --keyring-dir=$HOME/.ag0
```

Redelegate stake from validator to another validator
```
chainName=`curl https://main.agoric.net/network-config | jq -r .chainName`
ag0 tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 5000000ubld --from=agoric-wallet --chain-id=$chainName --gas=auto --keyring-dir=$HOME/.ag0
```

Withdraw rewards
```
chainName=`curl https://main.agoric.net/network-config | jq -r .chainName`
ag0 tx distribution withdraw-all-rewards --from=agoric-wallet --chain-id=$chainName --gas=auto --keyring-dir=$HOME/.ag0
```

### Migrate Agoric validator to another VPS
1. First of all you have to backup your configuration files on your old validator node located in `~/.ag0/config/`
2. Set up new VPS
3. Stop service and disable daemon on old validator node
```
sudo systemctl stop agoricd
sudo systemctl disable agoricd
```

_(Be sure that your ag-chain-cosmos is not running on the old machine. If it is, you will be slashed for double-signing.)_

4. Use guide for validator node setup - [Validator Guide for Incentivized Testnet](https://github.com/Agoric/agoric-sdk/wiki/Validator-Guide-for-Incentivized-Testnet)
>When you reach step [Syncing Your Node](https://github.com/Agoric/agoric-sdk/wiki/Validator-Guide-for-Incentivized-Testnet#syncing-your-node) you have to copy and replace configuration files located in `~/.ag0/config/` with those we saved in step 1
5. Finish setup by synchronizing your node with network
6. After your node catch up you have to restore your key. For that you will need 24-word mnemonic you saved on key creation
>To recover your key follow this guide - [How do I recover a key?](https://github.com/Agoric/agoric-sdk/wiki/Validator-Guide-for-Devnet#how-do-i-recover-a-key)
7. Make sure your validator is not jailed
>To unjail use this guide - [How do I unjail my validator?](https://github.com/Agoric/agoric-sdk/wiki/Validator-Guide#how-do-i-unjail-my-validator)
8. After you ensure your validator is producing blocks in explorer and is healthy you can shut down old validator server