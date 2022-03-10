# Solana Validator Node configuration for Equinix Metal

Make sure and check out the README document.

IMPORTANT: This guide is specifically for Equinix Machines from the Solana Reserve pool accessed through the Solana Foundation Server Program. https://solana.foundation/server-program

You must be running Ubuntu 20.04

So you have your shiny new beast of a server. Let's make it a Solana Validatoooooooor.
First things first - OS security updates
```
apt update
apt upgrade
apt dist-upgrade
```
create user sol

```
adduser sol

usermod -aG sudo sol

su - sol
```

Partition NVME into ~570gb (swap) and 3000gb (ledger and accounts) - for EQ1 Spec 3.8TB NVME

Adding new process using GPT partition with gdisk for larger filessytems.

Enter the "n" then hit enter
Enter the "1" then hit enter...and so on
```
sudo gdisk /dev/nvme0n1
n, 1, enter (2048 default first sector), +3000G, enter (8300 default), n, 2, enter (default first available sector), enter (max sector available), 8200, w, y
```

Now make filesystems, directories, delete and make new swap, etc.
```
sudo fdisk -l 

sudo mkfs -t ext4 /dev/nvme0n1p1

sudo mount /dev/nvme0n1p1 /mt

sudo mkswap /dev/nvme0n1p2

```
Discover the old swap directory, turn it off, and turn it on the new one.
```
sudo swapon --show
```
You need to look at the directory and pick the correct /dev/sd*

It could be /dev/sdb2 or /dev/sdc2 so edit the next line below to the proper sd**

It will almost always be the one showig 1.9GB of swap size
```
sudo swapoff /dev/sda2
```
Next is editing the swappiness to 10 and turning our new swap partition on.
```
echo 'vm.swappiness=10' | sudo tee --append /etc/sysctl.conf > /dev/null

sudo sysctl -p

sudo swapon /dev/nvme0n1p2
```
Capture nvme0n1p1 and nvme0n1p2 UUIDs to edit into /etc/fstab

Let's take a look at the file first to get an idea of what is needed here.
```
sudo nano /etc/fstab
```
You should see something similar to this:
UUID=e6eafc79-85c3-4208-82ac-41b73d75cd31       /       ext4    errors=remount-ro       0       1
UUID=4b8f8a7b-8b8f-4984-a341-5770f8b365a1       none    swap    none    0       0

These are the default OS drives and we will be modifying swap later. Do **not** modify the entry of root, the ext4 partition mounted at /. You will need to use the UUID's of the two partitions you just made (nvmeon1p1 and nvme0n1p2).

`ctrl + x` to exit

```
lsblk -f
```
Copy the section that looks similar to the below nvme0n1 partition tree and past it into a notepad (or VScode, etc) so that you can copy/past into fstab properly. We just need the UUID's so in the example below copy "5c24e241-239c-4aa5-baa6-fbb6fb44a847" and "37215cf2-244c-4f2e-98f9-6f327694fe7e" and note which partition each belongs to (/mt and swap respectively). Your UUIDs will be different!
```
nvme0n1
├─nvme0n1p1 ext4         5c24e241-239c-4aa5-baa6-fbb6fb44a847    2.8T     0% /mt
└─nvme1n1p2 swap   1     37215cf2-244c-4f2e-98f9-6f327694fe7e                [SWAP]
```
These UUID above need to be edited into the fstab config below
```
sudo nano /etc/fstab
```
Leave the first UUID alone (OS related), on the swap partition line, while your UUID values will be different, edit the previous one to have your new UUID similar to from
```
UUID=4b8f8a7b-8b8f-4984-a341-5770f8b365a1       none    swap    none    0       0
```
To be updated becoming
```
UUID=37215cf2-244c-4f2e-98f9-6f327694fe7e       none    swap    none    0       0
```
  Now **append these lines under whatever current UUIDs are listed** as the ones already in the file are boot/OS related.
also make sure UUID is correct as they can change

```
#Validator config
UUID=5c24e241-239c-4aa5-baa6-fbb6fb44a847 /mt  auto nosuid,nodev,nofail 0 0
#ramdrive and swap
tmpfs /mnt/ramdrive tmpfs rw,size=80G 0 0
```
save / exit  

`ctrl + o`, enter, `ctrl + x`   

The complete file should look like this (but with your own UUIDs):
```
UUID=e6eafc79-85c3-4208-82ac-41b73d75cd31       /       ext4    errors=remount-ro       0       1
UUID=37215cf2-244c-4f2e-98f9-6f327694fe7e       none    swap    none    0       0
#Validator config
UUID=5c24e241-239c-4aa5-baa6-fbb6fb44a847 /mt  auto nosuid,nodev,nofail 0 0
#ramdrive and swap
tmpfs /mnt/ramdrive tmpfs rw,size=80G 0 0
```

Create the ramdrive folder and mount everything.
```
sudo mkdir /mnt/ramdrive

sudo mount --all --verbose
```
Finish making directories
```
sudo mkdir -p /mt/ledger/validator-ledger

sudo mkdir /mt/solana-accounts

sudo mkdir ~/log
```

