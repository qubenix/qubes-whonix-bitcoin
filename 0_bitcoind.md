# Qubes 4 & Whonix 15: Bitcoin Core
Create a VM for running [Bitcoin Core](https://github.com/bitcoin/bitcoin), which will be built from source code and configured to:
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
Bitcoin Core is a [full node](https://en.bitcoin.it/wiki/Full_node), meaning it will verify that all incoming transactions and blocks are following Bitcoin's rules. This allows you to validate transactions without trusting third parties.

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
## Prerequisites
- Read the [README](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/README.md).
## I. Set Up Dom0
### A. In a `dom0` terminal, clone a Whonix workstation TemplateVM.
```
[user@dom0 ~]$ qvm-clone whonix-ws-15 whonix-ws-15-bitcoin
```
### B. Create a gateway.
**Notes:**
- This gateway should be independent of other Whonix gateways to isolate its onion service. See [here](https://www.whonix.org/wiki/Multiple_Whonix-Gateway).
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' \
--prop provides_network='True' --prop vcpus='1' --template whonix-gw-15 sys-bitcoin
```
### C. Create an AppVM, use newly created gateway and template.
1. Create AppVM.

**Note:** You must choose a label color, but it does not have to match this example.

```
[user@dom0 ~]$ qvm-create --label red --prop netvm='sys-bitcoin' \
--template whonix-ws-15-bitcoin bitcoind
```
2. Increase the private volume size.

```
[user@dom0 ~]$ qvm-volume resize bitcoind:private 350G
```
## II. Set Up TemplateVM
### A. In the `whonix-ws-15-bitcoin` terminal, update and install dependencies.
```
user@host:~$ sudo apt update && sudo apt install -y automake autotools-dev build-essential git \
libboost-chrono-dev libboost-filesystem-dev libboost-system-dev libboost-test-dev \
libboost-thread-dev libevent-dev libprotobuf-dev libqrencode-dev libssl-dev libtool libzmq3-dev \
pkg-config protobuf-compiler qttools5-dev qttools5-dev-tools
```
### B. Create system user.
```
user@host:~$ sudo adduser --system bitcoin
Adding system user `bitcoin' (UID 116) ...
Adding new user `bitcoin' (UID 116) with group `nogroup' ...
Creating home directory `/home/bitcoin' ...
```
### C. Shutdown TemplateVM.
```
user@host:~$ sudo poweroff
```
## III. Set Up Gateway.
### A. In a `sys-bitcoin` terminal, find out the gateway IP.
**Note:** Save your gateway IP (`10.137.0.50` in this example) to replace `<gateway-ip>` in later examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
### B. Configure `onion-grater`.
1. Create directory.

```
user@host:~$ sudo mkdir -p /usr/local/etc/onion-grater-merger.d/
```
2. Symlink the provided `bitcoind` profile.

```
user@host:~$ sudo ln -s /usr/share/doc/onion-grater-merger/examples/40_bitcoind.yml /usr/local/etc/onion-grater-merger.d/
```
3. Restart `onion-grater` service.

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

**Note:** At the time of writing the current [release](https://github.com/bitcoin/bitcoin/releases) branch is `0.20`, modify the following steps accordingly if the version has changed.

```
bitcoin@host:~$ git clone --branch 0.20 \
https://github.com/bitcoin/bitcoin ~/bitcoin
Cloning into '/home/user/bitcoin'...
remote: Enumerating objects: 2, done.
remote: Counting objects: 100% (2/2), done.
remote: Total 168496 (delta 1), reused 1 (delta 1), pack-reused 168494
Receiving objects: 100% (168496/168496), 145.29 MiB | 338.00 KiB/s, done.
Resolving deltas: 100% (118758/118758), done.
```
2. Enter the `~/bitcoin` directory and receive signing keys.

```
bitcoin@host:~$ cd ~/bitcoin/
bitcoin@host:~/bitcoin$ gpg --recv-keys $(<contrib/verify-commits/trusted-keys)
gpg: key 0x944D35F9AC3DB76A: public key "Michael Ford (bitcoin-otc) <fanquake@gmail.com>" imported
gpg: key 0xD300116E1C875A3D: public key "MeshCollider <dobsonsa68@gmail.com>" imported
gpg: key 0x3648A882F4316B9B: public key "Marco Falke <marco.falke@tum.de>" imported
gpg: key 0x29D4BCB6416F53EC: 1 duplicate signature removed
gpg: key 0x29D4BCB6416F53EC: 1 signature reordered
gpg: key 0x29D4BCB6416F53EC: public key "Jonas Schnelli <dev@jonasschnelli.ch>" imported
gpg: key 0x860FEB804E669320: public key "Pieter Wuille <pieter.wuille@gmail.com>" imported
gpg: key 0x74810B012346C9A6: public key "Wladimir J. van der Laan <laanwj@visucore.com>" imported
gpg: Total number processed: 6
gpg:               imported: 6
```
3. Verify source code.

**Note:** Your output may not match the example. Just check that it says `Good signature`.

```
bitcoin@host:~/bitcoin$ git verify-commit HEAD
gpg: Signature made Fri 10 Jul 2020 04:45:48 AM UTC
gpg:                using RSA key CFB16E21C950F67FA95E558F2EEB9F5CC09526C1
gpg: Good signature from "Michael Ford (bitcoin-otc) <fanquake@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: E777 299F C265 DD04 7930  70EB 944D 35F9 AC3D B76A
     Subkey fingerprint: CFB1 6E21 C950 F67F A95E  558F 2EEB 9F5C C095 26C1
```
### C. Build Berkeley DB and Bitcoin.
**Note:** This step will take some time and produce a lot of output. This is normal, be patient.

1. Build Berkeley DB using the provided script.

```
bitcoin@host:~/bitcoin$ ./contrib/install_db4.sh `pwd`
```
2. Generate Bitcoin Core configuration script.

```
bitcoin@host:~/bitcoin$ export BDB_PREFIX='/home/bitcoin/bitcoin/db4'; ./autogen.sh
```

**Note:** The next two steps will take some time and produce a lot of output. This is normal, be patient.

3. Configure.

```
bitcoin@host:~/bitcoin$ ./configure BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" \
BDB_CFLAGS="-I${BDB_PREFIX}/include" --prefix=/home/bitcoin
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
bitcoin@host:~$ mousepad ~/.bitcoin/bitcoin.conf
```
2. Paste the following.

**Note:** Be sure to replace `<gateway-ip>` with the information noted earlier.

```
listen=1
onion=<gateway-ip>:9111
onlynet=onion
proxy=<gateway-ip>:9111
txindex=1
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Fix permissions.

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
3. Restart firewall service.

```
user@host:~$ sudo systemctl restart whonix-firewall.service
```
### D. Use `systemd` to keep `bitcoind` running.
1. Create a persistent directory.

```
user@host:~$ sudo mkdir -m 0700 /rw/config/systemd
```
2. Create the service file.

```
user@host:~$ lxsu mousepad /rw/config/systemd/bitcoind.service
```
3. Paste the following.

```
[Unit]
Description=Bitcoin daemon
After=qubes-sysinit.service

[Service]
ExecStart=/home/bitcoin/bin/bitcoind \
  -conf=/home/bitcoin/.bitcoin/bitcoin.conf \
  -daemon \
  -pid=/run/bitcoind/bitcoind.pid
ExecStop=/home/bitcoin/bin/bitcoin-cli stop

RuntimeDirectory=bitcoind
RuntimeDirectoryMode=0710

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
4. Save the file: `Ctrl-S`.
5. Switch back to the terminal: `Ctrl-Q`.
6. Fix permissions.

```
user@host:~$ sudo chmod 0600 /rw/config/systemd/bitcoind.service
```
### E. Enable the service on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ lxsu mousepad /rw/config/rc.local
```
2. Paste the following at the bottom of the file.

```
cp /rw/config/systemd/bitcoind.service /lib/systemd/system/
systemctl daemon-reload
systemctl start bitcoind.service
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
## VI. Create Alias
1. Make an alias in order to control `bitcoind` easier.

```
user@host:~$ echo 'alias bitcoin-cli="sudo -u bitcoin /home/bitcoin/bin/bitcoin-cli"' >> ~/.bashrc
```
2. Source the file.

```
user@host:~$ source ~/.bashrc
```
## VII. Initial Blockchain Download
**Note:** Initial block download can take anywhere from a day to a week (or even more) depending on a number of factors including your hardware and internet connection.

### A. In a `bitcoind` terminal, start the `bitcoind` service.
```
user@host:~$ sudo systemctl start bitcoind
```
## VIII. Final Notes
- Once the initial synchronization has finished you may begin using your Bitcoin node as a backend for other services.
- To check the status of the server: `sudo tail -f /home/bitcoin/.bitcoin/debug.log`
- To interact with the server: `bitcoin-cli --help`
