# Qubes 4 & Whonix 14: Bitcoin Isolation Guides
A series of guides that use the Qubes [security by isolation](https://www.qubes-os.org/security/goals/) model, combined with [Whonix](https://www.whonix.org) for additional hardening and anonymity features, to give users a safer environment to use Bitcoin in multiple ways.

Each application will run in its own Whonix VM, and in the case of Electrum and JoinMarket, the wallets will have full functionality without any network connection. This is accomplished using Qubes' [`qrexec`](https://www.qubes-os.org/doc/qrexec3/).

There are some things to consider before following the guides in this series. The first one is that these guides are not fast or necessarily easy to follow. Instead, they strive to thoroughly address security and privacy concerns where practically possible.

The next possible issue is that these guides are very narrow. Each of these services are only set up for Bitcoin's mainnet, and they provide very specific features. There are also sacrifices made of computer resources (memory, processing, etc.) in order to provide more security.

The last shortcoming that should be made obvious is the fact that there is no update method described for any of these guides. For now the user is responsible for figuring out how to upgrade based on the install steps.
## Guides
|   | Guide Numbering Legend                         |
|---|------------------------------------------------|
|`0`| No prerequisites, required by all other guides.|
|`1`| Requires the `0` guide.                        |
|`2`| Requires a `1` and the `0` guide.              |
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
    - Provide full functionality from an offline VM.
    - Run the daemon on the [`0_bitcoind.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md) VM.
    - Utilize Tor stream isolation.
- [`2_electrum.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/2_electrum.md)
  - Install the [Electrum](https://electrum.org) wallet configured to:
    - Connect to either the [`1_electrum-personal-server.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrum-personal-server.md) or the [`1_electrumx.md`](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/1_electrumx.md) VM.
    - Provide full functionality from an offline VM.

## Guides To Come
- [BTCPay Server](https://github.com/btcpayserver/btcpayserver)
- [c-Lightning](https://github.com/ElementsProject/lightning)
- [Electrs](https://github.com/romanz/electrs)
- [LND](https://github.com/LightningNetwork/lnd)
- [Wasabi Wallet](https://github.com/zkSNACKs/WalletWasabi)

## Git Mirrors
- https://github.com/qubenix/qubes-whonix-bitcoin
- http://qubenixibxoyldm3l3a5fobreaydmvdweqqojllutyyi4vgtbmugvhad.onion/qubenix/qubes-whonix-bitcoin
