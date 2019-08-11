# Qubes 4 & Whonix 15: Electrs
Create a VM for running an [Electrs](https://github.com/romanz/electrs) server which will connect to your `bitcoind` VM. The `electrs` VM will be accessible from an Electrum Bitcoin wallet in an offline VM on the same host or remotely via a Tor onion service.
## What is Electrs?
Electrs is one of the possible server backends for the Electrum Bitcoin wallet. The other implementations covered in these guides are [Electrum Personal Server](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md) (EPS), and [Electrumx](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md).

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
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' --prop provides_network='True' --prop vcpus='1' --template whonix-gw-15 sys-electrs
```
### B. Create AppVM.
1. Create the AppVM for Electrumx with the newly created gateway, using the `whonix-ws-15-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM, but will significantly slow down initial sync.


```
[user@dom0 ~]$ qvm-create --label red --prop maxmem='800' --prop netvm='sys-electrs' --prop vcpus='1' --template whonix-ws-15-bitcoin electrs
```
2. Increase private volume size and enable `electrs` service.

**Note:**
- The disk usage will settle at somewhere under 60G, but during initial download and compaction the disk usage can get to under 140G.

```
[user@dom0 ~]$ qvm-volume resize electrs:private 140G
[user@dom0 ~]$ qvm-service --enable electrs electrs
```

### C. Create rpc policy to allow comms from `electrs` to `bitcoind`.
```
[user@dom0 ~]$ echo 'electrs bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.bitcoind_8332
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-15-bitcoin` terminal, update and install dependencies.
```
user@host:~$ sudo apt update && sudo apt install -y cargo clang cmake rustc
```
### B. Create a system user.
```
user@host:~$ sudo adduser --system electrs
Adding system user `electrs' (UID 117) ...
Adding new user `electrs' (UID 117) with group `nogroup' ...
Creating home directory `/home/electrs' ...
```
### C. Use `systemd` to keep `electrs` running.
1. Create `systemd` service file.

```
user@host:~$ lxsu mousepad /lib/systemd/system/electrs.service
```
2. Paste the following.

```
[Unit]
Description=Electrum Rust Server
ConditionPathExists=/var/run/qubes-service/electrs
After=qubes-sysinit.service

[Service]
User=electrs

Type=simple
ExecStart=/home/electrs/electrs/target/release/electrs -vvvv \
  --db-dir /home/electrs/.electrs/electrs-db \
  --electrum-rpc-addr="0.0.0.0:50001" \
  --index-batch-size=10 --jsonrpc-import
Restart=on-failure
RestartSec=60
Environment="RUST_BACKTRACE=1"

PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Enable the service.

```
user@host:~$ sudo systemctl enable electrs.service
Created symlink /etc/systemd/system/multi-user.target.wants/electrs.service → /lib/systemd/system/electrs.service.
```
### D. Shutdown the TemplateVM.
```
user@host:~$ sudo poweroff
```
## III. Set up RPC.
### A. In a `bitcoind` terminal, create RPC credentials for `electrs` to communicate with `bitcoind`.
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
# Electrs Auth
rpcauth=<rpc-user>:<hashed-pass>
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```
## IV. Install Electrs
### A. Download and verify Electrs.
1. Switch to user `electrs` and change to home directory.

```
user@host:~$ sudo -H -u electrs bash
electrs@host:/home/user$ cd
```
2. Clone the Electrs [repository](https://github.com/romanz/electrs).

**Note:**
- The current version of Electrs is `v0.7.1`, modify the following steps accordingly if the version has changed.

```
electrs@host:~$ git clone -b v0.7.1 https://github.com/romanz/electrs ~/electrs
Cloning into 'electrs'...
remote: Enumerating objects: 26, done.
remote: Counting objects: 100% (26/26), done.
remote: Compressing objects: 100% (21/21), done.
remote: Total 5649 (delta 5), reused 16 (delta 4), pack-reused 5623
Receiving objects: 100% (5649/5649), 1.44 MiB | 138.00 KiB/s, done.
Resolving deltas: 100% (3946/3946), done.
Note: checking out 'fcbf16b9f1932ca7fe91d1a172301cb64dc4cfe0'.
```
3. Receive signing key.

**Note:**
- You can verify Roman Zeyde's key fingerprint [here](https://gist.github.com/romanz/bae9a2ef48693830a805ca56940baebd).

```
electrs@host:~$ gpg --recv-keys 15C8C3574AE4F1E25F3F35C587CAE5FA46917CBB
gpg: keybox '/home/electrs/.gnupg/pubring.kbx' created
gpg: /home/electrs/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x87CAE5FA46917CBB: public key "Roman Zeyde <me@romanzey.de>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```
4. Enter `electrs/` directory.

```
electrs@host:~$ cd ~/electrs
```
5. Verify the latest version tag.

**Note:**
- Your output may not match the example. Just check that it says `Good signature`.

```
electrs@host:~/electrs$ git verify-tag v0.7.1
gpg: Signature made Sat 27 Jul 2019 02:43:19 PM UTC
gpg:                using ECDSA key 15C8C3574AE4F1E25F3F35C587CAE5FA46917CBB
gpg:                issuer "me@romanzey.de"
gpg: Good signature from "Roman Zeyde <me@romanzey.de>" [unknown]
gpg:                 aka "Roman Zeyde <roman.zeyde@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 15C8 C357 4AE4 F1E2 5F3F  35C5 87CA E5FA 4691 7CBB
```
### B. Build `electrs`.
**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
electrs@host:~/electrs$ cargo build --release
```
### C. Set up Electrs.
1. Create necessary directories.

```
electrs@host:~$ mkdir -m 0700 ~/.{bitcoin,electrs}
```
2. Create cookie file.

```
electrs@host:~$ mousepad ~/.bitcoin/.cookie
```
3. Paste the following.

**Notes:**
- Replace `<rpc-user>` and `<rpc-pass>` with the information noted earlier.

```
<rpc-user>:<rpc-pass>
```
4. Save the file: `Ctrl-S`.
5. Switch back to the terminal: `Ctrl-Q`.
### D. Switch back to main user account.
```
electrs@host:~$ exit
```
## VI. Set Up Communication Channels
### A. Remain in an `electrs` terminal, open communication with `bitcoind` on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:8332,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm bitcoind qubes.bitcoind_8332\" &" >> /rw/config/rc.local'
```
2. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### B. Set up `qubes-rpc` for `electrs`.
**Note:**
- This only creates the possibility for other VMs to communicate with `electrs`, it does not yet give them permission.

1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```
2. Create `qubes.electrum_50001` action file.

```
user@host:~$ sudo sh -c 'echo "socat STDIO TCP:127.0.0.1:50001" > /rw/usrlocal/etc/qubes-rpc/qubes.electrum_50001'
```
### C. Open firewall for Tor onion service.
1. Make persistent directory for new firewall rules.

```
user@host:~$ sudo mkdir -m 0755 /rw/config/whonix_firewall.d
```
2. Configure firewall.

```
user@host:~$ sudo sh -c 'echo "EXTERNAL_OPEN_PORTS+=\" 50001 \"" >> /rw/config/whonix_firewall.d/50_user.conf'
```
3. Restart firewall service.

```
user@host:~$ sudo systemctl restart whonix-firewall.service
```
### D. Find out the IP address of the `electrs` VM.
**Note:**
- Save the `electrs` IP (`10.137.0.50` in this example) to replace `<electrs-ip>` in later examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
## VII. Initial Electrumx Synchronization
### A. In an `electrs` terminal, start the `electrs` service.
```
user@host:~$ sudo systemctl start electrs.service
```
## VIII. Set Up Gateway.
### A. In a `sys-electrs` terminal, set up Tor.
1. Edit the Tor configuration file.

```
user@host:~$ lxsu mousepad /rw/usrlocal/etc/torrc.d/50_user.conf
```
2. Paste the following.

**Note:**
- Be sure to replace `<electrs-ip>` with the information noted earlier.

```
HiddenServiceDir /var/lib/tor/electrs/
HiddenServicePort 50001 <electrs-ip>:50001
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
user@host:~$ sudo cat /var/lib/tor/electrs/hostname
electrstoronionserviceaddressxxxxxxxxxxxxxxxxxxxxxxxxx.onion
```
## IX. Final Notes
- The intial sync can take anywhere from one hour to multiple hours depending on a number of factors including your hardware and resources dedicated to the `electrs` VM.
- Once the sync is finished you may connect your Electrum wallet via the Tor onion address and port `50001`.
- To check the status of the server: `sudo journalctl -fu electrs`
- To connect an offline Electrum wallet from a separate VM (split-electrum) use the guide: [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md).
