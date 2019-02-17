# Qubes 4 & Whonix 14: Electrumx
Create a VM for running an [Electrumx](https://github.com/kyuupichan/electrumx) server which will connect to your `bitcoind` VM. The `electrumx` VM will be accessible from an Electrum Bitcoin wallet in an offline VM on the same host or remotely via a Tor onion service.
## What is Electrumx?
Electrumx is one of the possible server backends for the Electrum Bitcoin wallet. The other implementation covered in these guides is [Electrum Personal Server](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md) (EPS).

Here are some of the differences between Electrumx and EPS:
- Electrumx is more versatile.
  - Electrumx can serve any wallet once fully sync'd.
  - EPS requires that each wallet's [MPK](https://bitcoin.stackexchange.com/a/50031) is in its config file.
- An Electrumx VM requires more disk space.
  - Electrumx VM disk space: 40G.
  - EPS VM disk space: 1G
- Electrumxc sync time is longer.
  - Initial Electrumx Sync: 1 or more days.
  - Initial EPS Sync: 10-20 min.
- Electrumx has an unsafe install process.
  - Electrumx pulls in dependencies without verification.
  - EPS only compiles itself.

This guide will set up a private server which will not broadcast it's onion address or connect to any peers. If a user wishes to serve other peers on the network, then they will be responsible for making the needed changes to the Electrumx configuration.
## Why Do This?
This will protect you from having to trust nodes ran by volunteers to provide you with vital information and services regarding your Electrum wallet and the Bitcoin stored therein.

There have already been multiple waves of attacks on Electrum users perpetrated by malicious Electrumx servers. These bad servers prevent users from sending transactions, instead sending back to them a bogus update requirement which actually leads to coin stealing malware.

In addition to preventing certain types of attacks, this setup also increases your privacy by not leaking information about your wallet to server operators. There are servers on the network which are using this information to build profiles on addresses and their interactions (eg. [blockchain analytic companies](https://duckduckgo.com/html?q=blockchain%20analytics)).
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
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' --prop provides_network='True' --prop vcpus='1' --template whonix-gw-14 sys-electrumx
```
### B. Create AppVM.
1. Create the AppVM for Electrumx with the newly created gateway, using the `whonix-ws-14-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label red --prop maxmem='800' --prop netvm='sys-electrumx' --prop vcpus='1' --template whonix-ws-14-bitcoin electrumx
```
2. Increase private volume size and enable `electrumx` service.

```
[user@dom0 ~]$ qvm-volume resize electrumx:private 50G
[user@dom0 ~]$ qvm-service --enable electrumx electrumx
```

### C. Create rpc policy to allow comms from `electrumx` to `bitcoind`.
```
[user@dom0 ~]$ echo 'electrumx bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.bitcoind
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-14-bitcoin` terminal, update and install dependency.
```
user@host:~$ sudo apt update && sudo apt install -y python-virtualenv
```
### B. Create a system user.
```
user@host:~$ sudo adduser --system electrumx
Adding system user `electrumx' (UID 117) ...
Adding new user `electrumx' (UID 117) with group `nogroup' ...
Creating home directory `/home/electrumx' ...
```
### C. Use `systemd` to keep `electrumx` running.
1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /lib/systemd/system/electrumx.service
```
2. Paste the following.

```
[Unit]
Description=Electrumx
ConditionPathExists=/var/run/qubes-service/electrumx
After=qubes-sysinit.service

[Service]
EnvironmentFile=/home/electrumx/.electrumx/electrumx.conf
ExecStart=/home/electrumx/exvenv/bin/python3.6 /home/electrumx/exvenv/bin/electrumx_server

User=electrumx
Restart=on-failure
LimitNOFILE=8192
TimeoutStopSec=30min

PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```
3. Save the file and switch back to the terminal.
4. Fix permissions.

```
user@host:~$ sudo chmod 0644 /lib/systemd/system/electrumx.service
```
5. Enable the service.

```
user@host:~$ sudo systemctl enable electrumx.service
Created symlink /etc/systemd/system/multi-user.target.wants/electrumx.service → /lib/systemd/system/electrumx.service.
```
### D. Shutdown TemplateVM.
```
user@host:~$ sudo shutdown now
```
## III. Set up Bitcoin.
### A. In a `bitcoind` terminal, create RPC credentials for `electrumx` to communicate with `bitcoind`.
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
user@host:~$ sudo -u bitcoin kwrite /home/bitcoin/.bitcoin/bitcoin.conf
```
2. Paste the following at the bottom of the file.

**Notes:**
- Be sure not to alter any of the existing information.
- Replace `<rpc-user>` and `<hashed-pass>` with the information noted earlier.

```
# Electrumx Auth
rpcauth=<rpc-user>:<hashed-pass>
```
3. Save the file and switch back to the terminal.
4. Restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```
## III. Upgrade Python
### A. In an `electrumx` terminal, change to `electrumx` user.
1. Switch to user `electrumx` and change to home directory.

```
user@host:~$ sudo -H -u electrumx bash
electrumx@host:/home/user$ cd
```
### B. Download and verify Python3.6.
1. Download Python3.6.

**Note:**
- At the time of writing the most recent stable version of Python3.6 is `3.6.8`, modify the following steps accordingly if the version has changed.

```
electrumx@host:~$ curl -O "https://www.python.org/ftp/python/3.6.8/Python-3.6.8.tar.xz" -O "https://www.python.org/ftp/python/3.6.8/Python-3.6.8.tar.xz.asc"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0   325k      0  0:00:51  0:00:51 --:--:--  345k
100   833  100   833    0     0    958      0 --:--:-- --:--:-- --:--:--     0
```
2. Receive signing key.

**Note:**
- You can verify the key ID on the [downloads page](https://www.python.org/downloads/#pubkeys).

```
electrumx@host:~$ gpg --recv-keys 0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
gpg: keybox '/home/electrumx/.gnupg/pubring.kbx' created
gpg: key 0x2D347EA6AA65421D: 18 signatures not checked due to missing keys
gpg: /home/electrumx/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x2D347EA6AA65421D: public key "Ned Deily (Python release signing key) <nad@python.org>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify.

```
electrumx@host:~$ gpg --verify Python-3.6.8.tar.xz.asc Python-3.6.8.tar.xz
gpg: Signature made Mon 24 Dec 2018 03:07:36 AM UTC
gpg:                using RSA key 0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
gpg: Good signature from "Ned Deily (Python release signing key) <nad@python.org>" [unknown]
gpg:                 aka "Ned Deily <nad@baybryj.net>" [unknown]
gpg:                 aka "keybase.io/nad <nad@keybase.io>" [unknown]
gpg:                 aka "Ned Deily (Python release signing key) <nad@acm.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 0D96 DF4D 4110 E5C4 3FBF  B17F 2D34 7EA6 AA65 421D
```
### C. Build and install Python3.6.
1. Extract and enter directory.

```
electrumx@host:~$ tar -C ~ -xf Python-3.6.8.tar.xz
electrumx@host:~$ cd ~/Python-3.6.8/
```
2. Configure.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
electrumx@host:~/Python-3.6.8$ ./configure --enable-optimizations --prefix=/home/electrumx --with-ensurepip=install
```
3. Make and install.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
electrumx@host:~/Python-3.6.8$ make && make install
```
4. Return to home directory.

```
electrumx@host:~/Python-3.6.8$ cd
```
## IV. Install Electrumx
### A. Download and verify the Electrumx source code.
1. Clone the repository.

**Note:**
- The current version of Electrumx is `1.9.5`, modify the following steps accordingly if the version has changed.

```
electrumx@host:~$ curl -LO "https://github.com/kyuupichan/electrumx/archive/1.9.5.tar.gz"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   127    0   127    0     0     47      0 --:--:--  0:00:02 --:--:--    47
100  292k    0  292k    0     0  57103      0 --:--:--  0:00:05 --:--:--  172k
```
2. Verify download.

**Note:**
- The developer of Electrumx doesn't understand the importance of software verification and therefore does not sign or provide hash sums for his releases.
- While it doesn't offer the same security, I have included the SHA256 sum of my `1.9.5` download for your verification.

```
electrumx@host:~$ echo 'cb327e99bf20db364299012b0c85ff0a697f6fdae60f8f198743640333989e31  1.9.5.tar.gz' | LC_ALL=C shasum -c
1.9.5.tar.gz: OK
```
3. Extract.

```
electrumx@host:~$ tar -C ~ -xf 1.9.5.tar.gz
```
### B. Create virtual environment.
1. Fix `$PATH`.

```
electrumx@host:~$ source ~/.profile
```
2. Change directory.

```
electrumx@host:~$ cd ~/electrumx-1.9.5/
```
3. Create virtual environment.

```
electrumx@host:~/electrumx-1.9.5$ virtualenv -p python3.6 ~/exvenv
Running virtualenv with interpreter /home/electrumx/bin/python3.6
Using base prefix '/home/electrumx'
New python executable in /home/electrumx/exvenv/bin/python3.6
Also creating executable in /home/electrumx/exvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
### C. Install inside virtual environment.
1. Source virtual environment.

```
electrumx@host:~/electrumx-1.9.5$ source ~/exvenv/bin/activate
```
2. Install Electrumx.

```
(exvenv) electrumx@host:~/electrumx-1.9.5$ python setup.py install
```
3. Deactivate virtual environment and return to home dir.

```
(exvenv) electrumx@host:~/electrumx-1.9.5$ deactivate; cd
```
## IV. Set Up Electrumx
### A. Remain in an `electrumx` terminal, configure Electrumx data directory.
1. Create Electrumx's data directory and subdirectories.

```
electrumx@host:~$ mkdir -m 0700 ~/.electrumx
electrumx@host:~$ mkdir -m 0700 ~/.electrumx/{certs,electrumx-db}
```
2. Create configuration file.

```
electrumx@host:~$ kwrite ~/.electrumx/electrumx.conf
```
3. Paste the following.

**Notes:**
- Be sure to replace `<rpc-user>` and `<rpc-pass>` with the information noted earlier.
- For a verbose desciption of these settings, look to the file: [`~/electrumx/docs/environment.rst`](https://electrumx.readthedocs.io/en/latest/environment.html).

```
## Required
COIN = BitcoinSegwit
DB_DIRECTORY = /home/electrumx/.electrumx/electrumx-db
DAEMON_URL = http://<rpc-user>:<rpc-pass>@127.0.0.1:8332/
USERNAME = electrumx
## Miscellaneous
NET = mainnet
DB_ENGINE = leveldb
HOST = ''
SSL_PORT = 50002
SSL_CERTFILE = /home/electrumx/.electrumx/certs/server.crt
SSL_KEYFILE = /home/electrumx/.electrumx/certs/server.key
## Peer Discovery
PEER_DISCOVERY = self
PEER_ANNOUNCE = ''
## Server Advertising
REPORT_TCP_PORT = 0
REPORT_SSL_PORT = 0
## Cache
CACHE_MB = 400
## Python3.6
PYTHONHOME = /home/electrumx/exvenv
```
4. Save the file and switch back to the terminal.

### B. Create certificate.
```
electrumx@host:~$ openssl req -x509 -sha256 -newkey rsa:4096 -keyout ~/.electrumx/certs/server.key -out ~/.electrumx/certs/server.crt -days 1825 -nodes -subj '/CN=localhost'
Generating a RSA private key
.........................................................................................++++
....++++
writing new private key to '/home/electrumx/.electrumx/certs/server.key'
-----
```
### C. Change back to original user.
```
electrumx@host:~$ exit
```
## V. Set Up Communication Channels
### A. Remain in an `electrumx` terminal, open communication with `bitcoind` on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:8332,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm bitcoind qubes.bitcoind\" &" >> /rw/config/rc.local'
```
2. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### B. Set up `qubes-rpc` for `electrumx`.
**Note:**
- This only creates the possibility for other VMs to communicate with `electrumx`, it does not yet give them permission.

1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```
2. Create `qubes.electrumx` action file.

```
user@host:~$ sudo sh -c 'echo "socat STDIO TCP:127.0.0.1:50002" > /rw/usrlocal/etc/qubes-rpc/qubes.electrumx'
```
3. Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/usrlocal/etc/qubes-rpc/qubes.electrumx
```
### C. Open firewall for Tor onion service.
1. Make persistent directory for new firewall rules.

```
user@host:~$ sudo mkdir -m 0755 /rw/config/whonix_firewall.d
```
2. Configure firewall.

```
user@host:~$ sudo sh -c 'echo "EXTERNAL_OPEN_PORTS+=\" 50002 \"" >> /rw/config/whonix_firewall.d/50_user.conf'
```
3. Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/config/whonix_firewall.d/50_user.conf
```
4. Restart firewall service.

```
user@host:~$ sudo systemctl restart whonix-firewall.service
```
### D. Find out the IP address of the `electrumx` VM.
**Note:**
- Save the `electrumx` IP (`10.137.0.50` in this example) to replace `<electrumx-ip>` in later examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
## VI. Initial Electrumx Synchronization
### A. In an `electrumx` terminal, start the `electrumx` service.
```
user@host:~$ sudo systemctl start electrumx.service
```
## VII. Set Up Gateway.
### A. In a `sys-electrumx` terminal, set up Tor.
1. Edit the Tor configuration file.

```
user@host:~$ sudo kwrite /rw/usrlocal/etc/torrc.d/50_user.conf
```
2. Paste the following.

**Note:**
- Be sure to replace `<electrumx-ip>` with the information noted earlier.

```
HiddenServiceDir /var/lib/tor/electrumx/
HiddenServicePort 50002 <electrumx-ip>:50002
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
user@host:~$ sudo cat /var/lib/tor/electrumx/hostname
electrumxtoronionserviceaddressxxxxxxxxxxxxxxxxxxxxxxxxx.onion
```
## VIII. Final Notes
- The intial sync can take anywhere from a day to multiple days depending on a number of factors including your hardware and resources dedicated to the `electrumx` VM.
- Once the sync is finished you may connect your Electrum wallet via the Tor onion address.
- To check the status of the server: `sudo journalctl -fu electrumx`
- To connect an offline Electrum wallet from a separate VM (split-electrum) use the guide: [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md).
