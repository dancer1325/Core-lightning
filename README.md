# Core Lightning (CLN): A specification compliant Lightning Network implementation in C

* Core Lightning
  * previously c-lightning
  * == implementation of the Lightning Network protocol /
    * lightweight
    * highly customizable
    * [standard compliant][std]

* [Getting Started](#getting-started)
    * [Installation](#installation)
    * [Starting lightningd](#starting-lightningd)
    * [Using the JSON-RPC Interface](#using-the-json-rpc-interface)
    * [Care And Feeding Of Your New Lightning Node](#care-and-feeding-of-your-new-lightning-node)
    * [Opening A Channel](#opening-a-channel)
	* [Sending and Receiving Payments](#sending-and-receiving-payments)
	* [Configuration File](#configuration-file)
* [Further Information](#further-information)
    * [FAQ](doc/FAQ.md)
    * [Pruning](#pruning)
    * [HD wallet encryption](#hd-wallet-encryption)
	* [Developers](#developers)
* [Documentation](https://docs.corelightning.org/docs)

## Project Status

[![Continuous Integration][actions-badge]][actions]
[![Pull Requests Welcome][prs-badge]][prs]
[![Documentation Status][docs-badge]][docs]
[![BoL2][bol2-badge]][bol2]
[![Telegram][telegram-badge]][telegram]
[![Discord][discord-badge]][discord]
[![Irc][IRC-badge]][IRC]

* uses
  * in production | Bitcoin mainnet 2018+
    * launch of the [Blockstream Store][blockstream-store-blog]
    * pretty stable
* recommendations
  * experiment | `testnet` (or `regtest`)
* ways to reach out
  * [Build-on-L2][bol2]
  * implementation-specific [mailing list][ml1]
  * Lightning Network-wide [mailing list][ml2]
  * [CLN Discord][discord]
  * [CLN Telegram][telegram]
  * IRC | [dev][irc1]/[gen][irc2] channel

## Getting Started

* requirements
  * OS
    * Linux
    * macOS
  * running `bitcoind` v25.0+ locally or remotely /
    * -- fully caught up with the -- network / you're running on
    * relays transactions (== `blocksonly=0`)
* Pruning
  * `prune=n` option | `bitcoin.conf`
  * partially supported
  * [more details](#pruning)

### Installation

* supported installation options
  * [pre-compiled binary | release page][releases] | GitHub
  * [provided docker images][dockerhub] | Docker Hub
  * [compile the source code](doc/getting-started/getting-started/installation.md)

### Starting `lightningd`

#### Regtest (local, fast-start) Option

* ```bash
  . contrib/startup_regtest.sh
  ```
  * goal
    * experiment with `lightningd`
  * == script / set up a `bitcoind` regtest test network / 2 local lightning nodes /
    * have `start_ln` helper
    * check notes | top of the `startup_regtest.sh`

#### Mainnet Option

* 👀== test with real bitcoin 👀
* steps
  * 
    ```bash
    # run a `bitcoind` node
    bitcoind -daemon
    ```

    * wait UNTIL `bitcoind` -- has synchronized with the -- network
    * recommendations
      * do NOT have `walletbroadcast=0` | your `~/.bitcoin/bitcoin.conf`
        * Reason: 🧠 you may run into trouble 🧠
  *
    ```bash
    # start `lightningd` 
    lightningd --network=bitcoin --log-level=debug
    ```
    * if NOT managed carefully -> see [below](#pruning)
    * creates a `.lightning/` | your home directory
    * see `man -l doc/lightningd.8` (or https://docs.corelightning.org/docs)
---

### Using The JSON-RPC Interface

* TODO:
Core Lightning exposes a [JSON-RPC 2.0][jsonrpcspec] interface over a Unix Domain socket; the `lightning-cli` tool can be used to access it, or there is a [python client library](contrib/pyln-client).

You can use `lightning-cli help` to print a table of RPC methods; `lightning-cli help <command>`
will offer specific information on that command.

Useful commands:

* [newaddr](doc/lightning-newaddr.7.md): get a bitcoin address to deposit funds into your lightning node.
* [listfunds](doc/lightning-listfunds.7.md): see where your funds are.
* [connect](doc/lightning-connect.7.md): connect to another lightning node.
* [fundchannel](doc/lightning-fundchannel.7.md): create a channel to another connected node.
* [invoice](doc/lightning-invoice.7.md): create an invoice to get paid by another node.
* [pay](doc/lightning-pay.7.md): pay someone else's invoice.
* [plugin](doc/lightning-plugin.7.md): commands to control extensions.

### Care And Feeding Of Your New Lightning Node

Once you've started for the first time, there's a script called
`contrib/bootstrap-node.sh` which will connect you to other nodes on
the lightning network.

There are also numerous plugins available for Core Lightning which add
capabilities: in particular there's a collection at: https://github.com/lightningd/plugins

Including [helpme][helpme-github] which guides you through setting up
your first channels and customizing your node.

For a less reckless experience, you can encrypt the HD wallet seed:
 see [HD wallet encryption](#hd-wallet-encryption).

You can also chat to other users at Discord [core-lightning][discord];
we are always happy to help you get started!


### Opening A Channel

First you need to transfer some funds to `lightningd` so that it can
open a channel:

```bash
# Returns an address <address>
lightning-cli newaddr
```

`lightningd` will register the funds once the transaction is confirmed.

Alternatively you can generate a taproot address should your source of funds support it:

```bash
# Return a taproot address
lightning-cli newaddr p2tr
```

Confirm `lightningd` got funds by:

```bash
# Returns an array of on-chain funds.
lightning-cli listfunds
```

Once `lightningd` has funds, we can connect to a node and open a channel.
Let's assume the **remote** node is accepting connections at `<ip>`
(and optional `<port>`, if not 9735) and has the node ID `<node_id>`:

```bash
lightning-cli connect <node_id> <ip> [<port>]
lightning-cli fundchannel <node_id> <amount_in_satoshis>
```

This opens a connection and, on top of that connection, then opens a channel.
The funding transaction needs 3 confirmation in order for the channel to be usable, and 6 to be announced for others to use.
You can check the status of the channel using `lightning-cli listpeers`, which after 3 confirmations (1 on testnet) should say that `state` is `CHANNELD_NORMAL`; after 6 confirmations you can use `lightning-cli listchannels` to verify that the `public` field is now `true`.

### Sending and Receiving Payments

Payments in Lightning are invoice based.
The recipient creates an invoice with the expected `<amount>` in
millisatoshi (or `"any"` for a donation), a unique `<label>` and a
`<description>` the payer will see:

```bash
lightning-cli invoice <amount> <label> <description>
```

This returns some internal details, and a standard invoice string called `bolt11` (named after the [BOLT #11 lightning spec][BOLT11]).

[BOLT11]: https://github.com/lightning/bolts/blob/master/11-payment-encoding.md

The sender can feed this `bolt11` string to the `decodepay` command to see what it is, and pay it simply using the `pay` command:

```bash
lightning-cli pay <bolt11>
```

Note that there are lower-level interfaces (and more options to these
interfaces) for more sophisticated use.

## Configuration File

`lightningd` can be configured either by passing options via the command line, or via a configuration file.
Command line options will always override the values in the configuration file.

To use a configuration file, create a file named `config` within your top-level lightning directory or network subdirectory
(eg. `~/.lightning/config` or `~/.lightning/bitcoin/config`).  See `man -l doc/lightningd-config.5`.

A sample configuration file is available at `contrib/config-example`.

## Further information

### Pruning

Core Lightning requires JSON-RPC access to a fully synchronized `bitcoind` in order to synchronize with the Bitcoin network.
Access to ZeroMQ is not required and `bitcoind` does not need to be run with `txindex` like other implementations.
The lightning daemon will poll `bitcoind` for new blocks that it hasn't processed yet, thus synchronizing itself with `bitcoind`.
If `bitcoind` prunes a block that Core Lightning has not processed yet, e.g., Core Lightning was not running for a prolonged period, then `bitcoind` will not be able to serve the missing blocks, hence Core Lightning will not be able to synchronize anymore and will be stuck.
In order to avoid this situation you should be monitoring the gap between Core Lightning's blockheight using `lightning-cli getinfo` and `bitcoind`'s blockheight using `bitcoin-cli getblockchaininfo`.
If the two blockheights drift apart it might be necessary to intervene.

### HD wallet encryption

You can encrypt the `hsm_secret` content (which is used to derive the HD wallet's master key) by passing the `--encrypted-hsm` startup argument, or by using the `hsmtool` (which you can find in the `tool/` directory at the root of this repo) with the `encrypt` method. You can unencrypt an encrypted `hsm_secret` using the `hsmtool` with the `decrypt` method.

If you encrypt your `hsm_secret`, you will have to pass the `--encrypted-hsm` startup option to `lightningd`. Once your `hsm_secret` is encrypted, you __will not__ be able to access your funds without your password, so please beware with your password management. Also, beware of not feeling too safe with an encrypted `hsm_secret`: unlike for `bitcoind` where the wallet encryption can restrict the usage of some RPC command, `lightningd` always needs to access keys from the wallet which is thus __not locked__ (yet), even with an encrypted BIP32 master seed.

### Developers

* [contribution guidelines](doc/contribute-to-core-lightning/coding-style-guidelines.md)

[blockstream-store-blog]: https://blockstream.com/2018/01/16/en-lightning-charge/
[std]: https://github.com/lightning/bolts
[prs-badge]: https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat
[prs]: http://makeapullrequest.com
[bol2-badge]: https://badgen.net/badge/BoL2/chat/blue
[bol2]: https://community.corelightning.org
[ml1]: https://lists.ozlabs.org/listinfo/c-lightning
[ml2]: https://lists.linuxfoundation.org/mailman/listinfo/lightning-dev
[discord-badge]: https://badgen.net/badge/Discord/chat/blue
[discord]: https://discord.gg/mE9s4rc5un
[telegram-badge]: https://badgen.net/badge/Telegram/chat/blue
[telegram]: https://t.me/lightningd
[IRC-badge]: https://img.shields.io/badge/IRC-chat-blue.svg
[IRC]: https://web.libera.chat/#c-lightning
[irc1]: https://web.libera.chat/#lightning-dev
[irc2]: https://web.libera.chat/#c-lightning
[docs-badge]: https://readthedocs.org/projects/lightning/badge/?version=docs
[docs]: https://docs.corelightning.org/docs
[releases]: https://github.com/ElementsProject/lightning/releases
[dockerhub]: https://hub.docker.com/r/elementsproject/lightningd/
[jsonrpcspec]: https://www.jsonrpc.org/specification
[helpme-github]: https://github.com/lightningd/plugins/tree/master/helpme
[actions-badge]: https://github.com/ElementsProject/lightning/workflows/Continuous%20Integration/badge.svg
[actions]: https://github.com/ElementsProject/lightning/actions
