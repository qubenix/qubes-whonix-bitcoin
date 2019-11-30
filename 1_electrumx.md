# Qubes 4 & Whonix 15: Electrumx
Create a VM for running an [Electrumx](https://github.com/kyuupichan/electrumx) server which will connect to your `bitcoind` VM. The `electrumx` VM will be accessible from an Electrum Bitcoin wallet in an offline VM on the same host or remotely via a Tor onion service.
## What is Electrumx?
Electrumx is one of the possible server backends for the Electrum Bitcoin wallet. The other implementations covered in these guides are [Electrs](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrs.md), and [Electrum Personal Server](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md) (EPS).

Here are some of the differences between the different implementations:
- Electrs and Electrumx are more versatile.
  - Electrs and Electrumx can serve any wallet once fully synchronized.
  - EPS requires that each wallet's [MPK](https://bitcoin.stackexchange.com/a/50031) is in its config file.
- Different disk space requirements.
  - Electrs and Electrumx VM disk space: under 60G.
  - EPS VM disk space: 1G.
- Different initial sync times.
  - Initial Electrs Sync: 1-12 hours.
  - Initial Electrumx Sync: 12-24 hours.
  - Initial EPS Sync: 10-20 min.
- Electrumx can be configured to be part of a p2p network of servers that support random Electrum wallet users. That setup is out of scope for these guides.

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
- This gateway should be independent of other Whonix gateways to isolate its onion service. See [here](https://www.whonix.org/wiki/Multiple_Whonix-Gateway).
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' \
--prop provides_network='True' --prop vcpus='1' --template whonix-gw-15 sys-electrumx
```
### B. Create AppVM.
1. Create the AppVM for Electrumx with the newly created gateway, using the `whonix-ws-15-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label red --prop maxmem='800' --prop netvm='sys-electrumx' \
--prop vcpus='1' --template whonix-ws-15-bitcoin electrumx
```
2. Increase private volume size.

```
[user@dom0 ~]$ qvm-volume resize electrumx:private 60G
```

### C. Allow comms from `electrumx` to `bitcoind`.
```
[user@dom0 ~]$ echo 'electrumx bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.ConnectTCP
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-15-bitcoin` terminal, update and install dependency.
```
user@host:~$ sudo apt update && sudo apt install -y python-virtualenv python3-aiohttp
```
### B. Create a system user.
```
user@host:~$ sudo adduser --system electrumx
Adding system user `electrumx' (UID 117) ...
Adding new user `electrumx' (UID 117) with group `nogroup' ...
Creating home directory `/home/electrumx' ...
```
### C. Shutdown TemplateVM.
```
user@host:~$ sudo poweroff
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
user@host:~$ sudo -u bitcoin mousepad /home/bitcoin/.bitcoin/bitcoin.conf
```
2. Paste the following at the bottom of the file.

**Notes:**
- Be sure not to alter any of the existing information.
- Replace `<rpc-user>` and `<hashed-pass>` with the information noted earlier.

```
# Electrumx Auth
rpcauth=<rpc-user>:<hashed-pass>
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```
## IV. Install Electrumx
### A. Download and verify Electrumx.
1. Switch to user `electrumx` and enter home directory.

```
user@host:~$ sudo -H -u electrumx bash
electrumx@host:/home/user$ cd
```
2. Download the latest Electrumx [release](https://github.com/kyuupichan/electrumx/releases).

**Note:**
- The current version of Electrumx is `1.13.0`, modify the following steps accordingly if the version has changed.

```
electrumx@host:~$ scurl-download https://github.com/kyuupichan/electrumx/archive/1.13.0.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   128    0   128    0     0     46      0 --:--:--  0:00:02 --:--:--    46
100  339k    0  339k    0     0  33692      0 --:--:--  0:00:10 --:--:-- 73462
curl: Saved to filename 'electrumx-1.13.0.tar.gz'
```
3. Verify download.

**Note:**
- The developer of Electrumx doesn't understand the importance of software verification and therefore does not sign or provide hash sums for his releases.
- While it doesn't offer the same security, I have included the SHA256 sum of my `electrumx-1.13.0.tar.gz` download for your verification.

```
electrumx@host:~$ echo '8dc7cf5b15fe48c5abbadc5c12b5bf25ca8bd5d10f4d781ca9613004fe8d4b50  electrumx-1.13.0.tar.gz' | shasum -c
electrumx-1.13.0.tar.gz: OK
```
4. Extract.

```
electrumx@host:~$ tar -C ~ -xf electrumx-1.13.0.tar.gz
```
### B. Create virtual environment, link installed packages.
1. Create virtual environment.

```
electrumx@host:~$ virtualenv -p python3 ~/exvenv
Already using interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/electrumx/exvenv/bin/python3
Also creating executable in /home/electrumx/exvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
2. Link installed packages to virtual environment.

```
electrumx@host:~$ ln -s /usr/lib/python3/dist-packages/* ~/exvenv/lib/python3.7/site-packages/
```
### C. Install inside virtual environment.
1. Source virtual environment.

```
electrumx@host:~$ source ~/exvenv/bin/activate
```
2. Change directory.

```
(exvenv) electrumx@host:~$ cd ~/electrumx-1.13.0/
```
3. Install Electrumx.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
(exvenv) electrumx@host:~/electrumx-1.13.0$ python setup.py install
```
4. Deactivate virtual environment and return to home dir.

```
(exvenv) electrumx@host:~/electrumx-1.13.0$ deactivate; cd
```
## V. Set Up Electrumx
### A. Remain in an `electrumx` terminal, configure Electrumx data directory.
1. Create Electrumx's data directory and subdirectories.

```
electrumx@host:~$ mkdir -m 0700 ~/.electrumx
electrumx@host:~$ mkdir -m 0700 ~/.electrumx/{certs,electrumx-db}
```
2. Create configuration file.

```
electrumx@host:~$ mousepad ~/.electrumx/electrumx.conf
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

## Services
SERVICES = ssl://:50002,rpc://

## Miscellaneous
NET = mainnet
DB_ENGINE = leveldb
SSL_CERTFILE = /home/electrumx/.electrumx/certs/server.crt
SSL_KEYFILE = /home/electrumx/.electrumx/certs/server.key

## Peer Discovery
PEER_DISCOVERY = self
PEER_ANNOUNCE = ''

## Cache
CACHE_MB = 400

## Python
PYTHONHOME = /home/electrumx/exvenv
```
4. Save the file: `Ctrl-S`.
5. Switch back to the terminal: `Ctrl-Q`.
### B. Create certificate.
```
electrumx@host:~$ openssl req -x509 -sha256 -newkey rsa:4096 \
-keyout ~/.electrumx/certs/server.key -out ~/.electrumx/certs/server.crt -days 1825 \
-nodes -subj '/CN=localhost'
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
### D. Use `systemd` to keep `electrumx` running.
1. Create a persistent directory.

```
user@host:~$ sudo mkdir -m 0700 /rw/config/systemd
```
2. Create the service file.

```
user@host:~$ lxsu mousepad /rw/config/systemd/electrumx.service
```
3. Paste the following.

```
[Unit]
Description=Electrumx daemon

[Service]
EnvironmentFile=/home/electrumx/.electrumx/electrumx.conf
ExecStart=/home/electrumx/exvenv/bin/python /home/electrumx/exvenv/bin/electrumx_server

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
4. Save the file: `Ctrl-S`.
5. Switch back to the terminal: `Ctrl-Q`.
6. Fix permissions.

```
user@host:~$ chmod 0600 /rw/config/systemd/electrumx.service
```
### E. Enable the service on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ lxsu mousepad /rw/config/rc.local
```
2. Paste the following at the bottom of the file.

```
cp /rw/config/systemd/electrumx.service /lib/systemd/system/
systemctl daemon-reload
systemctl start electrumx.service
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
## VI. Fix Networking
### A. Remain in an `electrumx` terminal, open communication with `bitcoind` on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo sh -c 'echo "qvm-connect-tcp 8332:bitcoind:8332" >> /rw/config/rc.local'
```
2. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### B. Open firewall for Tor onion service.
1. Make persistent directory for new firewall rules.

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
### D. Find out the IP address of the `electrumx` VM.
**Note:**
- Save the `electrumx` IP (`10.137.0.50` in this example) to replace `<electrumx-ip>` in later examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
## VII. Initial Electrumx Synchronization
### A. In an `electrumx` terminal, start the `electrumx` service.
```
user@host:~$ sudo systemctl start electrumx.service
```
## VIII. Set Up Gateway.
### A. In a `sys-electrumx` terminal, set up Tor.
1. Edit the Tor configuration file.

```
user@host:~$ lxsu mousepad /rw/usrlocal/etc/torrc.d/50_user.conf
```
2. Paste the following.

**Note:**
- Be sure to replace `<electrumx-ip>` with the information noted earlier.

```
HiddenServiceDir /var/lib/tor/electrumx/
HiddenServicePort 50002 <electrumx-ip>:50002
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Reload `tor`.

```
user@host:~$ sudo systemctl reload tor.service
```
6. Find out your onion hostname.

**Note:**
- Make a note of your server hostname for use with your remote Electrum wallet.

```
user@host:~$ sudo cat /var/lib/tor/electrumx/hostname
electrumxtoronionserviceaddressxxxxxxxxxxxxxxxxxxxxxxxxx.onion
```
## IX. Final Notes
- The intial sync can take anywhere from a day to multiple days depending on a number of factors including your hardware and resources dedicated to the `electrumx` VM.
- Once the sync is finished you may connect your Electrum wallet via the Tor onion address.
- To check the status of the server: `sudo journalctl -fu electrumx`
- To connect an offline Electrum wallet from a separate VM (split-electrum) use the guide: [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md).
