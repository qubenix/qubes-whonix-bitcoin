# Qubes 4 & Whonix 14: Bitcoin Isolation Guides
A series of guides that use the Qubes [security by isolation](https://www.qubes-os.org/security/goals/) model, combined with [Whonix](https://www.whonix.org) for additional hardening and anonymity features, to give users a safe system for using Bitcoin in multiple different ways.

Each application will run in its own Whonix VM, and in the case of Electrum and JoinMarket, the wallets will have full functionality without any network connection. This is accomplished using Qubes' [`qrexec`](https://www.qubes-os.org/doc/qrexec3/).

Start with [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md), every other guide depends on it. Each guide will list it's own prerequisites.
## Guides
- [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md)
  - Build a [Bitcoin Core](https://github.com/bitcoin/bitcoin) full node configured to:
    - Allow other VMs to connect when given permission from `dom0`.
    - Communicate only over Tor.
    - Index all transactions.
    - Prefer Tor onion endpoints, use them exclusively if possible.
    - Use an ephemeral Tor onion address for serving peers.
    - Utilize Tor stream isolation.
- [`1_electrum-personal-server.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md)
  - Install an [Electrum Personal Server](https://github.com/chris-belcher/electrum-personal-server) configured to:
    - Allow a local Electrum wallet to connect from an offline VM.
    - Allow a remote Electrum wallet to connect via a Tor onion service.
    - Use the [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md) VM as its backend.
- [`1_electrumx.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md)
  - Install an [Electrumx](https://github.com/kyuupichan/electrumx) server configured to:
    - Allow a local Electrum wallet to connect from an offline VM.
    - Allow a remote Electrum wallet to connect via a Tor onion address.
    - Deny peer discovery.
    - Keep Tor onion address private.
    - Use the [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md) VM as its backend.
- [`1_joinmarket.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_joinmarket.md)
  - Install [JoinMarket](https://github.com/JoinMarket-Org/joinmarket-clientserver) configured to:
    - Communicate only over Tor.
    - Connnect to only Tor onion endpoints.
    - Run the daemon on the [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md) VM.
    - Utilize Tor stream isolation.
    - Provide full functionality from an offline VM.
- [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md)
  - Install the [Electrum](https://electrum.org) wallet configured to:
    - Connect to either the [`1_electrum-personal-server.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md) or the [`1_electrumx.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md) VM.
    - Provide full functionality from an offline VM.

## Guides To Come
- [BTCPay Server](https://github.com/btcpayserver/btcpayserver)
- [c-Lightning](https://github.com/ElementsProject/lightning)
- [Electrs](https://github.com/romanz/electrs)
- [LND](https://github.com/LightningNetwork/lnd)
- [Wasabi Wallet](https://wasabiwallet.io/)

## Guide Numbering Legend
- `0`: No prerequisites.
- `1`: Requires a `0` guide.
- `2`: Requires a `1` and a `0` guide.
- ... so on.

## Git Mirrors
- https://github.com/qubenix/qubes-whonix-bitcoin
- http://qubenixibxoyldm3l3a5fobreaydmvdweqqojllutyyi4vgtbmugvhad.onion/qubenix/qubes-whonix-bitcoin
