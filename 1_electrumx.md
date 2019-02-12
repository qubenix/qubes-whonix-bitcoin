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
Requires=qubes-mount-dirs.service

[Service]
EnvironmentFile=/home/electrumx/.electrumx/electrumx.conf
ExecStart=/home/electrumx/exvenv/bin/electrumx_server

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
- Save your username (`7PXaFZ5DLG2alSeiGxnM` in this example) for later to replace `<rpc-user>` in examples.

```
user@host:~$ head -c 15 /dev/urandom | base64
7PXaFZ5DLG2alSeiGxnM
```
2. Use Bitcoin's tool to create a random RPC password and config entry. Do not use the one shown.

**Notes:**
- Save the hased password (`9ffa7d78e1ddcb25ace4597bc31a1c8d$541c44f5d34044d532db47b74e9755ca4f0d87f805dd5895f0b36ea3a8d8c84c` in this example) for later to replace `<hashed-pass>` in examples.
- Save your password (`GKkkKy-GAEDUw_6dp32O7Rh3DhHAnYhBUwNwNWUZPrI=` in this example) for later to replace `<rpc-pass>` in examples.
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
## III. Install Electrumx
### A. In an `electrumx` terminal, upgrade Python.
1. Download Python3.6.

**Note:**
- At the time of writing the most recent stable version of Python3.6 is `3.6.8`, modify the following steps accordingly if the version has changed.

```
user@host:~$ curl -O "https://www.python.org/ftp/python/3.6.8/Python-3.6.8.tar.xz" -O "https://www.python.org/ftp/python/3.6.8/Python-3.6.8.tar.xz.asc"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0   325k      0  0:00:51  0:00:51 --:--:--  345k
100   833  100   833    0     0    958      0 --:--:-- --:--:-- --:--:--     0
```
2. Receive signing key.

