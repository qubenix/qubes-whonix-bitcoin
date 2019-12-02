# Qubes 4 & Whonix 15: JoinMarket
Create a VM without networking to host a [JoinMarket](https://github.com/JoinMarket-Org/joinmarket-clientserver) wallet. The `joinmarketd` daemon will run on the `bitcoind` VM, communicate only over Tor onion services, and use stream isolation.

The offline `joinmarket` VM will communicate with your `bitcoind` VM using Qubes' [`qrexec`](https://www.qubes-os.org/doc/qrexec3/).
## What is JoinMarket?
Joinmarket is an open source, decentralized, and trustless market for [coinjoins](https://en.bitcoin.it/wiki/CoinJoin). Anyone holding Bitcoin can offer coinjoins for a fee, and anyone can pay a fee to have their transactions obfuscated.

See the [Bitcoin wiki](https://en.bitcoin.it/wiki/JoinMarket) for a more detailed explanation of JoinMarket.
## Why Do This?
This increases the privacy and security of your JoinMarket wallet while still maintaining full functionality. Enhanced privacy is achieved by using only Tor onion endpoints, and security is improved by removing the need for a network connection on the wallet VM.
## Prerequisites
- To complete this guide you must have first completed:
  - [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md)

## I. Set Up Dom0
### A. In a `dom0` terminal, create an AppVM.
1. Create the AppVM for JoinMarket's wallet with no networking, using the `whonix-ws-15-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label black --prop maxmem='800' --prop netvm='' \
--prop vcpus='1' --template whonix-ws-15-bitcoin joinmarket
```
### B. Allow comms from `joinmarket` to `bitcoind` VM.
```
[user@dom0 ~]$ echo 'joinmarket bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.ConnectTCP
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-15-bitcoin` terminal, update and install dependencies.
```
user@host:~$ sudo apt update && sudo apt install -y libffi-dev libgmp-dev libsecp256k1-dev \
python-configparser python-virtualenv python3-argon2 python3-cffi python3-dev python3-future \
python3-libnacl python3-matplotlib python3-mnemonic python3-pip python3-pyaes python3-qrcode
```
### B. Create system user.
```
user@host:~$ sudo adduser --system joinmarket
Adding system user `joinmarket' (UID 117) ...
Adding new user `joinmarket' (UID 117) with group `nogroup' ...
Creating home directory `/home/joinmarket' ...
```
### C. Shutdown TemplateVM.
```
user@host:~$ sudo poweroff
```
## III. Install JoinMarket
### A. In a `bitcoind` terminal, download and verify JoinMarket source code.
1. Clone the JoinMarket [repository](https://github.com/JoinMarket-Org/joinmarket-clientserver).

**Note:** At the time of writing the current JoinMarket [release](https://github.com/JoinMarket-Org/joinmarket-clientserver/releases) is `v0.6.0`, modify the following steps accordingly if the version has changed.

```
user@host:~$ git clone --branch v0.6.0 \
https://github.com/JoinMarket-Org/joinmarket-clientserver ~/joinmarket-clientserver
Cloning into '/home/user/joinmarket-clientserver'...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (29/29), done.
remote: Total 5238 (delta 9), reused 14 (delta 3), pack-reused 5206
Receiving objects: 100% (5238/5238), 3.85 MiB | 156.00 KiB/s, done.
Resolving deltas: 100% (3434/3434), done.
Note: checking out 'df3712f662c03db4b353e4c1c92d901153c5b8f0'.
```
2. Receive signing key.

**Note:** You can verify the key fingerprint in the [release notes](https://github.com/JoinMarket-Org/joinmarket-clientserver/releases).

```
user@host:~$ gpg --recv-keys "2B6F C204 D9BF 332D 062B 461A 1410 01A1 AF77 F20B"
gpg: key 0x141001A1AF77F20B: 2 signatures not checked due to missing keys
gpg: key 0x141001A1AF77F20B: public key "Adam Gibson (CODE SIGNING KEY) <ekaggata@gmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Enter directory and verify.

**Note:** Your output may not match the example. Just check that it says `Good signature`.

```
user@host:~$ cd ~/joinmarket-clientserver/
user@host:~/joinmarket-clientserver$ git verify-tag v0.6.0
gpg: Signature made Sat 30 Nov 2019 04:41:03 PM UTC
gpg:                using RSA key 2B6FC204D9BF332D062B461A141001A1AF77F20B
gpg: Good signature from "Adam Gibson (CODE SIGNING KEY) <ekaggata@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 2B6F C204 D9BF 332D 062B  461A 1410 01A1 AF77 F20B
```
### B. Install JoinMarket dependencies.
1. Create Python virtual environment.

```
user@host:~/joinmarket-clientserver$ virtualenv -p python3 ~/joinmarket-clientserver/jmvenv
Already using interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/user/joinmarket-clientserver/jmvenv/bin/python3
Also creating executable in /home/user/joinmarket-clientserver/jmvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```

2. Link installed packages to virtual environment.

**Note:** Ignore the `File exists` errors.

```
user@host:~/joinmarket-clientserver$ ln -s /usr/lib/python3/dist-packages/* ~/joinmarket-clientserver/jmvenv/lib/python3.7/site-packages/
```
3. Install dependencies to virtual environment.

**Notes:**
- This step, and the next optional step, will produce a lot of output and take some time. This is normal, be patient.
- If you have errors re-run the `setupall.py` step a second time.

```
user@host:~/joinmarket-clientserver$ source ~/joinmarket-clientserver/jmvenv/bin/activate
(jmvenv) user@host:~/joinmarket-clientserver$ python ~/joinmarket-clientserver/setupall.py --all
```
4. Upgrade `service-identity` to avoid an annoying warning.

```
(jmvenv) user@host:~/joinmarket-clientserver$ pip install service-identity --upgrade
```
#### Optional Step: Install QT dependencies for JoinMarket GUI.
**Note:** You can safely skip this step if you do not intend to use the JoinMarket GUI.

```
(jmvenv) user@host:~/joinmarket-clientserver$ pip install PySide2 \
https://github.com/sunu/qt5reactor/archive/58410aaead2185e9917ae9cac9c50fe7b70e4a60.zip
```
5. Deactivate virtual environment and make relocatable.

```
(jmvenv) user@host:~/joinmarket-clientserver$ deactivate
user@host:~/joinmarket-clientserver$ virtualenv -p python3 --relocatable ~/joinmarket-clientserver/jmvenv
```
6. Return to home directory.

```
user@host:~/joinmarket-clientserver$ cd
```
### C. Relocate `joinmarket-clientserver/` directory.
1. Copy `joinmarket-clientserver/` directory to the `joinmarket` user's home directory, change ownership.

```
user@host:~$ sudo cp -r ~/joinmarket-clientserver/ /home/joinmarket/
user@host:~$ sudo chown -R joinmarket:nogroup /home/joinmarket/
```
2. Copy `joinmarket-clientserver/` directory to the `joinmarket` VM.

**Note:** Select `joinmarket` from the `dom0` pop-up.

```
user@host:~$ qvm-copy ~/joinmarket-clientserver/
```
### D. Start `joinmarketd` service.
```
user@host:~$ sudo systemctl start joinmarketd.service
```
## IV. Configure `bitcoind` and `joinmarketd`
### A. In a `sys-bitcoin` terminal, find out the gateway IP.
**Note:** Save your gateway IP (`10.137.0.50` in this example) to replace `<gateway-ip>` in later examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
### B. In a `bitcoind` terminal, create RPC credentials for JoinMarket to communicate with `bitcoind`.
1. Create a random RPC username. Do not use the one shown.

**Note:** Save your username (`uJDzc07zxn5riJDx7N5m` in this example) to replace `<rpc-user>` in later examples.

```
user@host:~$ head -c 15 /dev/urandom | base64
uJDzc07zxn5riJDx7N5m
```
2. Use Bitcoin's tool to create a random RPC password and config entry. Do not use the one shown.

**Notes:**
- Save the hased password (`838c4dd74606918f1f27a5a2a52b168$9634018b87451bca05082f51b0b5b876fc72ef877bad98298e97e277abd5f90c` in this example) to replace `<hashed-pass>` in later examples.
- Save your password (`IuziNnTsOUkonsDD3jn5WatPnFrFOMSnGUsRSUaq5Qg=` in this example) to replace `<rpc-pass>` in later examples.
- Replace `<rpc-user>` with the information noted earlier.

```
user@host:~$ ~/bitcoin/share/rpcauth/rpcauth.py <rpc-user>
String to be appended to bitcoin.conf:
rpcauth=uJDzc07zxn5riJDx7N5m:838c4dd74606918f1f27a5a2a52b168$9634018b87451bca05082f51b0b5b876fc72ef877bad98298e97e277abd5f90c
Your password:
IuziNnTsOUkonsDD3jn5WatPnFrFOMSnGUsRSUaq5Qg=
```
### C. Configure `bitcoind`.
1. Open `bitcoin.conf`.

```
user@host:~$ sudo -u bitcoin mousepad /home/bitcoin/.bitcoin/bitcoin.conf
```
2. Paste the following at the bottom of the file.

**Notes:**
- Be sure not to alter any of the existing information.
- Replace `<rpc-user>` and `<hashed-pass>` with the information noted earlier.

```
# JoinMarket Auth
rpcauth=<rpc-user>:<hashed-pass>
# JoinMarket Wallet
wallet=joinmarket
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```
### D. Use `systemd` to keep `joinmarketd` running.
1. Create the service file.

```
user@host:~$ lxsu mousepad /rw/config/systemd/joinmarketd.service
```
2. Paste the following.

```
[Unit]
Description=JoinMarket daemon

[Service]
WorkingDirectory=/home/joinmarket/joinmarket-clientserver
ExecStart=/bin/sh -c 'jmvenv/bin/python scripts/joinmarketd.py'

User=joinmarket
Type=idle
Restart=on-failure

PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Fix permissions.

```
user@host:~$ chmod 0600 /rw/config/systemd/joinmarketd.service
```
### E. Enable the service on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ lxsu mousepad /rw/config/rc.local
```
2. Paste the following at the bottom of the file.

```
cp /rw/config/systemd/joinmarketd.service /lib/systemd/system/
systemctl daemon-reload
systemctl start joinmarketd.service
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
## VI. Configure `joinmarket` VM
### A. In a `joinmarket` terminal, open communication ports on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ lxsu mousepad /rw/config/rc.local
```
2. Paste the following at the bottom of the file.

```
qvm-connect-tcp 8332:bitcoind:8332
qvm-connect-tcp 27183:bitcoind:27183
qvm-connect-tcp 27184:bitcoind:27184
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### B. Move copied JoinMarket directory to your home directory.
```
user@host:~$ mv ~/QubesIncoming/bitcoind/joinmarket-clientserver/ ~
```
### C. In a `joinmarket` terminal, configure JoinMarket.
1. Source the virtual environment and change to the JoinMarket `scripts/` directory.

```
user@host:~$ source ~/joinmarket-clientserver/jmvenv/bin/activate
(jmvenv) user@host:~$ cd ~/joinmarket-clientserver/scripts/
```
2. Generate a configuration file.

```
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ python wallet-tool.py
Created a new `joinmarket.cfg`. Please review and adopt the settings and restart joinmarket.
```
3. Make a backup and edit the file `joinmarket.cfg`.

```
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ cp joinmarket.cfg joinmarket.cfg.orig
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ echo > joinmarket.cfg
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ mousepad joinmarket.cfg
```
4. Paste the following.

**Notes:**
- Be sure to replace `<gateway-ip>`, `<rpc-user>`, and `<rpc-pass>` with the information noted earlier.
- For verbose desciptions of these setting, look to the original config file: `~/joinmarket-clientserver/scripts/joinmarket.cfg.orig`.

```
[DAEMON]
no_daemon = 0
daemon_port = 27183
daemon_host = 127.0.0.1
use_ssl = false

[BLOCKCHAIN]
blockchain_source = bitcoin-rpc
network = mainnet
rpc_host = 127.0.0.1
rpc_port = 8332
rpc_user = <rpc-user>
rpc_password = <rpc-pass>
rpc_wallet_file = joinmarket

[MESSAGING:CyberguerrillaIRC]
channel = joinmarket-pit
host = epynixtbonxn4odv34z4eqnlamnpuwfz6uwmsamcqd62si7cbix5hqad.onion
port = 6697
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9180
usessl = true

[MESSAGING:AgoraAnarplexIRC]
channel = joinmarket-pit
host = agora.anarplex.net
port = 14716
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9181
usessl = true

[MESSAGING:DarkScienceIRC]
channel = joinmarket-pit
host = darksci3bfoka7tw.onion
port = 6697
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9182
usessl = true

[LOGGING]
console_log_level = INFO
color = true

[TIMEOUT]
maker_timeout_sec = 45
unconfirm_timeout_sec = 180
confirm_timeout_hours = 6

[POLICY]
segwit = true
native = false
merge_algorithm = default
tx_fees = 3
absurd_fee_per_kb = 350000
tx_broadcast = self
minimum_makers = 2
taker_utxo_retries = 3
taker_utxo_age = 5
taker_utxo_amtpercent = 20
accept_commitment_broadcasts = 1
```
4. Save the file: `Ctrl-S`.
5. Close the editor: `Ctrl-Q`.

## VI. Final Notes
- Once `bitcoind` has finished syncing in the `bitcoind` VM you will be able to use JoinMarket's wallet from the `joinmarket` VM.

## VII. Optional Steps
### A. In a `joinmarket` terminal, source virtual envrionment and change to JoinMarket's `scripts/` directory on boot.
1. Edit the file `~/.bashrc`.

```
user@host:~$ mousepad ~/.bashrc
```
2. Paste the following at the bottom of the file.

```
source /home/user/joinmarket-clientserver/jmvenv/bin/activate
cd /home/user/joinmarket-clientserver/scripts/
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Source the file to take effect.

```
user@host:~$ source ~/.bashrc
```
