# Qubes 4 & Whonix 14: Electrum
Create a VM without networking to host an [Electrum](https://electrum.org) Bitcoin wallet. The offline `electrum` VM will communicate with either an `electrumx` or `eps` VM using Qubes' [`qrexec`](https://www.qubes-os.org/doc/qrexec3/).
## What is Electrum?
Electrum is a popular lightweight Bitcoin wallet based on a client-server protocol. More info [here](https://en.bitcoin.it/wiki/Electrum).
## Why Do This?
This increases the security and privacy of your Electrum wallet while still maintaining full functionality. Privacy is enhanced by preventing data leakage to server operators, and security is improved by removing the need for an internet connection on the wallet.
## Prerequisites
- To complete this guide you must have first completed:
  - [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md)
  - You will also need either one of these, but not both:
    - [`1_electrum-personal-server.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md)
    - [`1_electrumx.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md)

## I. Set Up Dom0
### A. In a `dom0` terminal, create AppVM.
1. Create an AppVM for Electrum with no networking, using the `whonix-ws-14-bitcoin` TemplateVM.

**Notes:**
- You must choose a label color, but it does not have to match this example.
- It is safe to lower the `maxmem` and `vcpus` on this VM.

```
[user@dom0 ~]$ qvm-create --label black --prop maxmem='800' --prop netvm='' --prop vcpus='1' --template whonix-ws-14 electrum
```
### B. Create rpc policy to allow comms from `electrum` to `eps` or `electrumx` VM.
1. Allow `electrum` to communicate with `eps`.

**Note:**
- Skip this step if you did not install `eps` as your server VM.

```
[user@dom0 ~]$ echo 'electrum eps allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.eps
```
2. Allow `electrum` to communicate with `electrumx`.

**Note:**
- Skip this step is you did not install `electrumx` as your server VM.

```
[user@dom0 ~]$ echo 'electrum electrumx allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.electrumx
```
## II. Install Electrum
### A. In a `bitcoind` terminal, download and verify the Electrum appimage.
1. Download the latest Electrum [appimage and signature](https://electrum.org/#download).

**Note:**
- At the time of writing the most recent version of Electrum is `3.3.4`, modify the following steps accordingly if the version has changed.

```
user@host:~$ curl -O "https://download.electrum.org/3.3.4/electrum-3.3.4-x86_64.AppImage" -O "https://download.electrum.org/3.3.4/electrum-3.3.4-x86_64.AppImage.asc"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 85.1M  100 85.1M    0     0   320k      0  0:04:31  0:04:31 --:--:--  249k
100   833  100   833    0     0    332      0  0:00:02  0:00:02 --:--:--   519
```
2. Receive signing key.

**Note:**
- You can verify the Thomas Voegtlin's key fingerprint on the Electrum [about page](https://electrum.org/#about).

```
user@host:~$ gpg --recv-keys 6694D8DE7BE8EE5631BED9502BD5824B7F9470E6
key 0x2BD5824B7F9470E6:
166 signatures not checked due to missing keys
gpg: key 0x2BD5824B7F9470E6: public key "Thomas Voegtlin (https://electrum.org) <thomasv@electrum.org>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify the appimage.

```
user@host:~$ gpg --verify electrum-3.3.4-x86_64.AppImage.asc electrum-3.3.4-x86_64.AppImage
gpg: Signature made Wed 13 Feb 2019 10:08:30 PM UTC
gpg:                using RSA key 6694D8DE7BE8EE5631BED9502BD5824B7F9470E6
gpg: Good signature from "Thomas Voegtlin (https://electrum.org) <thomasv@electrum.org>" [unknown]
gpg:                 aka "ThomasV <thomasv1@gmx.de>" [unknown]
gpg:                 aka "Thomas Voegtlin <thomasv1@gmx.de>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 6694 D8DE 7BE8 EE56 31BE  D950 2BD5 824B 7F94 70E6
```
4. Make the appimage executable.

```
user@host:~$ chmod a+x electrum-3.3.4-x86_64.AppImage
```
### B. Move appimage to the `electrum` VM.
**Note:**
- Select `electrum` from the `dom0` pop-up.

```
user@host:~$ qvm-move electrum-3.3.4-x86_64.AppImage
```
## III. Set Up Electrum
### A. In an `electrum` terminal, open communication with `eps` or `electrumx` on boot.
1. Edit the file `/rw/config/local` for `eps`.

**Note:**
- Skip this step if you did not install `eps` as your server VM.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:50002,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm eps qubes.eps\" &" >> /rw/config/rc.local'
```
2. Edit the file `/rw/config/local` for `electrumx`.

**Note:**
- Skip this step is you did not install `electrumx` as your server VM.

```
user@host:~$ sudo sh -c 'echo "socat TCP-LISTEN:50002,fork,bind=127.0.0.1 EXEC:\"qrexec-client-vm electrumx qubes.electrumx\" &" >> /rw/config/rc.local'
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
user@host:~$ kwrite ~/.electrum/config
```
3. Paste the following.

```
{
    "auto_connect": true,
    "check_updates": false,
    "oneserver": true,
    "server": "127.0.0.1:50002:s",
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
user@host:~$ cp electrum-3.3.4-x86_64.AppImage ~/bin/electrum
```
3. Source profile to fix `$PATH`.

```
user@host:~$ source ~/.profile
```
## IV. Final Notes
- Once your `eps` or `electrumx` server has sync'd you will be able to use your Electrum wallet.
- To launch the wallet:

```
user@host:~$ electrum
```
- To get help on usage:

```
user@host:~$ electrum --help
```
- For more information on using the Electrum wallet see the [official documentation](http://docs.electrum.org/en/latest/).
