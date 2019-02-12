# Qubes 4 & Whonix 14: Electrum Personal Server
Create a VM for running an [Electrum Personal Server](https://github.com/chris-belcher/electrum-personal-server) (EPS) which will connect to your `bitcoind` VM. The `eps` VM will be accessible from an Electrum Bitcoin wallet in an offline VM on the same host or remotely via a Tor onion service.
## What is Electrum Personal Server?
EPS is one of the possible server backends for the Electrum Bitcoin wallet. The other implementation covered in these guides is [Electrumx](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md).

Here are some of the differences between EPS and Electrumx:
- EPS is less versatile.
  - EPS requires that each wallet's [MPK](https://bitcoin.stackexchange.com/a/50031) is in its config file.
  - Electrumx can serve any wallet once fully sync'd.
- An EPS VM requires less disk space.
  - EPS VM disk space: 1G
  - Electrumx VM disk space: 40G.
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
[user@dom0 ~]$ qvm-create --label purple --prop maxmem='400' --prop netvm='sys-firewall' --prop provides_network='True' --prop vcpus='1' --template whonix-gw-14 sys-eps
```
### B. Create AppVM.
1. Create the AppVM for EPS with the newly created gateway, using the `whonix-ws-14-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label red --prop maxmem='400' --prop netvm='sys-eps' --prop vcpus='1' --template whonix-ws-14-bitcoin eps
```
2. Enable `eps` service.

```
[user@dom0 ~]$ qvm-service --enable eps eps
```
### C. Create rpc policy to allow comms from `eps` to `bitcoind`.
```
[user@dom0 ~]$ echo 'eps bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.bitcoind
```
### D. Get IP of the `sys-eps` gateway.
**Note:**
- Save your gateway IP (`10.137.0.50` in this example) for later to replace `<gateway-ip>` in examples.

```
[user@dom0 ~]$ qvm-prefs sys-eps ip
10.137.0.50
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-14-bitcoin` terminal, update and install dependency.
```
user@host:~$ sudo apt update && sudo apt install -y python-virtualenv
```
### B. Create a system user.
```
user@host:~$ sudo adduser --system eps
Adding system user `eps' (UID 117) ...
Adding new user `eps' (UID 117) with group `nogroup' ...
Creating home directory `/home/eps' ...
```
### C. Use `systemd` to keep `electrum-personal-server` running.
1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /lib/systemd/system/eps.service
```

2. Paste the following.

```
[Unit]
Description=Electrum Personal Server
ConditionPathExists=/var/run/qubes-service/eps
After=qubes-sysinit.service

[Service]
ExecStart=/home/eps/epsvenv/bin/electrum-personal-server \
-l /home/eps/.eps/debug.log --loglevel DEBUG /home/eps/.eps/config.cfg

User=eps
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
user@host:~$ sudo chmod 0644 /lib/systemd/system/eps.service
```

5. Enable the service.

```
user@host:~$ sudo systemctl enable eps.service
Created symlink /etc/systemd/system/multi-user.target.wants/eps.service → /lib/systemd/system/eps.service.
```
### D. Shutdown TemplateVM.
```
user@host:~$ sudo shutdown now
```
## III. Set up Bitcoin.
### A. In a `bitcoind` terminal, create RPC credentials for `eps` to communicate with `bitcoind`.
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
# EPS Auth
rpcauth=<rpc-user>:<hashed-pass>
# EPS Wallet
wallet=eps
```
3. Save the file and switch back to the terminal.
4. Restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```
## III. Install EPS
### A. Download and verify the EPS source code.
1. Download the latest EPS [release](https://github.com/chris-belcher/electrum-personal-server/releases) and signature.

**Note:**
- At the time of writing the most recent version of EPS is `v0.1.6`, modify the following steps accordingly if the version has changed.

```
user@host:~$ curl -LO "https://github.com/chris-belcher/electrum-personal-server/archive/eps-v0.1.6.tar.gz" -O "https://github.com/chris-belcher/electrum-personal-server/releases/download/eps-v0.1.6/eps-v0.1.6.tar.gz.asc"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   150    0   150    0     0     50      0 --:--:--  0:00:02 --:--:--    50
100 69510    0 69510    0     0  11483      0 --:--:--  0:00:06 --:--:-- 58608
100   612    0   612    0     0    998      0 --:--:-- --:--:-- --:--:--   998
100   819  100   819    0     0    250      0  0:00:03  0:00:03 --:--:--   460
```
2. Receive signing key.

**Note:**
- You can verify the key fingerprint in the [release notes](https://github.com/chris-belcher/electrum-personal-server/releases).

```
user@host:~$ gpg --recv-keys "0A8B 038F 5E10 CC27 89BF CFFF EF73 4EA6 77F3 1129"
gpg: keybox '/home/user/.gnupg/pubring.kbx' created
key 0xEF734EA677F31129:
4 signatures not checked due to missing keys
gpg: /home/user/.gnupg/trustdb.gpg: trustdb created
gpg: key 0xEF734EA677F31129: public key "Chris Belcher <false@email.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify source code.

**Note:**
- Your output may not match the example. Just check that it says `Good signature`.

```
user@host:~$ gpg --verify eps-v0.1.6.tar.gz.asc eps-v0.1.6.tar.gz
gpg: Signature made Thu 15 Nov 2018 10:31:06 PM UTC
gpg:                using RSA key 0xEF734EA677F31129
gpg: Good signature from "Chris Belcher <false@email.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 0A8B 038F 5E10 CC27 89BF  CFFF EF73 4EA6 77F3 1129
```
4. Extract and enter directory.

```
user@host:~$ tar -C ~ -xf eps-v0.1.6.tar.gz
```
### C. Install EPS.
1. Create virtual environment.

```
user@host:~$ virtualenv -p python3 ~/epsvenv
Already using interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/user/epsvenv/bin/python3
Also creating executable in /home/user/epsvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
2. Source the virtual environment and enter the EPS directory.

```
user@host:~$ source ~/epsvenv/bin/activate
(epsvenv) user@host:~$ cd ~/electrum-personal-server-eps-v0.1.6/
(epsvenv) user@host:~/electrum-personal-server-eps-v0.1.6$ python setup.py install
```
3. Deactivate virtual environment, return to home dir, and make virtual environment relocatable.

```
(epsvenv) user@host:~/electrum-personal-server-eps-v0.16$ deactivate
user@host:~/electrum-personal-server-eps-v0.16$ cd
user@host:~$ virtualenv -p python3 --relocatable ~/epsvenv/
```
### D. Relocate virtual environment directory.
```
user@host:~$ sudo cp -r ~/epsvenv/ /home/eps/
```
## IV. Set Up EPS
### A. Remain in an `eps` terminal, configure EPS data directory.
1. Create EPS's data directory and subdirectories.

```
user@host:~$ sudo mkdir -m 0700 /home/eps/.eps
user@host:~$ sudo mkdir -m 0700 /home/eps/.eps/certs
```
2. Create configuration file.

```
user@host:~$ sudo kwrite /home/eps/.eps/config.cfg
```
3. Paste the following.

**Notes:**
- Be sure to replace `<rpc-user>` and `<rpc-pass>` with the information noted earlier.
- For a verbose desciption of these settings, look to the file: [`~/electrum-personal-server-eps-v0.1.6/config.cfg_sample`](https://github.com/chris-belcher/electrum-personal-server/blob/master/config.cfg_sample).
- At this point you may add your Electrum master public keys or individual addresses to the config file.

```
## Electrum Personal Server configuration file

[master-public-keys]
## Add electrum master public keys to this section.
## Create a wallet in electrum then go Wallet -> Information to get the MPK.

# any_name_works = xpub661MyMwAqRbcFseXCwRdRVkhVuzEiskg4QUp5XpUdNf2uGXvQmnD4zcofZ1MN6Fo8PjqQ5cemJQ39f7RTwDVVputHMFjPUn8VRp2pJQMgEF

## Multiple master public keys maybe added by simply adding another line
# my_second_wallet = xpubanotherkey

## Multisig wallets use format `required-signatures [list of master pub keys]`
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
certfile = /home/eps/.eps/certs/server.crt
keyfile = /home/eps/.eps/certs/server.key
```
4. Save the file and switch back to the terminal.
5. Fix permissions.

```
user@host:~$ sudo chmod 0600 /home/eps/.eps/config.cfg
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
4. Copy all cert files to the EPS data directory.

```
user@host:~$ sudo install -m 0600 -t /home/eps/.eps/certs/ server.{crt,csr,key}
```
### C. Change owner.
```
user@host:~$ sudo chown -R eps:nogroup /home/eps/
```
## V. Set Up Communication Channels
### A. Remain in an `eps` terminal, open communication with `bitcoind` on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:8332,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm bitcoind qubes.bitcoind\" &" >> /rw/config/rc.local'
```
2. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### B. Set up `qubes-rpc` for `eps`.
**Note:**
- This only creates the possibility for other VMs to communicate with `eps`, it does not yet give them permission.

1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```
2. Create `qubes.eps` action file.

```
user@host:~$ sudo sh -c 'echo "socat STDIO TCP:127.0.0.1:50002" > /rw/usrlocal/etc/qubes-rpc/qubes.eps'
```
3. Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/usrlocal/etc/qubes-rpc/qubes.eps
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
### D. Find out the IP address of the `eps` VM.
**Note:**
- Save the `eps` IP (`10.137.0.51` in this example) for later to replace `<eps-ip>` in examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.51
```
## VI. Set Up Gateway.
### A. In a `sys-eps` terminal, set up Tor.
1. Edit the Tor configuration file.

```
user@host:~$ sudo kwrite /rw/usrlocal/etc/torrc.d/50_user.conf
```
2. Paste the following.

**Note:**
- Be sure to replace `<eps-ip>` with the information noted earlier.

```
HiddenServiceDir /var/lib/tor/eps/
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
user@host:~$ sudo cat /var/lib/tor/eps/hostname
electrumpersonalservertoronionserviceaddressxxxxxxxxxxxx.onion
```
## VII. Initial EPS Synchronization
### A. In an `eps` terminal, start the `eps` service.
```
user@host:~$ sudo systemctl start eps.service
```
### B. Check the status of the server.
1. For an INFO level log.

```
user@host:~$ sudo journalctl -fu eps
```
2. For a DEBUG level log.

```
user@host:~$ sudo tail -f /home/eps/.eps/debug.log
```
## VIII. Final Notes
- Initial sync will take about 10-20 minutes for each wallet pubkey in your EPS config file.
- Once the sync is finished you may connect your Electrum wallet via the Tor onion address.
- To connect an offline Electrum wallet from a separate VM (split-electrum) use the guide: [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md).
