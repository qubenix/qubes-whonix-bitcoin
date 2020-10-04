# Qubes 4 & Whonix 15: Electrum Personal Server
Create a VM for running an [Electrum Personal Server](https://github.com/chris-belcher/electrum-personal-server) (EPS) which will connect to your `bitcoind` VM. The `electrum-personal-server` VM will be accessible from an Electrum Bitcoin wallet in an offline VM on the same host or remotely via a Tor onion service.
## What is Electrum Personal Server?
EPS is one of the possible server backends for the Electrum Bitcoin wallet. The other implementations covered in these guides are [Electrs](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrs.md), and [Electrumx](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md).

Here are some of the differences between the different implementations:
- Electrs and Electrumx are more versatile.
  - Electrs and Electrumx can serve any wallet once fully synchronized.
  - EPS requires that each wallet's [MPK](https://bitcoin.stackexchange.com/a/50031) is in its config file.
- Different disk space requirements.
  - Electrs and Electrumx VM disk space: under 80G.
  - EPS VM disk space: 1G.
- Different initial sync times.
  - Initial Electrs Sync: 1-12 hours.
  - Initial Electrumx Sync: 12-24 hours.
  - Initial EPS Sync: 10-20 min.
- Electrumx can be configured to be part of a p2p network of servers that support random Electrum wallet users. That setup is out of scope for these guides.

For more information see the
[mailing list email](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-February/015707.html)
and [bitcointalk thread](https://bitcointalk.org/index.php?topic=2664747.msg27179198).
## Why Do This?
This will protect you from having to trust nodes ran by volunteers to provide you with vital information and services regarding your Electrum wallet and your Bitcoin.

There have already been multiple waves of attacks on Electrum users perpetrated by malicious servers. These bad servers prevent users from sending transactions, instead sending back to them a bogus update requirement which actually leads to coin stealing malware.

This setup also preserves your privacy. When connecting to any server your wallet will leak information which can be used to tie your addresses together. There are definitely servers on the network which are using this information to build profiles on addresses and their interactions.
## Prerequisites
- Read the [README](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/README.md).
- To complete this guide you must have first completed:
  - [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md)

## I. Set Up Dom0
### A. In a `dom0` terminal, create a gateway.
**Notes:**
- This gateway should be independent of other Whonix gateways to isolate its onion service. See [here](https://www.whonix.org/wiki/Multiple_Whonix-Gateway).
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' \
--prop provides_network='True' --prop vcpus='1' --template whonix-gw-15 \
sys-electrum-personal-server
```
### B. Create AppVM.
1. Create the AppVM for Electrum Personal Server with the newly created gateway, using the `whonix-ws-15-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label red --prop maxmem='400' \
--prop netvm='sys-electrum-personal-server' --prop vcpus='1' \
--template whonix-ws-15-bitcoin electrum-personal-server
```
### C. Allow comms from `electrum-personal-server` to `bitcoind`.
```
[user@dom0 ~]$ echo 'electrum-personal-server bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.ConnectTCP
```
### D. Get IP of the `sys-electrum-personal-server` gateway.
**Note:**
- Save your gateway IP (`10.137.0.50` in this example) to replace `<gateway-ip>` in later examples.

```
[user@dom0 ~]$ qvm-prefs sys-electrum-personal-server ip
10.137.0.50
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-15-bitcoin` terminal, update and install dependency.
```
user@host:~$ sudo apt update && sudo apt install -y python-virtualenv
```
### B. Create a system user.
```
user@host:~$ sudo adduser --system electrum-personal-server
Adding system user `electrum-personal-server' (UID 117) ...
Adding new user `electrum-personal-server' (UID 117) with group `nogroup' ...
Creating home directory `/home/electrum-personal-server' ...
```
### C. Shutdown TemplateVM.
```
user@host:~$ sudo poweroff
```
## III. Set up Bitcoin.
### A. In a `bitcoind` terminal, create RPC credentials for `electrum-personal-server` to communicate with `bitcoind`.
1. Create a random RPC username. Do not use the one shown.

**Note:**
- Save your username (`7PXaFZ5DLG2alSeiGxnM` in this example) to replace `<rpc-user>` in later examples.

```
user@host:~$ head -c 15 /dev/urandom | base64
7PXaFZ5DLG2alSeiGxnM
```
2. Use Bitcoin's tool to create a random RPC password and config entry. Do not use the one shown.

**Notes:** Save the hased password (`9ffa7d78e1ddcb25ace4597bc31a1c8d$541c44f5d34044d532db47b74e9755ca4f0d87f805dd5895f0b36ea3a8d8c84c` in this example) to replace `<hashed-pass>` in later examples.
- Save your password (`GKkkKy-GAEDUw_6dp32O7Rh3DhHAnYhBUwNwNWUZPrI=` in this example) to replace `<rpc-pass>` in later examples.
- Replace `<rpc-user>` with the information noted earlier.

```
user@host:~$ sudo -u bitcoin /home/bitcoin/bitcoin/share/rpcauth/rpcauth.py <rpc-user>
String to be appended to bitcoin.conf:
rpcauth=7PXaFZ5DLG2alSeiGxnM:9ffa7d78e1ddcb25ace4597bc31a1c8d$541c44f5d34044d532db47b74e9755ca4f0d87f805dd5895f0b36ea3a8d8c84c
Your password:
GKkkKy-GAEDUw_6dp32O7Rh3DhHAnYhBUwNwNWUZPrI=
```
### B. Configure `bitcoind`.
1. Open `bitcoin.conf`.

```
user@host:~$ sudo -u bitcoin mousepad /home/bitcoin/.bitcoin/bitcoin.conf
```
2. Paste the following at the bottom of the file.

**Notes:**
- Be sure not to alter any of the existing information.
- Replace `<rpc-user>` and `<hashed-pass>` with the information noted earlier.

```
# Electrum Personal Server Auth
rpcauth=<rpc-user>:<hashed-pass>
# Electrum Personal Server Wallet
wallet=electrum-personal-server
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```
## IV. Install Electrum Personal Server
### A. In an `electrum-personal-server` terminal, change to `electrum-personal-server` user.
1. Switch to user `electrum-personal-server` and change to home directory.

```
user@host:~$ sudo -H -u electrum-personal-server bash
electrum-personal-server@host:/home/user$ cd
```
### B. Download and verify the Electrum Personal Server source code.
1. Download the latest Electrum Personal Server [release and signature](https://github.com/chris-belcher/electrum-personal-server/releases).

**Note:** At the time of writing the most recent version of Electrum Personal Server is `v0.2.1.1`, modify the following steps accordingly if the version has changed.

```
electrum-personal-server@host:~$ scurl-download \
https://github.com/chris-belcher/electrum-personal-server/archive/eps-v0.2.1.1.tar.gz \
https://github.com/chris-belcher/electrum-personal-server/releases/download/eps-v0.2.1.1/eps-v0.2.1.1.tar.gz.asc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612    0   612    0     0    194      0 --:--:--  0:00:03 --:--:--   194
100   819  100   819    0     0    154      0  0:00:05  0:00:05 --:--:--   390
curl: Saved to filename 'eps-v0.2.0.tar.gz.asc'
100   150    0   150    0     0    374      0 --:--:-- --:--:-- --:--:--   374
100 80371    0 80371    0     0  23666      0 --:--:--  0:00:03 --:--:-- 36935
curl: Saved to filename 'electrum-personal-server-eps-v0.2.0.tar.gz'
```
2. Receive signing key.

**Note:** You can verify the key fingerprint in the [release notes](https://github.com/chris-belcher/electrum-personal-server/releases).

```
electrum-personal-server@host:~$ gpg --recv-keys "0A8B 038F 5E10 CC27 89BF CFFF EF73 4EA6 77F3 1129"
gpg: keybox '/home/electrum-personal-server/.gnupg/pubring.kbx' created
gpg: key 0xEF734EA677F31129: 5 signatures not checked due to missing keys
gpg: /home/electrum-personal-server/.gnupg/trustdb.gpg: trustdb created
gpg: key 0xEF734EA677F31129: public key "Chris Belcher <false@email.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify source code.

**Note:** Your output may not match the example. Just check that it says `Good signature`.

```
electrum-personal-server@host:~$ gpg --verify eps-v0.2.1.1.tar.gz.asc \
electrum-personal-server-eps-v0.2.1.1.tar.gz
gpg: Signature made Tue 09 Jun 2020 01:32:21 PM UTC
gpg:                using RSA key 0xEF734EA677F31129
gpg: Good signature from "Chris Belcher <false@email.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 0A8B 038F 5E10 CC27 89BF  CFFF EF73 4EA6 77F3 1129
```
### C. Install Electrum Personal Server.
1. Make `eps` directory.

```
electrum-personal-server@host:~$ mkdir ~/eps
```
2. Extract and enter directory.

```
electrum-personal-server@host:~$ tar -C ~/eps/ \
-xf electrum-personal-server-eps-v0.2.1.1.tar.gz \
--strip-components=1
```
3. Create virtual environment.

```
electrum-personal-server@host:~$ virtualenv -p python3 ~/epsvenv
Already using interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/electrum-personal-server/epsvenv/bin/python3
Also creating executable in /home/electrum-personal-server/epsvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
4. Source the virtual environment.

```
electrum-personal-server@host:~$ source ~/epsvenv/bin/activate
```
5. Enter the EPS directory, install EPS, and deactivate virtual environment.

```
(epsvenv) electrum-personal-server@host:~$ cd ~/eps/
(epsvenv) electrum-personal-server@host:~/eps$ python setup.py install
(epsvenv) electrum-personal-server@host:~/eps$ deactivate
```
6. Return to home directory.

```
(epsvenv) electrum-personal-server@host:~/eps$ cd
```
## V. Set Up EPS
### A. Remain in an `electrum-personal-server` terminal, configure EPS data directory.
1. Create EPS's data directory and subdirectories.

```
electrum-personal-server@host:~$ mkdir -m 0700 ~/.eps
electrum-personal-server@host:~$ mkdir -m 0700 ~/.eps/certs
```
2. Create configuration file.

```
electrum-personal-server@host:~$ mousepad ~/.eps/config.cfg
```
3. Paste the following.

**Notes:**
- Be sure to replace `<rpc-user>` and `<rpc-pass>` with the information noted earlier.
- For a verbose desciption of these settings, look to the file: [`~/eps/config.ini_sample`](https://github.com/chris-belcher/electrum-personal-server/blob/master/config.ini_sample).
- At this point you may add your Electrum wallet master public keys (MPK) or individual addresses to the config file.

```
## Electrum Personal Server configuration file

[master-public-keys]
## Add electrum master public keys to this section.
## Create a wallet in electrum then go Wallet -> Information to get the MPK.

# any_name_works = xpub661MyMwAqRbcFseXCwRdRVkhVuzEiskg4QUp5XpUdNf2uGXvQmnD4zcofZ1MN6Fo8PjqQ5cemJQ39f7RTwDVVputHMFjPUn8VRp2pJQMgEF

## Multiple master public keys maybe added by simply adding another line.
# my_second_wallet = xpubanotherkey

## Multisig wallets use format `required-signatures [list of master pub keys]`.
# multisig_wallet = 2 xpub661MyMwAqRbcFseXCwRdRVkhVuzEiskg4QUp5XpUdNf2uGXvQmnD4zcofZ1MN6Fo8PjqQ5cemJQ39f7RTwDVVputHMFjPUn8VRp2pJQMgEF xpub661MyMwAqRbcFseXCwRdRVkhVuzEiskg4QUp5XpUdNf2uGXvQmnD4zcofZ1MN6Fo8PjqQ5cemJQ39f7RTwDVVputHMFjPUn8VRp2pJQMgEF

[watch-only-addresses]
## Add addresses to this section, space separated list.
## Multiple `addr =` entries allowed.
# addr = 1DuqpoeTB9zLvVCXQG53VbMxvMkijk494n

[bitcoin-rpc]
host = 127.0.0.1
port = 8332
rpc_user = <rpc-user>
rpc_password = <rpc-pass>
poll_interval_listening = 30
poll_interval_connected = 5
initial_import_count = 1000
gap_limit = 25

[electrum-server]
host = 0.0.0.0
port = 50002
ip_whitelist = *
certfile = /home/electrum-personal-server/.eps/certs/server.crt
keyfile = /home/electrum-personal-server/.eps/certs/server.key
broadcast_method = own-node

[logging]
log_level_stdout = DEBUG
log_file_location = /home/electrum-personal-server/.eps/debug.log
append_log = false
log_format = %(levelname)s:%(asctime)s: %(message)s
```
4. Save the file: `Ctrl-S`.
5. Switch back to the terminal: `Ctrl-Q`.
6. Fix permissions.

```
electrum-personal-server@host:~$ chmod 0600 /home/electrum-personal-server/.eps/config.cfg
```
### B. Create certificate.
```
electrum-personal-server@host:~$ openssl req -x509 -sha256 -newkey rsa:4096 \
-keyout ~/.eps/certs/server.key -out ~/.eps/certs/server.crt -days 1825 \
-nodes -subj '/CN=localhost'
Generating a RSA private key
.........................................................................................++++
....++++
writing new private key to '/home/electrum-personal-server/.eps/certs/server.key'
-----
```
### C. Change back to original user.
```
electrumx@host:~$ exit
```
### D. Use `systemd` to keep `electrum-personal-server` running.
1. Create a persistent directory.

```
user@host:~$ sudo mkdir -m 0700 /rw/config/systemd
```
2. Create the service file.

```
user@host:~$ lxsu mousepad /rw/config/systemd/electrum-personal-server.service
```
3. Paste the following.

```
[Unit]
Description=Electrum Personal Server

[Service]
ExecStart=/home/electrum-personal-server/epsvenv/bin/electrum-personal-server \
    /home/electrum-personal-server/.eps/config.cfg

User=electrum-personal-server
Restart=on-failure

PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```
4. Save the file: `Ctrl-S`.
5. Switch back to the terminal: `Ctrl-Q`.
6. Fix permissions.

```
user@host:~$ sudo chmod 0600 /rw/config/systemd/electrum-personal-server.service
```
### E. Enable the service on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ lxsu mousepad /rw/config/rc.local
```
2. Paste the following at the bottom of the file.

```
cp /rw/config/systemd/electrum-personal-server.service /lib/systemd/system/
systemctl daemon-reload
systemctl start electrum-personal-server.service
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
## VI. Fix Networking
### A. Remain in an `electrum-personal-server` terminal, open communication with `bitcoind` on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo sh -c 'echo "\nqvm-connect-tcp 8332:bitcoind:8332" >> /rw/config/rc.local'
```
2. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### C. Open firewall for Tor onion service.
1. Make persistent directory.

```
user@host:~$ sudo mkdir -m 0755 /rw/config/whonix_firewall.d
```
2. Configure firewall.

```
user@host:~$ sudo sh -c 'echo "EXTERNAL_OPEN_PORTS+=\" 50002 \"" >> /rw/config/whonix_firewall.d/50_user.conf'
```
3. Restart firewall service.

```
user@host:~$ sudo systemctl restart whonix-firewall.service
```
### D. Find out the IP address of the `electrum-personal-server` VM.
**Note:** Save the `electrum-personal-server` IP (`10.137.0.51` in this example) to replace `<eps-ip>` in later examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.51
```
## VII. Initial EPS Synchronization
### A. In an `electrum-personal-server` terminal, start the `electrum-personal-server` service.
```
user@host:~$ sudo systemctl start electrum-personal-server.service
```
## VIII. Set Up Gateway.
### A. In a `sys-electrum-personal-server` terminal, set up Tor.
1. Edit the Tor configuration file.

```
user@host:~$ lxsu mousepad /rw/usrlocal/etc/torrc.d/50_user.conf
```
2. Paste the following.

**Note:** Be sure to replace `<eps-ip>` with the information noted earlier.

```
HiddenServiceDir /var/lib/tor/electrum-personal-server/
HiddenServicePort 50002 <eps-ip>:50002
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Reload `tor`.

```
user@host:~$ sudo systemctl reload tor.service
```
6. Find out your onion hostname.

**Note:** Make a note of your server hostname for use with your remote Electrum wallet.

```
user@host:~$ sudo cat /var/lib/tor/electrum-personal-server/hostname
electrumpersonalservertoronionserviceaddressxxxxxxxxxxxx.onion
```
## VIII. Final Notes
- Initial sync will take about 10-20 minutes for each MPK in your EPS config file.
- Once the sync is finished you may connect your Electrum wallet via the Tor onion address.
- To check the status of the server:
  - For an `INFO` level log: `sudo journalctl -fu electrum-personal-server`
  - For a `DEBUG` level log: `sudo tail -f /home/electrum-personal-server/.eps/debug.log`
- To connect an offline Electrum wallet from a separate VM (split-electrum) use the guide: [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md).