Edit permissions and make sure user sol is the owner for solana directories
```
sudo chown -R sol:sol /mt/*

sudo chown sol:sol ~/log
```

Set up the firewall / ssh
```
sudo snap install ufw

sudo ufw enable

sudo ufw allow ssh
```

Open ports in UFW firewall for Solana Validator operation:
```
sudo ufw allow 53 

sudo ufw allow 8000:8010/udp
```

Install the Solana CLI! Don't forget to check for current version (1.8.14 as of 2/4/2022)

```
sh -c "$(curl -sSfL https://release.solana.com/v1.8.14/install)"
```
I will ask you to map the PATH just copy and paste the below:

```
export PATH="/home/sol/.local/share/solana/install/active_release/bin:$PATH"
```
You are now able to join Solana gossip which is an overarching network communication layer which all RPCs and Validators chatter in. If you see a steam of logs, and no errors then have officially connected directly to the Solana network.

```
solana-gossip spy --entrypoint entrypoint.mainnet-beta.solana.com:8001
```
If your machine is gossiping without any errors it can be spun up on the mainnet to start reading the chain data.

exit gossip with ctrl + c
#
## Creating keys

You need three keys minimum so let's start with a summary:  

1) validator ID which this guide chooses to name `validator-keypair.json` which is located in the home directly `/home/sol` - This key identified the validator in gossip (the Solana Protocol) and pay for voting. This account will pay roughly 1 SOL per 24 hours in transactions fees to participate in block building (voting) consensus.
2) vote account ID which this guide refers to as `vote-account-keypair.json` located in `/home/sol` - This key is also referred to as the rewards account. Stakers, or delegators point their delegations to this address. This keypair collects rewards at the end of each epoch based on the vote credits earned over that epoch in combination with your total stake.
3) withdrawal authority ID which this guide refers to as `authority-keypair.json` (but can be names whatever) and **SHOULD NOT BE LEFT ON THE MACHINE** but instead, **air-gapped** after creation. More on this in managing the Validator.

**THIS IS OPTIONAL**. There are some other IDs that are useful when setting up Validator if you want to trigger leader-slot placement prior to receiving anyone else's stake. You need to create a stake account.

4) stake account ID - referred to as `stake-account.json` located `/home/sol` and will coordinate with `authority-keypair.json` to delegate to `vote-account-keypair.json`

You will need a little bit of SOL in `validator-keypair.json` and if you want to optionally trigger leader slot placement early you will need a little bit of SOL in `stake-account.json`
#
### **IMPORTANT** - COPY DOWN YOUR SEED PHRASES! Keep them safe. Back them up (physically and more than once).   
#

### Make keys!
```
solana-keygen new --outfile ~/authority-keypair.json

solana-keygen new -o ~/validator-keypair.json

solana-keygen new -o ~/vote-account-keypair.json
```
OPTIONAL
```
solana-keygen new -o ~/stake-account.json
```
Tell Solana CLI you have a new Validator ID
```
solana config set --keypair ~/validator-keypair.json
```


Transfer 0.1 SOL into the ~/validator-keypair.json (to create vote account and at the same time set the withdrawal authority)
```
solana create-vote-account ~/vote-account-keypair.json ~/validator-keypair.json ~/authority-keypair.json

```
Set commission with the CLI. The number 10 means 10% ( you can only do whole integers at this time). You can do 0% by placing a 0 instead of 10.
```
solana vote-update-commission ~/vote-account-keypair.json 10 ~/authority-keypair.json
```

### Making system services (sol.service and systuner.service) and the startup script.   

solana-validator shell script
```
sudo nano ~/start-validator.sh
```
Edit this into start-validator.sh ( updated 02/07/2022):
```
#!/bin/bash
exec solana-validator \
--identity ~/validator-keypair.json \
--vote-account ~/vote-account-keypair.json \
--authorized-voter ~/vote-account-keypair.json \
--entrypoint entrypoint.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint2.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint3.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint4.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint5.mainnet-beta.solana.com:8001 \
--trusted-validator DDnAqxJVFo2GVTujibHt5cjevHMSE9bo8HJaydHoshdp \
--trusted-validator Ninja1spj6n9t5hVYgF3PdnYz2PLnkt7rvaw3firmjs \
--trusted-validator wWf94sVnaXHzBYrePsRUyesq6ofndocfBH6EmzdgKMS \
--trusted-validator 7cVfgArCheMR6Cs4t6vz5rfnqd56vZq4ndaBrY5xkxXy \
--expected-genesis-hash 5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d \
--ledger /mt/ledger/validator-ledger \
--dynamic-port-range 8000-8010 \
--private-rpc \
--rpc-bind-address 127.0.0.1 \
--no-untrusted-rpc \
--expected-shred-version 13490 \
--wal-recovery-mode skip_any_corrupted_record \
--log ~/solana-validator.log \
--accounts /mnt/ramdrive/solana-accounts \
--limit-ledger-size 50000000 \
```
save / exit (ctrl+s, ctrl+x)