**Note:**
- You can verify the key ID on the [downloads page](https://www.python.org/downloads/#pubkeys).

```
user@host:~$ gpg --recv-keys 0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
gpg: keybox '/home/user/.gnupg/pubring.kbx' created
key 0x2D347EA6AA65421D:
18 signatures not checked due to missing keys
gpg: /home/user/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x2D347EA6AA65421D: public key "Ned Deily (Python release signing key) <nad@python.org>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify.

```
user@host:~$ gpg --verify Python-3.6.8.tar.xz.asc Python-3.6.8.tar.xz
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
4. Extract and enter directory.

```
user@host:~$ tar -C ~ -xf Python-3.6.8.tar.xz
user@host:~$ cd ~/Python-3.6.8/
```
5. Build and install.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
user@host:~/Python-3.6.8$ ./configure --enable-optimizations --with-ensurepip=install && make && sudo make install
```
6. Return to home directory.

```
user@host:~/Python-3.6.8$ cd
```
### B. Download and verify the Electrumx source code.
1. Clone the repository.

**Notes:**
- This is a fork of the [original](https://github.com/kyuupichan/electrumx) Electrumx repository which still supports the version of Electrum in Debian repos.
- This is the only source I've found which is signed.

```
user@host:~$ git clone -b proto_compat_1_2 https://github.com/SomberNight/electrumx ~/electrumx
Cloning into '/home/user/electrumx'...
remote: Enumerating objects: 37, done.
remote: Counting objects: 100% (37/37), done.
remote: Compressing objects: 100% (29/29), done.
remote: Total 8522 (delta 11), reused 21 (delta 8), pack-reused 8485
Receiving objects: 100% (8522/8522), 2.88 MiB | 273.00 KiB/s, done.
Resolving deltas: 100% (5795/5795), done.
```
2. Receive signing key.

**Note:**
- You can verify the key fingerprint on the developers [keybase page](https://keybase.io/ghost43).

```
user@host:~$ gpg --recv-keys 4AD64339DFA05E20B3F6AD51E7B748CDAF5E5ED9
key 0xE7B748CDAF5E5ED9:
4 signatures not checked due to missing keys
gpg: key 0xE7B748CDAF5E5ED9: public key "SomberNight <somber.night@protonmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify.

**Note:**
- Your output may not match the example. Just check that it says `Good signature`.

```
user@host:~$ cd ~/electrumx/
user@host:~/electrumx$ git verify-commit HEAD
gpg: Signature made Sat 02 Feb 2019 05:22:57 PM UTC
gpg:                using RSA key 6D7A2116DA909E00AC21108BB33B5F232C6271E9
gpg: Good signature from "SomberNight <somber.night@protonmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 4AD6 4339 DFA0 5E20 B3F6  AD51 E7B7 48CD AF5E 5ED9
     Subkey fingerprint: 6D7A 2116 DA90 9E00 AC21  108B B33B 5F23 2C62 71E9
```
### C. Install to virtual environment.
1. Create virtual environment.

```
user@host:~/electrumx$ virtualenv -p python3.6 ~/exvenv
Running virtualenv with interpreter /usr/local/bin/python3.6
Using base prefix '/usr/local'
New python executable in /home/user/exvenv/bin/python3.6
Also creating executable in /home/user/exvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
2. Install Electrumx and dependencies inside virtual environment.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
user@host:~/electrumx$ source ~/exvenv/bin/activate
(exvenv) user@host:~/electrumx$ python setup.py install
```
3. Deactivate virtual environment and make relocatable.

```
(exvenv) user@host:~/electrumx$ deactivate
user@host:~/electrumx$ virtualenv -p python3.6 --relocatable ~/exvenv/
```
4. Return to home directory.

```
user@host:~/electrumx$ cd
```
### D. Relocate virtual environment directory.
```
user@host:~$ sudo cp -r ~/exvenv/ /home/electrumx/
```
## IV. Set Up Electrumx
### A. Remain in an `electrumx` terminal, configure Electrumx data directory.
1. Create Electrumx's data directory and subdirectories.

```
user@host:~$ sudo mkdir -m 0700 /home/electrumx/.electrumx
user@host:~$ sudo mkdir -m 0700 /home/electrumx/.electrumx/{certs,electrumx-db}
```
2. Create configuration file.

```
user@host:~$ sudo kwrite /home/electrumx/.electrumx/electrumx.conf
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
PEER_DISCOVERY = 'self'
PEER_ANNOUNCE = ''
## Server Advertising
REPORT_TCP_PORT = 0
## Cache
CACHE_MB = 400
```
4. Save the file and switch back to the terminal.
5. Fix permissions.

```
user@host:~$ sudo chmod 0600 /home/electrumx/.electrumx/electrumx.conf
```
### B. Make certs.
1. Create server key.

```
user@host:~$ openssl genrsa -out server.key 2048
Generating RSA private key, 2048 bit long modulus
.........+++++
..............................................+++++
e is 65537 (0x010001)
```
2. Create certificate.

**Note:**
- Answer the questions or leave them blank.

```
user@host:~$ openssl req -new -key server.key -out server.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
3. Sign the certificate.

```
user@host:~$ openssl x509 -req -days 1825 -in server.csr -signkey server.key -out server.crt
Signature ok
subject=C = AU, ST = Some-State, O = Internet Widgits Pty Ltd
Getting Private key
```
4. Copy all cert files to the Electrumx data directory.

```
user@host:~$ sudo install -m 0600 -t /home/electrumx/.electrumx/certs/ server.{crt,csr,key}
```
### C. Change owner.
```
user@host:~$ sudo chown -R electrumx:nogroup /home/electrumx/
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
1. Make persistent directory.

```
user@host:~$ sudo mkdir -m 0755 /rw/config/whonix_firewall.d
```
2. Configure firewall, and fix permissions.

```
user@host:~$ sudo sh -c 'echo "EXTERNAL_OPEN_PORTS+=\" 50002 \"" >> /rw/config/whonix_firewall.d/50_user.conf'
user@host:~$ sudo chmod 0644 /rw/config/whonix_firewall.d/50_user.conf
```
3. Restart firewall service.

```
user@host:~$ sudo systemctl restart whonix-firewall.service
```
### D. Find out the IP address of the `electrumx` VM.
**Note:**
- Save the `electrumx` IP (`10.137.0.50` in this example) for later to replace `<electrumx-ip>` in examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
## VI. Set Up Gateway.
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
## VII. Initial Electrumx Synchronization
### A. In an `electrumx` terminal, start the `electrumx` service.
```
user@host:~$ sudo systemctl start electrumx.service
```
### B. Check the status of the server.
```
user@host:~$ sudo journalctl -fu electrumx
```
## VIII. Final Notes
- The intial sync can take anywhere from a day to multiple days depending on a number of factors including your hardware and resources dedicated to the `electrumx` VM.
- Once the sync is finished you may connect your Electrum wallet via the Tor onion address.
- To connect an offline Electrum wallet from a separate VM (split-electrum) use the guide: [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md).
