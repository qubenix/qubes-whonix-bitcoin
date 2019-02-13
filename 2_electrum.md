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
[user@dom0 ~]$ qvm-create --label black --prop netvm='' --template whonix-ws-14-bitcoin electrum
```
### B. Create rpc policy to allow comms from `electrum` to `eps` or `electrumx`.
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
## II. Set Up Template
### A. In a `whonix-ws-14-bitcoin` terminal, modify Debian sources.
1. Add unstable source.

**Notes:**
- These instructions are taken from an older version of the [Whonix wiki](https://www.whonix.org/wiki/Electrum).
- You should use the unstable repo since it has a newer version of Electrum (`3.2.3`) than backports (`3.1.3`) with substantial [improvements](https://github.com/spesmilo/electrum/blob/master/RELEASE-NOTES).

```
user@host:~$ sudo sh -c "echo 'deb tor+http://vwakviie2ienjx6t.onion/debian sid main' > /etc/apt/sources.list.d/unstable.list"
```
2. Make `stretch` the default repo.

```
user@host:~$ sudo sh -c "echo 'APT::Default-Release \"stretch\";' > /etc/apt/apt.conf.d/70defaultrelease"
```
### B. Update.
```
user@host:~$ sudo apt update
```
### C. Install `electrum`.
```
user@host:~$ sudo apt install -y electrum/sid
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
    "oneserver": true,
    "server": "127.0.0.1:50002:s",
}
```
4. Save the file and switch back to the terminal.
5. Fix permissions.

```
user@host:~$ chmod 0600 ~/.electrum/config
```
## IV. Final Notes
- Once your `eps` or `electrumx` server has sync'd you will be able to use your `electrum` wallet.
- For more information on using the Electrum wallet see the [official documentation](http://docs.electrum.org/en/latest/).
