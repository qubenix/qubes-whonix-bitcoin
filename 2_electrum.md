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
### A. In a `bitcoind` terminal, download the Electrum AppImage and signature.
1. Download the latest Electrum [appimage and signature](https://download.electrum.org/3.3.8/).

**Note:**
- At the time of writing the most recent version of Electrum is `3.3.8`, modify the following steps accordingly if the version has changed.
- Due to the Cloudflare settings on https://electrum.org, downloading via the command line is not currently possible.

```
user@host:~$ torbrowser https://download.electrum.org/3.3.8/electrum-3.3.8-x86_64.AppImage https://download.electrum.org/3.3.8/electrum-3.3.8-x86_64.AppImage.asc
```
2. Click `Yes` in the pop-up.
3. Click `Download file` in the next pop-up.
4. Select `Save File`, then click `OK`.
5. File should be named: `electrum-3.3.8-x86_64.AppImage`. Click `Save`.
6. Switch to the other tab on the browser. It should show some text that begins with `-----BEGIN PGP SIGNATURE-----`.
7. Save the page: `Ctrl-S`.
8. File should be named: `electrum-3.3.8-x86_64.AppImage.asc`. Click `Save`.
9. Once downloads have finished, close the browser: `Ctrl-Q`.
### B. In a `bitcoind` terminal, verify Electrum download.
1. Receive signing key.

**Note:**
- You can verify the Thomas Voegtlin's key fingerprint on the Electrum [about page](https://electrum.org/#about).

```
user@host:~$ gpg --recv-keys 6694D8DE7BE8EE5631BED9502BD5824B7F9470E6
gpg: key 0x2BD5824B7F9470E6: 212 signatures not checked due to missing keys
gpg: key 0x2BD5824B7F9470E6: public key "Thomas Voegtlin (https://electrum.org) <thomasv@electrum.org>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
2. Enter `Downloads/` directory, and verify the appimage.

```
user@host:~$ cd ~/.tb/tor-browser/Browser/Downloads/
user@host:~/.tb/tor-browser/Browser/Downloads$ gpg --verify electrum-3.3.8-x86_64.AppImage.asc
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
3. Make the appimage executable.

```
user@host:~/.tb/tor-browser/Browser/Downloads$ chmod +x electrum-3.3.8-x86_64.AppImage
```
### B. Move appimage to the `electrum` VM.
**Note:**
- Select `electrum` from the `dom0` pop-up.

```
user@host:~/.tb/tor-browser/Browser/Downloads$ qvm-move electrum-3.3.8-x86_64.AppImage
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
4. Save the file and switch back to the terminal.
5. Fix permissions.

```
user@host:~$ chmod 0600 ~/.electrum/config
```
### C. Add executable to `$PATH`.
1. Make `bin/` directory.

```
user@host:~$ mkdir -m 0700 ~/bin
```
2. Copy executable to `bin/` directory.

```
user@host:~$ cp electrum-3.3.8-x86_64.AppImage ~/bin/electrum
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
