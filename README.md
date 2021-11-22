# Lukso Validator Node

This is a guide to setup a Lukso validator node in home environment. The guide suggests a use of a dedicated machine to run a node with validators, separate from a personal working machine.

> **_NOTE:_** Most of the steps require working in a terminal

> **_NOTE:_** This is a guide for L15 test net

## Prerequisites

- [Ubuntu](https://ubuntu.com/) server or desktop installed
- A personal computer with Unix like OS (Mac OS, Linux, etc.)

## System Setup

> **_NOTE:_** Following steps are performed directly on a node machine.

In order to remotelly access a machine running a node, it needs to be configured.

### Update

```shell=
sudo apt update
sudo apt upgrade -y
sudo apt install -y vim
```

### Remote Access

SSH is used to enable remote access from other machine using localy network through WiFi or broadband connections. This is a common practice and can be quite useful if a node machine does not have input (keyboard/mouse) nor a display. Once setup, a node machine can be placed elsewhere and only personal computer could be used to control and maintain it.

#### Install SSH

```shell=
sudo apt install --assume-yes openssh-server
```

#### Confiugre SSH

Choose a port number larger than `50000`. This will be used later.

```shell=
sudo vim /etc/ssh/sshd_config
```

Change and enable a port by uncommenting (removing `#`) and changing `22` to new chosen port number:
```shell=
Port 50000
```

Save and close editor by pressing `SHIFT` + `:`, then type `wq`, and hit enter.

#### Configure Firewall

Enable ssh in firewall by replacing *replace-port* with new port:

```shell=
sudo ufw allow replace-port
```

#### Enable SSH

```shell=
sudo systemctl start ssh
sudo systemctl enable ssh
```

#### Resolve Hostname

In order to locate a node machine in local network, it requires either IP address or a local host name. Execute following command to resolve a node machine's host name.

```shell=
hostname
```

The host name would be a result of above command appended with `.local`. E.g. if a machine has been called `lukso`, hostname would return `lukso`, thus actual host name is `lukso.local`.

Close ssh session by executing `exit`.

### Remote Access

> **_NOTE:_** Following steps are performed a personal computer.

Verify basic access to a node machine by using ssh. SSH requires user name of a node machine, its hostname and previously chosen ssh port.

```shell=
vim ~/.ssh/config 
```

Type in the following and replace *replace-user*, *replace-hostname*, and *replace-port*:

```shell=
Host lukso
  User replace-user
  HostName replace-hostname.local
  Port replace-port
```

Attempt to connect to verify the configuration:

```shell=
ssh lukso
```

Once connected, enter a password of user on a node machine. If a connection was okay, a shell should be presented in a terminal. At this point, it could closed.

#### Disable Password Authentication

On a personal computer, create new key pair for ssh authentication if needed.

```shell=
ssh-keygen -t rsa -b 4096
```

Copy a generated public key **keyname.pub** to a node machine. Replace **keyname.pub** with a key in home directory.

```shell=
ssh-copy-id -i ~/.ssh/keyname.pub lukso
```

#### Disable Non-Key Remote Access

On a personal computer, try to ssh again. This time it should not prompt for a password.

```shell=
ssh lukso
```

Configure SSH by opening a configuration file and modifying several options:

```shell=
sudo vim /etc/ssh/sshd_config
```

Options:

```shell=
ChallengeResponseAuthentication no
PasswordAuthentication no
PermitRootLogin prohibit-password
PermitEmptyPasswords no
```

Save and close editor by pressing `SHIFT` + `:`, then type `wq`, and hit enter. Validate SSH configuration and restart ssh service.

```shell=
sudo sshd -t
sudo systemctl restart sshd
```

Close ssh session by executing `exit`.

#### Verify Remote Access

```shell=
ssh lukso
```
    
Stay connected to a remote node machine to perform next steps.

### Keep System Up to Date
    
Update a system manually:
    
```shell=
sudo apt-get update -y
sudo apt dist-upgrade -y
sudo apt-get autoremove
sudo apt-get autoclean
```

Keep a system up to date automatically:

```shell=
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Disable Root Access
    
A root access should not be used. Instead, a user should be using `sudo` to perform privilged operations on a system.

```shell=
sudo passwd -l root
```

### Block Unathorised Access

Install `fail2ban` to block IP addresses that exceed failed ssh login attempts.

```shell=
sudo apt-get install fail2ban -y
```

Edit a config to monitor ssh logins

```shell=
sudo vim /etc/fail2ban/jail.local
```

Replace *replace-port* to match the ssh port number.

```shell=
[sshd]
enabled=true
port=replace-port
filter=sshd
logpath=/var/log/auth.log
maxretry=3
ignoreip=
```

Save and close editor by pressing `SHIFT` + `:`, then type `wq`, and hit enter. Restart `fail2ban` service:

```shell=
sudo systemctl restart fail2ban
```

### Confiugre Firewall

By default deny all traffic:

```shell=
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow P2P ports for Lukso client:

```shell=
sudo ufw allow 13000/tcp
sudo ufw allow 12000/udp
```

Enable Firewall:

```shell=
sudo ufw enable
```

Verify firewall configuration:

```shell=
sudo ufw status numbered
```

It should look something like this:

```shell=
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 13000/tcp                  ALLOW IN    Anywhere               
[ 2] ssh-port/tcp               ALLOW IN    Anywhere               
[ 3] 12000/udp                  ALLOW IN    Anywhere               
[ 4] 9090/tcp                   ALLOW IN    Anywhere              
[ 5] 3000/tcp                   ALLOW IN    Anywhere               
[ 6] 13000/tcp (v6)             ALLOW IN    Anywhere (v6)          
[ 7] ssh-port/tcp (v6)          ALLOW IN    Anywhere (v6)          
[ 8] 12000/udp (v6)             ALLOW IN    Anywhere (v6)          
[ 9] 9090/tcp (v6)              ALLOW IN    Anywhere (v6)          
[10] 3000/tcp (v6)              ALLOW IN    Anywhere (v6)
```

> **_NOTE:_** 9090 and 3000 ports are for grafana configuration. This will be revisited later in the setup.

### Improve SSH Connection

While setting up a system, ssh terminal may seem to be slow due wifi power management settings on a node machine. To disable it, modify a config.

```shell=
sudo vim /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
```

Config:
```shell=
[connection]
wifi.powersave = 2
```

Save and close editor by pressing `SHIFT` + `:`, then type `wq`, and hit enter. Restart `NetworkManager` service:

```shell=
sudo systemctl restart NetworkManager
```

## Node Setup

> **_NOTE:_** Following steps are performed on personal machine.

Access a remote node machine

```shell=
ssh lukso
```

### Install Lukso CLI

```shell=
curl https://install.l15.lukso.network | bash
```

### Create Wallet

Prepare a working folder

```shell=
mkdir -p ~/node/l15-prod/vanguard_wallet/
```

#### Password

[Generate a password](https://passwordsgenerator.net/) containing numbers, letters, and special symbols (`%`, `!`, etc). Save password locally:

```shell=
echo 'insert-password-here' > ~/node/l15-prod/vanguard_wallet/password
```

#### Deposit Keys

Generate new deposit keys (mnemonic)

```shell=
lukso-deposit-cli new-mnemonic
```

Follow step by step guide:
1. Choose a language (default is English)
2. Choose a number of validator to run. More validators can be added later
3. Type `l15-prod` for network/chain name
4. Type the password twice to confirm
5. Take a note of mnemonic (24 words). **Do not save them locally, store it offline**. That being said, it is okay for L15 as this is a test network
6. Hit enter and type all 24 words to confirm

Import validator keys into a wallet:

```shell=
mv validator_keys ~/node/l15-prod/
```

#### Import Wallet

Import validator keys to a wallet:

```shell=
lukso wallet --wallet-password-file ~/node/l15-prod/vanguard_wallet/password
```

Follow guide:
1. Deposit keys: `~/node/l15-prod/validator_keys`
2. Store wallet: `~/node/l15-prod/vanguard_wallet`
3. Password: previously generated password

#### Copy Deposit Data

Exit ssh session to fetch deposit data. On a local machine fetch validator_keys folder containing deposit data json files. Replace *username* as needed:

```shell=
scp -r lukso:/home/username/node/l15-prod/validator_keys/ ./
```

In the work directory on a local machine, there should be `validator_keys` directory containing json files looking like `deposit_data-1636138343.json`.

#### Metamask

[Install Metamask](https://metamask.io/) to create a depositor wallet (address). Create a wallet following Metamask guide.

##### L15 Network

In Metamask go to `Settings` > `Networks` > `Add Network`. Configure it as following:
- Network Name: `LUKSO L15`
- New RPC URL: `https://rpc.l15.lukso.network`
- Chain ID: `23`
- Currency Symbol: `LYXt`
- Block Explorer URL: `https://explorer.pandora.l15.lukso.network/`

#### Fund Wallet

Take a note of address in Metamask. Proceed to a [facuet](https://faucet.l15.lukso.network/) to fund this wallet with testnet LYXt.

#### Deposit LYXt

In order to run validators on Lukso network, a deposit(s) of LYX must be made. To do so proceed to [launchpad](https://launchpad.l15.lukso.network/) and fund it with Metamask wallet. In the guide, it will instruct to deposit data json files which can be located in `validator_keys` folder from earlier steps. Make sure to use same number of validator set when the deposit keys were generated.

### Configure Node

Ssh to a node machine:
```shell=
ssh lukso
```

#### Start Node

Prepare scripts to start and stop a node with validators

```shell=
sudo vim /usr/local/bin/lukso-start
```

Following changes are needed:
1. Coinbase: a wallet address from Metamask which deposited LYX
2. Node name: a node name of a choice
3. Replace *username*

```shell=
#!/bin/bash

lukso start \
	--validate \
	--coinbase "depositor-wallet-address" \
	--node-name "l15-node" \
	--wallet-dir /home/username/node/l15-prod/vanguard_wallet \
	--wallet-password-file /home/username/node/l15-prod/vanguard_wallet/password \
	--datadir /home/username/node/l15-prod/data \
	--logsdir /home/username/node/l15-prod/logs
```

Save and close editor by pressing `SHIFT` + `:`, then type `wq`, and hit enter. Prepare a stop script

```shell=
sudo vim /usr/local/bin/lukso-stop
```

With content:

```shell=
#!/bin/bash

lukso stop
```

#### Node Service

Create a system service to control a lukso client. This is useful to auto start a lukso client or restart if it crashes.

```shell=
sudo vim /etc/systemd/system/lukso.service
```

Provide description and replace *username* with correct name.

> To find a group of a *username* execute: `groups username`

```shell=
[Unit]
Description=Lukso node and validators
After=network.target network-online.target

[Service]
User=username
Group=group
Type=forking
ExecStart=/usr/local/bin/lukso-start
ExecStop=/usr/local/bin/lukso-stop
TimeoutSec=30
Restart=on-failure
RestartSec=30
StartLimitInterval=350
StartLimitBurst=10

[Install]
WantedBy=multi-user.target
```

Enable node service

```shell=
sudo systemctl daemon-reload
sudo systemctl start lukso
```

Verify service status:
```shell=
sudo systemctl status lukso
```

It should print green indicator to signal active status and contain following message: `Active: active (running)`.

Enable and restart or stop service as needed:
```shell=
sudo systemctl stop lukso
sudo systemctl enable lukso
sudo systemctl restart lukso
```

> Verify service auto-start by rebooting node machine, ssh, and poll status on lukso service to see it being active and running.

> Verify a node machine can auto start when there is a power outage. If not, most likely BIOS settings needs to tweaked for the machine to enable this option.
