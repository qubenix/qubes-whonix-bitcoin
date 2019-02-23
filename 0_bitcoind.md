# Qubes 4 & Whonix 14: Building a Bitcoin Core Full Node
Build a [Bitcoin Core](https://github.com/bitcoin/bitcoin) full node from source code and configure it to:
- Allow other VMs to connect when given explicit permission from `dom0`.
- Communicate only over Tor.
- Easily plugin other applications which require a `bitcoind` backend.
- Index all transactions.
- Prefer hidden services, use them exclusively if possible.
- Use ephemeral hidden services when serving peers.
- Utilize Tor stream isolation.

## What is Bitcoin Core?
The server daemon for the Bitcoin distributed cryptocurrency (`bitcoind`), command line tools (`bitcoin-cli`), and a gui wallet (`bitcoin-qt`). These tools can be used to observe and interact with Bitcoin's blockchain.
## Why Do This?
Bitcoin Core is a "full node" implementation, meaning it will verify that all incoming transactions and blocks are following Bitcoin's rules. This allows you to validate transactions without trusting third parties.

An indexed node can be used as a backend for other software which needs access to the blockchain, such as:
- [BTCPay Server](https://github.com/btcpayserver/btcpayserver)
- [c-Lightning](https://github.com/ElementsProject/lightning)
- [Electrum Personal Server](https://github.com/chris-belcher/electrum-personal-server)
  - Guide: [`1_electrum-personal-server.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md)
- [Electrumx](https://github.com/kyuupichan/electrumx)
  - Guide: [`1_electrumx.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md)
- [JoinMarket](https://github.com/JoinMarket-Org/joinmarket-clientserver)
  - Guide: [`1_joinmarket.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_joinmarket.md)
- [LND](https://github.com/LightningNetwork/lnd)

Using `qrexec` we can connect any of these tools to `bitcoind` from their own VM, making use of the Qubes security by isolation model.
## I. Set Up Dom0
### A. In a `dom0` terminal, clone a Whonix workstation TemplateVM.
```
[user@dom0 ~]$ qvm-clone whonix-ws-14 whonix-ws-14-bitcoin
```
### B. Create a gateway.
**Notes:**
- This gateway should be independent of other Whonix gateways to isolate its onion service. See [here](https://www.whonix.org/wiki/Multiple_Whonix-Workstations#Multiple_Whonix-Gateways).
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' --prop provides_network='True' --prop vcpus='1' --template whonix-gw-14 sys-bitcoin
```
### C. Create an AppVM, use newly created gateway and template.
**Note:**
- You must choose a label color, but it does not have to match this example.

```
[user@dom0 ~]$ qvm-create --label red --prop netvm='sys-bitcoin' --template whonix-ws-14-bitcoin bitcoind
```
2. Increase private volume size and enable `bitcoind` service.

```
[user@dom0 ~]$ qvm-volume resize bitcoind:private 250G
[user@dom0 ~]$ qvm-service --enable bitcoind bitcoind
```
## II. Set Up TemplateVM
### A. In the `whonix-ws-14-bitcoin` terminal, update and install dependencies.
```
user@host:~$ sudo apt update && sudo apt install -y automake autotools-dev build-essential git \
libboost-chrono-dev libboost-filesystem-dev libboost-system-dev libboost-test-dev libboost-thread-dev \
libevent-dev libprotobuf-dev libqrencode-dev libqt5core5a libqt5dbus5 libqt5gui5 libssl-dev libtool \
libzmq3-dev pkg-config protobuf-compiler qttools5-dev qttools5-dev-tools
```
### B. Create system user.
```
user@host:~$ sudo adduser --system bitcoin
Adding system user `bitcoin' (UID 116) ...
Adding new user `bitcoin' (UID 116) with group `nogroup' ...
Creating home directory `/home/bitcoin' ...
```
### C. Use `systemd` to keep `bitcoind` running.
1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /lib/systemd/system/bitcoind.service
```
2. Paste the following.

```
[Unit]
Description=Bitcoin daemon
ConditionPathExists=/var/run/qubes-service/bitcoind
After=qubes-sysinit.service

[Service]
ExecStart=/home/bitcoin/bin/bitcoind -daemon -pid=/run/bitcoind/bitcoind.pid
ExecStop=/home/bitcoin/bin/bitcoin-cli stop

RuntimeDirectory=bitcoind
User=bitcoin
Type=forking
PIDFile=/run/bitcoind/bitcoind.pid
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
4. Fix permissions.

```
user@host:~$ sudo chmod 0644 /lib/systemd/system/bitcoind.service
```
5. Enable the service.

```
user@host:~$ sudo systemctl enable bitcoind.service
Created symlink /etc/systemd/system/multi-user.target.wants/bitcoind.service → /lib/systemd/system/bitcoind.service.
```
### D. Shutdown TemplateVM.
```
user@host:~$ sudo poweroff
```
## III. Set Up Gateway.
### A. In a `sys-bitcoin` terminal, find out the gateway IP.
**Note:**
- Save your gateway IP (`10.137.0.50` in this example) to replace `<gateway-ip>` in later examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
### B. Configure `onion-grater`.
1. Install provided profile for `bitcoind` to persistent directory.

```
user@host:~$ sudo install -D -t /usr/local/etc/onion-grater-merger.d/ /usr/share/onion-grater-merger/examples/40_bitcoind.yml
```
2. Restart `onion-grater` service.

```
user@host:~$ sudo systemctl restart onion-grater.service
```
## IV. Install Bitcoin
### A. In a `bitcoind` terminal, change to `bitcoin` user.
1. Switch to user `bitcoin` and change to home directory.

```
user@host:~$ sudo -H -u bitcoin bash
bitcoin@host:/home/user$ cd
```
### B. Download and verify the Bitcoin source code.
1. Clone the repository.

**Note:**
- At the time of writing the current release branch is `0.17`, modify the following steps accordingly if the version has changed.

```
bitcoin@host:~$ git clone --branch 0.17 https://github.com/bitcoin/bitcoin ~/bitcoin
Cloning into '/home/bitcoin/bitcoin'...
remote: Enumerating objects: 15, done.
remote: Counting objects: 100% (15/15), done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 132465 (delta 7), reused 5 (delta 4), pack-reused 132450
Receiving objects: 100% (132465/132465), 117.69 MiB | 441.00 KiB/s, done.
Resolving deltas: 100% (92773/92773), done.
```
2. Enter `bitcoin/` directory and receive signing keys.

```
bitcoin@host:~/bitcoin$ cd ~/bitcoin/
bitcoin@host:~/bitcoin$ gpg --recv-keys $(<contrib/verify-commits/trusted-keys)
gpg: keybox '/home/bitcoin/.gnupg/pubring.kbx' created
gpg: key 0x3648A882F4316B9B: 43 signatures not checked due to missing keys
gpg: /home/bitcoin/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x3648A882F4316B9B: public key "Marco Falke <marco.falke@tum.de>" imported
gpg: key 0x29D4BCB6416F53EC: 4 duplicate signatures removed
gpg: key 0x29D4BCB6416F53EC: 11 signatures not checked due to missing keys
gpg: key 0x29D4BCB6416F53EC: 1 signature reordered
gpg: key 0x29D4BCB6416F53EC: public key "Jonas Schnelli <dev@jonasschnelli.ch>" imported
gpg: key 0x860FEB804E669320: 61 signatures not checked due to missing keys
gpg: key 0x860FEB804E669320: public key "Pieter Wuille <pieter.wuille@gmail.com>" imported
gpg: key 0x74810B012346C9A6: 12 duplicate signatures removed
gpg: key 0x74810B012346C9A6: 73 signatures not checked due to missing keys
gpg: key 0x74810B012346C9A6: public key "Wladimir J. van der Laan <laanwj@protonmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 4
gpg:               imported: 4
```
3. Verify source code.

**Note:**
- Your output may not match the example. Just check that it says `Good signature`.

```
bitcoin@host:~/bitcoin$ git verify-commit HEAD
gpg: Signature made Sat Feb  2 11:30:09 2019 UTC
gpg:                using RSA key 9DEAE0DC7063249FB05474681E4AED62986CD25D
gpg: Good signature from "Wladimir J. van der Laan <laanwj@visucore.com>" [unknown]
gpg:                 aka "Wladimir J. van der Laan <laanwj@gmail.com>" [unknown]
gpg:                 aka "Wladimir J. van der Laan <laanwj@protonmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 71A3 B167 3540 5025 D447  E8F2 7481 0B01 2346 C9A6
     Subkey fingerprint: 9DEA E0DC 7063 249F B054  7468 1E4A ED62 986C D25D
```
### C. Build Berkeley DB and Bitcoin.
**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

1. Build Berkeley DB using the provided script.

```
bitcoin@host:~/bitcoin$ ./contrib/install_db4.sh `pwd`
```
2. Generate Bitcoin Core configuration script.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
bitcoin@host:~/bitcoin$ export BDB_PREFIX='/home/bitcoin/bitcoin/db4'; ./autogen.sh
```
3. Configure.

```
bitcoin@host:~/bitcoin$ ./configure BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" BDB_CFLAGS="-I${BDB_PREFIX}/include" --prefix=/home/bitcoin
```
4. Make and install.

```
bitcoin@host:~/bitcoin$ make check && make install
```
5. Return to home directory.

```
bitcoin@host:~/bitcoin$ cd
```
## V. Set Up Bitcoin.
### A. Remain in a `bitcoind` terminal, configure Bitcoin.
1. Create Bitcoin's data directory and configuration file.

```
bitcoin@host:~$ mkdir -m 0700 ~/.bitcoin
bitcoin@host:~$ kwrite ~/.bitcoin/bitcoin.conf
```
2. Paste the following.

**Note:**
- Be sure to replace `<gateway-ip>` with the information noted earlier.

```
listen=1
onion=<gateway-ip>:9111
onlynet=onion
proxy=<gateway-ip>:9111
txindex=1
```
3. Save the file and switch back to the terminal.
4. Fix permissions.

```
bitcoin@host:~$ chmod 0600 ~/.bitcoin/bitcoin.conf
```
### B. Change back to original user.
```
bitcoin@host:~$ exit
```
### C. Open firewall for Tor onion service.
1. Make persistent directory for new firewall rules.

```
user@host:~$ sudo mkdir -m 0755 /rw/config/whonix_firewall.d
```
2. Configure firewall.

```
user@host:~$ sudo sh -c 'echo "EXTERNAL_OPEN_PORTS+=\" 8333 \"" >> /rw/config/whonix_firewall.d/50_user.conf'
```
3. Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/config/whonix_firewall.d/50_user.conf
```
4. Restart firewall service.

```
user@host:~$ sudo systemctl restart whonix-firewall.service
```
## VI. Create Communication Channel
**Note:**
- This only creates the possibility for other VMs to communicate with `bitcoind`, it does not yet give them permission.

### A. In a `bitcoind` terminal, set up `qubes-rpc` for `bitcoind`.
1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```
2. Create `qubes.bitcoind` action file.

```
user@host:~$ sudo sh -c 'echo "socat STDIO TCP:127.0.0.1:8332" > /rw/usrlocal/etc/qubes-rpc/qubes.bitcoind'
```
3. Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/usrlocal/etc/qubes-rpc/qubes.bitcoind
```
## VII. Create Alias
1. Make an alias in order to control `bitcoind` easier.

```
user@host:~$ echo 'alias bitcoin-cli="sudo -u bitcoin /home/bitcoin/bin/bitcoin-cli"' >> ~/.bashrc
```
2. Source the file.

```
user@host:~$ source ~/.bashrc
```
## VIII. Initial Blockchain Download
**Note:**
- Initial block download can take anywhere from a day to a week (or even more) depending on a number of factors including your hardware and internet connection.

### A. In a `bitcoind` terminal, start the `bitcoind` service.
```
user@host:~$ sudo systemctl start bitcoind
```
## VIII. Final Notes
- Once the initial synchronization has finished you may begin using your Bitcoin node as a backend for other services.
- To check the status of the server: `sudo tail -f /home/bitcoin/.bitcoin/debug.log`
- To interact with the server: `bitcoin-cli --help`
