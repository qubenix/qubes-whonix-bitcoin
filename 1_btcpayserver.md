# Qubes 4 & Whonix 15: BTCPayServer
Create a VM for running a [BTCPayServer](https://github.com/btcpayserver/btcpayserver) which will connect to your `bitcoind` VM. The `btcpayserver` VM will be accessible locally, or remotely via a Tor onion service.
## What is BTCPayServer
## Why do this?
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
--prop provides_network='True' --prop vcpus='1' --template whonix-gw-15 sys-btcpayserver
```
### B. Create TemplateVM.
```
[user@dom0 ~]$ qvm-clone whonix-ws-15 whonix-ws-15-dotnet
```
### C. Create AppVM.
1. Create the AppVM for BTCPayServer with the newly created gateway, using the `whonix-ws-15-dotnet` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label red --prop maxmem='1000' --prop netvm='sys-btcpayserver' \
--prop vcpus='1' --template whonix-ws-15-dotnet btcpayserver
```
2. Increase private volume size.

```
[user@dom0 ~]$ qvm-volume resize btcpayserver:private 20G
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-15-dotnet` terminal, add source for Microsoft.
```
user@host:~$ sudo sh -c 'echo "deb [arch=amd64,arm64,armhf] \
https://packages.microsoft.com/debian/10/prod buster main" > /etc/apt/sources.list.d/microsoft-prod.list'
```
### B. Get Microsoft's signing key.
1. Download the signing key.

```
user@host:~$ curl -O https://packages.microsoft.com/keys/microsoft.asc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   983  100   983    0     0    269      0  0:00:03  0:00:03 --:--:--   269
```
2. Verify the key fingerprint.

**Note:**
- If key fingerprint does not match example, then something is awry. See Microsoft's [official site](https://dotnet.microsoft.com/download/linux-package-manager/debian10/sdk-current) for help.

```
user@host:~$ gpg --with-fingerprint microsoft.asc
gpg: WARNING: no command supplied.  Trying to guess what you mean ...
pub   rsa2048/0xEB3E94ADBE1229CF 2015-10-28 [SC]
      Key fingerprint = BC52 8686 B50D 79E3 39D3  721C EB3E 94AD BE12 29CF
uid                             Microsoft (Release signing) <gpgsecurity@microsoft.com>
```
3. Add to apt keyring.

```
user@host:~$ sudo apt-key add microsoft.asc
OK
```
### C. Update and install software.
```
user@host:~$ sudo apt update && sudo apt install -y dotnet-sdk-2.1 dotnet-sdk-3.0
```
### D. Add environment variable.
1. Create a profile file.

**Note:**
- Microsoft will collect data about your usage of `dotnet` if you do not add this variable to your environment.

```
user@host:~$ lxsu mousepad /etc/profile.d/10_dotnet_cli_telemetry_output.sh
```
2. Paste the following.

```
#!/bin/sh

export DOTNET_CLI_TELEMETRY_OPTOUT="1"
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Make the file executable.

```
user@host:~$ sudo chmod +x /etc/profile.d/10_dotnet_cli_telemetry_output.sh
```
### E. Create a system user.
```
user@host:~$ sudo adduser --system btcpayserver
Adding system user `btcpayserver' (UID 117) ...
Adding new user `btcpayserver' (UID 117) with group `nogroup' ...
Creating home directory `/home/btcpayserver' ...
```
### F. Shutdown TemplateVM.
```
user@host:~$ sudo poweroff
```
## III. Set up Bitcoin.
### A. In a `bitcoind` terminal, create RPC credentials for `btcpayserver` to communicate with `bitcoind`.
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
# BTCPayServer Auth
rpcauth=<rpc-user>:<hashed-pass>
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```
## IV. Install and configure BTCPayServer.
### A. In a `btcpayserver` terminal, get Nicolas Dorier's signing key.
1. Switch to user `btcpayserver` and enter home directory.

```
user@host:~$ sudo -H -u btcpayserver bash
btcpayserver@host:/home/user$ cd
```
2. Download the key from [keybase](https://keybase.io/nicolasdorier).

```
btcpayserver@host:~$ curl -o nicolas.asc \
https://keybase.io/nicolasdorier/pgp_keys.asc?fingerprint=ab4cfa9895aca0dbe27f6b346618763ef09186fe
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3186  100  3186    0     0   1093      0  0:00:02  0:00:02 --:--:--  1092
```
3. Verify fingerprint.

**Note:**
- If key fingerprint does not match example, then something is awry. Try to delete `nicolas.asc` and download again, or follow the link from the previous step to manually download the key.

```
btcpayserver@host:~$ gpg --with-fingerprint nicolas.asc
gpg: WARNING: no command supplied.  Trying to guess what you mean ...
pub   rsa4096/0x6618763EF09186FE 2018-09-05 [SC] [expires: 2034-09-01]
      Key fingerprint = AB4C FA98 95AC A0DB E27F  6B34 6618 763E F091 86FE
uid                             Nicolas Dorier <nicolas.dorier@gmail.com>
sub   rsa4096/0xE032B92DFA79E1A0 2018-09-05 [E] [expires: 2034-09-01]
```
4. Import the key.

```
btcpayserver@host:~$ gpg --import nicolas.asc
gpg: key 0x6618763EF09186FE: public key "Nicolas Dorier <nicolas.dorier@gmail.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```
### B. Install NBXplorer.
1. Clone the NBXplorer repository.

```
btcpayserver@host:~$ git clone -b latest \
https://github.com/dgarage/NBXplorer ~/NBXplorer
Cloning into 'NBXplorer'...
remote: Enumerating objects: 64, done.
remote: Counting objects: 100% (64/64), done.
remote: Compressing objects: 100% (50/50), done.
remote: Total 6843 (delta 29), reused 40 (delta 11), pack-reused 6779
Receiving objects: 100% (6843/6843), 1.73 MiB | 172.00 KiB/s, done.
Resolving deltas: 100% (5019/5019), done.
```
2. Change the directory and verify the download.

**Note:**
- Your output may not match the example. Just check that it says `Good signature`.

```
btcpayserver@host:~$ cd ~/NBXplorer/
btcpayserver@host:~/NBXplorer$ git verify-commit HEAD
gpg: Signature made Wed 23 Oct 2019 05:43:46 AM UTC
gpg:                using RSA key AB4CFA9895ACA0DBE27F6B346618763EF09186FE
gpg: Good signature from "Nicolas Dorier <nicolas.dorier@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: AB4C FA98 95AC A0DB E27F  6B34 6618 763E F091 86FE
```
3. Build NBXplorer.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
btcpayserver@host:~/NBXplorer$ export DOTNET_CLI_TELEMETRY_OUTPUT="1"; ./build.sh
```
### C. Install BTCPayServer.
1. Clone the BTCPayServer repository.

```
btcpayserver@host:~/NBXplorer$ git clone -b v1.0.3.137 \
https://github.com/btcpayserver/btcpayserver ~/btcpayserver
Cloning into '/home/btcpayserver/btcpayserver'...
remote: Enumerating objects: 24, done.
remote: Counting objects: 100% (24/24), done.
remote: Compressing objects: 100% (19/19), done.
remote: Total 26289 (delta 3), reused 16 (delta 3), pack-reused 26265
Receiving objects: 100% (26289/26289), 13.17 MiB | 332.00 KiB/s, done.
Resolving deltas: 100% (20063/20063), done.
Note: checking out 'aab7c085007495bdd8f8ae66f25337eedf4bd79c'.
...
```
2. Change the directory and verify the download.

**Note:**
- Your output may not match the example. Just check that it says `Good signature`.

```
btcpayserver@host:~/NBXplorer$ cd ~/btcpayserver/
btcpayserver@host:~/btcpayserver$ git verify-commit HEAD
gpg: Signature made Thu 07 Nov 2019 09:59:51 AM UTC
gpg:                using RSA key AB4CFA9895ACA0DBE27F6B346618763EF09186FE
gpg: Good signature from "Nicolas Dorier <nicolas.dorier@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: AB4C FA98 95AC A0DB E27F  6B34 6618 763E F091 86FE
```
3. Build BTCPayServer.

**Note:**
- This step will take some time and produce a lot of output. This is normal, be patient.

```
btcpayserver@host:~/btcpayserver$ export DOTNET_CLI_TELEMETRY_OUTPUT="1"; ./build.sh
```
### D. Configure NBXplorer.
1. Create NBXplorer's data directory and configuration file.

```
btcpayserver@host:~$ mkdir -m 0700 ~/.nbxplorer
btcpayserver@host:~$ mkdir -m 0700 ~/.nbxplorer/Main
btcpayserver@host:~$ mousepad ~/.nbxplorer/Main/settings.config
```
2. Paste the following.

**Note:**
- Be sure to replace `<rpc-user>` and `<rpc-pass>` with the information noted earlier.

```
btc.rpc.auth=<rpc-user>:<rpc-pass>
mainnet=1
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Fix permissions.

```
btcpayserver@host:~$ chmod 0600 ~/.nbxplorer/Main/settings.config
```
### E.Configure BTCPayServer.
1. Create BTCPayServer's data directory and configuration file.

```
btcpayserver@host:~$ mkdir -m 0700 ~/.btcpayserver
btcpayserver@host:~$ mkdir -m 0700 ~/.btcpayserver/Main
btcpayserver@host:~$ mousepad ~/.btcpayserver/Main/settings.config
```
2. Paste the following.

```
bind=0.0.0.0
```
3. Save the file: `Ctrl-S`.
4. Switch back to the terminal: `Ctrl-Q`.
5. Fix permissions.

```
btcpayserver@host:~$ chmod 0600 ~/.btcpayserver/Main/settings.config
```
## V. Configure `systemd`.
## VI. Configure firewall.