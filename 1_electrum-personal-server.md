# Qubes 4 & Whonix 15: Electrum Personal Server
Create a VM for running an [Electrum Personal Server](https://github.com/chris-belcher/electrum-personal-server) (EPS) which will connect to your `bitcoind` VM. The `electrum-personal-server` VM will be accessible from an Electrum Bitcoin wallet in an offline VM on the same host or remotely via a Tor onion service.
## What is Electrum Personal Server?
EPS is one of the possible server backends for the Electrum Bitcoin wallet. The other implementation covered in these guides is [Electrumx](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md).

Here are some of the differences between EPS and Electrumx:
- EPS is less versatile.
  - EPS requires that each wallet's [MPK](https://bitcoin.stackexchange.com/a/50031) is in its config file.
  - Electrumx can serve any wallet once fully sync'd.
- An EPS VM requires less disk space.
  - EPS VM disk space: 1G
  - Electrumx VM disk space: 50G.
- EPS sync time is shorter.
  - Initial EPS Sync: 10-20 min.
  - Initial Electrumx Sync: 1 or more days.
- EPS has a safe install process.
  - EPS only compiles itself.
  - Electrumx pulls in dependencies without verification.

For more information see the
[mailing list email](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-February/015707.html)
and [bitcointalk thread](https://bitcointalk.org/index.php?topic=2664747.msg27179198).
## Why Do This?
This will protect you from having to trust nodes ran by volunteers to provide you with vital information and services regarding your Electrum wallet and your Bitcoin.

There have already been multiple waves of attacks on Electrum users perpetrated by malicious servers. These bad servers prevent users from sending transactions, instead sending back to them a bogus update requirement which actually leads to coin stealing malware.

This setup also preserves your privacy. When connecting to any server your wallet will leak information which can be used to tie your addresses together. There are definitely servers on the network which are using this information to build profiles on addresses and their interactions.
## Prerequisites
- To complete this guide you must have first completed:
  - [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md)

## I. Set Up Dom0
### A. In a `dom0` terminal, create a gateway.
**Notes:**
- This gateway should be independent of other Whonix gateways to isolate its onion service. See [here](https://www.whonix.org/wiki/Multiple_Whonix-Workstations#Multiple_Whonix-Gateways).
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' --prop provides_network='True' --prop vcpus='1' --template whonix-gw-15 sys-electrum-personal-server
```
### B. Create AppVM.
1. Create the AppVM for Electrum Personal Server with the newly created gateway, using the `whonix-ws-15-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label red --prop maxmem='400' --prop netvm='sys-electrum-personal-server' --prop vcpus='1' --template whonix-ws-15-bitcoin electrum-personal-server
```
2. Enable `electrum-personal-server` service.

```
[user@dom0 ~]$ qvm-service --enable electrum-personal-server electrum-personal-server
```
### C. Create rpc policy to allow comms from `electrum-personal-server` to `bitcoind`.
```
[user@dom0 ~]$ echo 'electrum-personal-server bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.bitcoind
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
### C. Use `systemd` to keep `electrum-personal-server` running.
1. Create `systemd` service file.

```
user@host:~$ lxsu mousepad /lib/systemd/system/electrum-personal-server.service
```

2. Paste the following.

```
[Unit]
Description=Electrum Personal Server
ConditionPathExists=/var/run/qubes-service/electrum-personal-server
After=qubes-sysinit.service

[Service]
ExecStart=/home/electrum-personal-server/epsvenv/bin/electrum-personal-server /home/electrum-personal-server/.eps/config.cfg

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

3. Save the file and switch back to the terminal.
4. Enable the service.

```
user@host:~$ sudo systemctl enable electrum-personal-server.service
Created symlink /etc/systemd/system/multi-user.target.wants/electrum-personal-server.service → /lib/systemd/system/electrum-personal-server.service.
```
### D. Shutdown TemplateVM.
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

**Notes:**
- Save the hased password (`9ffa7d78e1ddcb25ace4597bc31a1c8d$541c44f5d34044d532db47b74e9755ca4f0d87f805dd5895f0b36ea3a8d8c84c` in this example) to replace `<hashed-pass>` in later examples.
- Save your password (`GKkkKy-GAEDUw_6dp32O7Rh3DhHAnYhBUwNwNWUZPrI=` in this example) to replace `<rpc-pass>` in later examples.
- Replace `<rpc-user>` with the information noted earlier.

```
user@host:~$ ~/bitcoin/share/rpcauth/rpcauth.py <rpc-user>
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
3. Save the file and switch back to the terminal.
4. Restart the `bitcoind` service.

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

**Note:**
- At the time of writing the most recent version of Electrum Personal Server is `v0.1.6`, modify the following steps accordingly if the version has changed.

```
electrum-personal-server@host:~$ scurl-download https://github.com/chris-belcher/electrum-personal-server/archive/electrum-personal-server-v0.1.7.tar.gz https://github.com/chris-belcher/electrum-personal-server/releases/download/electrum-personal-server-v0.1.7/electrum-personal-server-v0.1.7.tar.gz.asc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   171    0   171    0     0     25      0 --:--:--  0:00:06 --:--:--    35
100 68723    0 68723    0     0   6561      0 --:--:--  0:00:10 --:--:-- 18776
curl: Saved to filename 'electrum-personal-server-electrum-personal-server-v0.1.7.tar.gz'
100   633    0   633    0     0    756      0 --:--:-- --:--:-- --:--:--  618k
100   819  100   819    0     0    141      0  0:00:05  0:00:05 --:--:--   190
curl: Saved to filename 'electrum-personal-server-v0.1.7.tar.gz.asc'
```
2. Receive signing key.

**Note:**
- You can verify the key fingerprint in the [release notes](https://github.com/chris-belcher/electrum-personal-server/releases).

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

```
electrum-personal-server@host:~$ gpg --verify electrum-personal-server-v0.1.7.tar.gz.asc electrum-personal-server-electrum-personal-server-v0.1.7.tar.gz
gpg: Signature made Fri 26 Apr 2019 04:08:13 PM UTC
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
electrum-personal-server@host:~$ tar -C ~/eps/ -xf electrum-personal-server-electrum-personal-server-v0.1.7.tar.gz --strip-components=1
```
1. Create virtual environment.

```
electrum-personal-server@host:~$ virtualenv -p python3 ~/epsvenv
Already using interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/electrum-personal-server/epsvenv/bin/python3
Also creating executable in /home/electrum-personal-server/epsvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
2. Source the virtual environment.

```
electrum-personal-server@host:~$ source ~/epsvenv/bin/activate
```
3. Enter the EPS directory, install EPS, and deactivate virtual environment.

```
(epsvenv) electrum-personal-server@host:~$ cd ~/eps/
(epsvenv) electrum-personal-server@host:~/eps$ python setup.py install
(epsvenv) electrum-personal-server@host:~/eps$ deactivate
```
4. Return to home directory.

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
- For a verbose desciption of these settings, look to the file: [`~/electrum-personal-server-eps-v0.1.6/config.cfg_sample`](https://github.com/chris-belcher/electrum-personal-server/blob/master/config.cfg_sample).
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
4. Save the file and switch back to the terminal.
5. Fix permissions.

```
electrum-personal-server@host:~$ chmod 0600 /home/electrum-personal-server/.eps/config.cfg
```
### B. Create certificate.
```
electrum-personal-server@host:~$ openssl req -x509 -sha256 -newkey rsa:4096 -keyout ~/.eps/certs/server.key -out ~/.eps/certs/server.crt -days 1825 -nodes -subj '/CN=localhost'
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
## VI. Set Up Communication Channels
### A. Remain in an `electrum-personal-server` terminal, open communication with `bitcoind` on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:8332,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm bitcoind qubes.bitcoind\" &" >> /rw/config/rc.local'
```
2. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### B. Set up `qubes-rpc` for `electrum-personal-server`.
**Note:**
- This only creates the possibility for other VMs to communicate with `electrum-personal-server`, it does not yet give them permission.

1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```
2. Create `qubes.electrum-personal-server` action file.

```
user@host:~$ sudo sh -c 'echo "socat STDIO TCP:127.0.0.1:50002" > /rw/usrlocal/etc/qubes-rpc/qubes.electrum-personal-server'
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
**Note:**
- Save the `electrum-personal-server` IP (`10.137.0.51` in this example) to replace `<eps-ip>` in later examples.

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

**Note:**
- Be sure to replace `<eps-ip>` with the information noted earlier.

```
HiddenServiceDir /var/lib/tor/electrum-personal-server/
HiddenServicePort 50002 <eps-ip>:50002
```
3. Save the file and switch back to the terminal.
4. Reload `tor`.

```
user@host:~$ sudo systemctl reload tor.service
```
5. Find out your onion hostname.

**Note:**
- Make a note of your server hostname for use with your remote Electrum wallet.

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
