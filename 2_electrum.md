# Qubes 4 & Whonix 15: Electrum
Create a VM without networking to host an [Electrum](https://electrum.org) Bitcoin wallet. The offline `electrum` VM will communicate with either an `electrum-personal-server` or `electrumx` VM using Qubes' [`qrexec`](https://www.qubes-os.org/doc/qrexec3/).
## What is Electrum?
Electrum is a popular lightweight Bitcoin wallet based on a client-server protocol. See the [Bitcoin wiki](https://en.bitcoin.it/wiki/Electrum) for a more detailed explanation of Electrum.
## Why Do This?
This increases the privacy and security of your Electrum wallet while still maintaining full functionality. Enhanced privacy is achieved by preventing data leakage to server operators, and security is improved by removing the need for a network connection on the wallet VM.
## Prerequisites
- To complete this guide you must have first completed:
  - [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md)
  - You will also need either one of these, but not both:
    - [`1_electrum-personal-server.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md)
    - [`1_electrumx.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md)

## I. Set Up Dom0
### A. In a `dom0` terminal, create AppVM.
1. Create an AppVM for Electrum with no networking, using the `whonix-ws-15` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label black --prop maxmem='800' --prop netvm='' --prop vcpus='1' --template whonix-ws-15 electrum
```
### B. Create rpc policy to allow comms from `electrum` to `electrum-personal-server` or `electrumx` VM.
1. Allow `electrum` to communicate with `electrum-personal-server`.

**Note:**
- Skip this step if you did not install `electrum-personal-server` as your server VM.

```
[user@dom0 ~]$ echo 'electrum electrum-personal-server allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.electrum-personal-server
```
2. Allow `electrum` to communicate with `electrumx`.

**Note:**
- Skip this step is you did not install `electrumx` as your server VM.

```
[user@dom0 ~]$ echo 'electrum electrumx allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.electrumx_50002
```
## II. Install Electrum
### A. In a `bitcoind` terminal, download Electrum files.
1. Download the latest Electrum [appimage](https://download.electrum.org/3.3.8/electrum-3.3.8-x86_64.AppImage) and [signature](https://download.electrum.org/3.3.8/electrum-3.3.8-x86_64.AppImage.asc).

**Note:**
- At the time of writing the most recent version of Electrum is `3.3.8`, modify the following steps accordingly if the version has changed.

```
user@host:~$ scurl-download https://download.electrum.org/3.3.8/electrum-3.3.8-x86_64.AppImage \
https://download.electrum.org/3.3.8/electrum-3.3.8-x86_64.AppImage.asc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 47.0M  100 47.0M    0     0   501k      0  0:01:36  0:01:36 --:--:--  697k
100   833    0   833    0     0   1735      0 --:--:-- --:--:-- --:--:--  1735
```
### B. In a `bitcoind` terminal, verify Electrum appimage.
1. Receive signing key.

```
user@host:~$ scurl-download https://raw.githubusercontent.com/spesmilo/electrum/master/pubkeys/ThomasV.asc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4739  100  4739    0     0   2492      0  0:00:01  0:00:01 --:--:--  2491
```
2. Verify the key fingerprint.

**Note:**
- You can verify Thomas Voegtlin's key fingerprint on the Electrum [about page](https://electrum.org/#about).

```
user@host:~$ gpg --with-fingerprint ThomasV.asc
gpg: WARNING: no command supplied.  Trying to guess what you mean ...
pub   rsa4096/0x2BD5824B7F9470E6 2011-06-15 [SC]
      Key fingerprint = 6694 D8DE 7BE8 EE56 31BE  D950 2BD5 824B 7F94 70E6
uid                             ThomasV <thomasv1@gmx.de>
uid                             Thomas Voegtlin <thomasv1@gmx.de>
uid                             Thomas Voegtlin (https://electrum.org) <thomasv@electrum.org>
sub   rsa4096/0x1A25C4602021CD84 2011-06-15 [E]
```
3. Import the key.

```
user@host:~$ gpg --import ThomasV.asc
gpg: key 0x2BD5824B7F9470E6: public key "Thomas Voegtlin (https://electrum.org) <thomasv@electrum.org>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```
4. Verify the appimage.

**Note:**
- Your output may not match the example. Just check that it says `Good signature`.

```
user@host:~$ gpg --verify electrum-3.3.8-x86_64.AppImage.asc
gpg: assuming signed data in 'electrum-3.3.8-x86_64.AppImage'
gpg: Signature made Thu 11 Jul 2019 02:26:15 PM UTC
gpg:                using RSA key 6694D8DE7BE8EE5631BED9502BD5824B7F9470E6
gpg: Good signature from "Thomas Voegtlin (https://electrum.org) <thomasv@electrum.org>" [unknown]
gpg:                 aka "ThomasV <thomasv1@gmx.de>" [unknown]
gpg:                 aka "Thomas Voegtlin <thomasv1@gmx.de>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 6694 D8DE 7BE8 EE56 31BE  D950 2BD5 824B 7F94 70E6
```
5. Make the appimage executable.

```
user@host:~$ chmod +x electrum-3.3.8-x86_64.AppImage
```
### C. Move appimage to the `electrum` VM.
**Note:**
- Select `electrum` from the `dom0` pop-up.

```
user@host:~$ qvm-move electrum-3.3.8-x86_64.AppImage
```
## III. Set Up Electrum
### A. In an `electrum` terminal, open communication with `electrum-personal-server` or `electrumx` on boot.
1. Edit the file `/rw/config/local` for `electrum-personal-server`.

**Note:**
- Skip this step if you did not install `electrum-personal-server` as your server VM.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:50002,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm electrum-personal-server qubes.electrum-personal-server\" &" >> /rw/config/rc.local'
```
2. Edit the file `/rw/config/local` for `electrumx`.

**Note:**
- Skip this step is you did not install `electrumx` as your server VM.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:50002,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm electrumx qubes.electrumx_50002\" &" >> /rw/config/rc.local'
```
3. Execute the file.

```
user@host:~$ sudo /rw/config/rc.local
```
### B. Configure `electrum`.
1. Make data directory.

```
user@host:~$ mkdir -m 0700 ~/.electrum
```
2. Create configuration file.

```
user@host:~$ mousepad ~/.electrum/config
```
3. Paste the following.

```
{
    "auto_connect": false,
    "check_updates": false,
    "oneserver": true,
    "server": "127.0.0.1:50002:s"
}
```
4. Save the file: `Ctrl-S`.
5. Switch back to the terminal: `Ctrl-Q`.
6. Fix permissions.

```
user@host:~$ chmod 0600 ~/.electrum/config
```
### C. Add executable to `$PATH`.
1. Make `bin/` directory.

```
user@host:~$ mkdir -m 0700 ~/bin
```
2. Move executable to `bin/` directory.

```
user@host:~$ mv ~/QubesIncoming/bitcoin/electrum-3.3.8-x86_64.AppImage ~/bin/electrum
```
3. Source profile to fix `$PATH`.

```
user@host:~$ source ~/.profile
```
## IV. Final Notes
- Once your `electrum-personal-server` or `electrumx` server has sync'd you will be able to use your Electrum wallet.
- To launch the wallet:

```
user@host:~$ electrum
```
- To get help on usage:

```
user@host:~$ electrum --help
```
- For more information on using the Electrum wallet see the [official documentation](https://electrum.readthedocs.io/).