make executable
```
sudo chmod +x ~/start-validator.sh
sudo chown sol:sol start-validator.sh
```
Create system service - sol.service (allows Solana to run on boot, auto-restart when sys fail) 
```
sudo nano /etc/systemd/system/sol.service
```
Edit this into file:
```
[Unit]
Description=Solana Validator
After=network.target
Wants=systuner.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=on-failure
RestartSec=1
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
User=sol
Environment=PATH=/bin:/usr/bin:/home/sol/.local/share/solana/install/active_release/bin
Environment=SOLANA_METRICS_CONFIG=host=https://metrics.solana.com:8086,db=mainnet-beta,u=mainnet-beta_write,p=password
WorkingDirectory=/home/sol/
ExecStart=/home/sol/start-validator.sh

[Install]
WantedBy=multi-user.target
```
save/exit (:wq)

make system tuner service - systuner.service
```
sudo nano /etc/systemd/system/systuner.service
```
Edit this into file:
```
[Unit]
Description=Solana System Tuner
After=network.target
[Service]
Type=simple
Restart=on-failure
RestartSec=1
LogRateLimitIntervalSec=0
ExecStart=/home/sol/.local/share/solana/install/active_release/bin/solana-sys-tuner --user sol
[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
```
Set up log rotation for ~/log/solana-validator.log
```
sudo nano /etc/logrotate.d/solana
```
Edit this into file:
```
/home/sol/log/solana-validator.log {
  su sol sol
  daily
  rotate 3
  missingok
  postrotate
    systemctl kill -s USR1 sol.service
  endscript
}
```
Reset log rotate
```
sudo systemctl restart logrotate
```
Start up and test (suggest opening a tmux or screen to do this so that you can tail log in the native session)
run bash first to generate log then kill bash and use service
```
tmux

bash ~/start-validator.sh

ctrl + b, d (to detach from tmux)

sudo tail -f ~/log/solana-validator.log

ctrl+c
```
if the service did not kickstart then debug, otherwise continue to make into system services:
```
sudo systemctl enable --now systuner.service
sudo systemctl status systuner.service
sudo systemctl enable --now sol.service
sudo systemctl status sol.service
```
This starts the system process and the Validator should begin running. Debug if not by tailing the log and filter for error of warnings.
```
sudo tail -f ~/log/solana-validator.log
```
### **If you have not done so already, join the Solana Discord and get help in the "validator-support" channel.**

#
Create stake account and delegate a small amount of stake in order to activate voting on the machine

In the below example the .1 is the amount of SOL being staked to the `vote-account-keypair.json` owned by your validator ID.
```
solana create-stake-account --from ~/validator-keypair.json ~/stake-account.json .1 --stake-authority ~/authority-keypair.json --withdraw-authority ~/authority-keypair.json --fee-payer ~/validator-keypair.json

```
Next delegate the small amount of stake to the validator just made. **You will need to replace the addresses in the below examples with your own addresses** If the validator still has not caught up to the cluster then it will say "unable to delegate. vote account has no root slot" you will need to wait and try again later. You might have to wait up to 45 minutes for a Validator to catch up.
```
solana delegate-stake --stake-authority ~/authority-keypair.json 8QhaDK7J5dpdpbKxHvA3pwZo6Zj7ghq4THNnmyz13ys wWf94sVnaXHzBYrePsRUyesq6ofndocfBH6EmzdgKMS --fee-payer ~/validator-keypair.json

```
You can look up your local filesystem wallet addresses using:
```
solana-keygen pubkey ~/validator-kaypair.json

solana-keygen pubkey ~/authority-keypair.json
```
Remember to use the `solana --help` feature to learn syntax. You can look up specifc commands for example:
```
solana create-stake-account --help
```
#
Watchtower Monitoring (using the Shadowy Super Coder DAO's validator ID as an example)
create webhook first in discord and then run the following in a `screen` or `tmux` detached session:

Example webhook (you will need to make your own fir this to work):
```
export DISCORD_WEBHOOK="https://discord.com/api/webhooks/123123123123123123/ljkahdfjahdflkajdflkjhadfwhateveryourwebhookhashishere"


solana-watchtower --validator-identity wWf94sVnaXHzBYrePsRUyesq6ofndocfBH6EmzdgKMS --interval 30 --minimum-validator-identity-balance 2
```
Publish your validator to chain. This example assumes your validator's name is "SomethingCreative" and that you have created a keybase.io account also names "SomethingCreative."

```
solana validator-info publish "SomethingCreative" -n SomethingCreative -w "https://www.somethingcreative.com/stake" -d "SomethingCreative is the absolute best choice by far for your staking needs. Please enjoy this detailed description of why this validator is amazing."
```

Install a monitoring stack of your choice. There are many great public guides. For example, look up Ubuntu `TIG` motoring stack how to guide. TIG stand for Telegraf, InfluxDB, Grafana.

Join the Solana Discord channel for validator support! 
