[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md)


# Chapter 2. Payments

Payments are the *raison d'être* of Stellar, and account holders can pay each other using a variety of currencies and assets. In this chapter, we'll discuss and experiment with payments using Stellar's native asset, the lumen.

### Detour: Working with the Lumen tool

The Lumen command-line-interface (CLI) is broken up into commands, subcommands, and flags. You can always find out more about a command's usage and parameters with the `--help` flag.

```sh
$ lumen --help
Lumen is a commandline client for the Stellar blockchain

Usage:
  lumen [flags]
  lumen [command]

Available Commands:
  account     manage stellar keypairs and accounts
  asset       manage stellar assets
  balance     check the balance of [asset] on [account]
  del         delete variable
  dex         trade assets on the DEX
  friendbot   fund [address] on the test network with friendbot
  get         get variable
  help        Help about any command
  ns          set namespace to [namespace]
  pay         send [amount] of [asset] from [source] to [target]
  set         set variable
  signer      manage signers on account
  trust       manage trustlines between accounts and assets
  version     get version of lumen CLI
  watch       watch the account on the ledger

Flags:
  -h, --help             help for lumen
      --network string   network to use (test) (default "test")
      --ns string        namespace to use (default) (default "default")
      --store string     namespace to use (default) (default "file:/Users/mo/.lumen-data.yml")
  -v, --verbose          verbose output (false)

Use "lumen [command] --help" for more information about a command.
```

To make payments, we use `lumen pay`. You can get command-specific help for pay with `lumen pay --help`.

```sh
 $ lumen pay --help
send [amount] of [asset] from [source] to [target]

Usage:
  lumen pay [amount] [asset] --from [source] --to [target] [flags]

Flags:
      --from string           source account seed or name
      --fund                  fund a new account
  -h, --help                  help for pay
      --max string            spend no more than this much during path payments
      --memoid string         memo ID
      --memotext string       memo text
      --path stringSlice      comma-separated list of paths, uses auto pathfinder if empty
      --signers stringSlice   alternate signers (comma separated)
      --to string             target account address or name
      --with string           make a path payment with this asset

Global Flags:
      --network string   network to use (test) (default "test")
      --ns string        namespace to use (default) (default "default")
      --store string     namespace to use (default) (default "file:/Users/mo/.lumen-data.yml")
  -v, --verbose          verbose output (false)
```

## Your first payment

Before we proceed, lets make sure we're using the test network. (If you want to use the public network, type `public` instead of `test` below.)

```sh
$ lumen set config:network test
```

Kelly pays Bob 5 XLM for the cookie she just bought, and leaves a thank you message with the `--memotext` flag.

```sh
$ lumen pay 5 --from kelly --to bob --memotext 'thanks for the cookie'
$ lumen balance kelly
$ lumen balance bob
```

Done! Kelly just sent Bob 5 XLM.

Every transaction can have a short memo associated (upto 28 bytes long.) The memo can used by a payment processor or business to redirect funds. For example, *Mary's Bank* can have a single stellar account for all its customers, and require payers to fill in the memo to designate the recipient. Instead of a text memo, the bank could require a numeric one, for which we use the `--memoid` flag.

```sh
$ lumen account set MarysBank GAFUU44WASFPD4YHIU5TKVMGLFMOHAQIOUJTTFQ432W65PFYQVVXFSUW
$ lumen pay 5 --from kelly --to MarysBank --memoid 485532245
```

Notice that we created an account alias for Mary's Bank called `MarysBank`. Aliases make it easier to work with Stellar, since you don't have to keep pasting in long addresses in your commands. Lumen keeps track of these aliases in a state file in your home directory. Lumen also recognizes if an alias is an address or seed and uses them contextually depending on the command.

For example, in the above `pay` command, Lumen used Kelly's seed instead of her address (because the seed is what actually *signs* and authorizes the transaction.) If we didn't have Kelly's seed, the transaction would fail. Lets see what happens if you try to make a payment from Mary's Bank.

```sh
$ lumen pay 10 --from MarysBank --to kelly --memotext 'i am h4xor'
# Output: error
```

Obviously, we don't have their private seed -- and hopefully we never will.

## Exploring Transactions

So, how do you know it worked? A great way to debug transactions on the Stellar platform is to use the [Stellar Laboratory](https://www.stellar.org/laboratory).

You can use the [endpoint explorer](https://www.stellar.org/laboratory/#explorer?network=test) in the lab to look up accounts, ledger balances, and go through their transaction history.

For example, here's the transaction history of an account on the test network: [GDELI4BPSO7SZGNNIDJ33N2HMJDQKB6PDD6P633U6LKGM26BYDVPRXU3](https://www.stellar.org/laboratory/#explorer?resource=accounts&endpoint=single&values=eyJhY2NvdW50X2lkIjoiR0RFTEk0QlBTTzdTWkdOTklESjMzTjJITUpEUUtCNlBERDZQNjMzVTZMS0dNMjZCWURWUFJYVTMifQ%3D%3D&network=test) (scroll down to see the results.)

## Managing account aliases

Aliases make it simpler to work with Stellar. You can add and remove aliases with `lumen account set` and `lumen account del`. You can also generate new key pairs and alias them with `lumen account new`.

```sh
$ lumen account del MarysBank
$ lumen account address MarysBank
# ERRO[0000] could not get address for account: MarysBank    cmd=account subcmd=address

$ lumen account new bill
# GAT5JWOXDCHR423M7VAPMC52NH6KQS4MTATXRJT7GTXDSERKQFFQNLC5 SCKNY3E6WSYFZASR3S34NGQLZLC6WJYJN3OX4AMOPPKQQP66YF2P5NQR

$ lumen account seed bill
# SCKNY3E6WSYFZASR3S34NGQLZLC6WJYJN3OX4AMOPPKQQP66YF2P5NQR
```

## Using Namespaces

Sometimes, you may want to start from a clean slate, but still keep your existing aliases. You can use Lumen's *namespaces* feature to do this. Use `lumen ns mynamespace` to create a new namespace (or switch to an existing one.)

```sh
$ lumen ns project1
$ lumen account new bill
# GCVO44W7CY4NA4WMAFW2ZAIFTQBFYNCW2QVRMLGOUFZQ67233D332UCO SDSNH72UMGBYA6ABJHVMRQLJGTTDS7EIAHKGA44RXU6JB6SDLQECW6ZI
$ lumen account address bill
# GCVO44W7CY4NA4WMAFW2ZAIFTQBFYNCW2QVRMLGOUFZQ67233D332UCO

# Now lets switch to a new namespace
$ lumen ns project2
$ lumen account address bill
# RRO[0000] could not get address for account: bill    cmd=account subcmd=address

# Switch back
$ lumen ns project1
$ lumen account address bill
# GCVO44W7CY4NA4WMAFW2ZAIFTQBFYNCW2QVRMLGOUFZQ67233D332UCO

# Get the current namespace
$ lumen ns
# project1
```

Namespaces are great to switch between the public and test networks and keep your addresses segregated.

```sh
# Create a namespace called prod and associate it with the public Stellar network
$ lumen ns prod
$ lumen set config:network public

# Create a namespace called test and associate it with the testnet
$ lumen ns test
$ lumen set config:network test

# Switch to the prod namespace
$ lumen ns prod

# We're now transacting on the public network. Any new aliases are tied to
# this network.
$ lumen account new kelly
$ lumen pay --from mo --to kelly --fund

# Switch back to the test namespace
$ lumen ns test

# We're now transacting on the test network. Lets create a new alias for Kelly
# here.
$ lumen account new kelly
```

## Concepts

Transactions in Stellar can consist of one or more [operations](https://www.stellar.org/developers/guides/concepts/operations.html). Each of these operations modify the ledger in some way, and is charged a **base fee** of 100 stroops. So, a transaction with 9 operations would pay 900 stroops.

For example, when you make a payment with Lumen using the `pay` command, it creates a [Payment Operation](https://www.stellar.org/developers/guides/concepts/list-of-operations.html#payment), and bundles it into a new transaction. It then signs it with your private seed, and submits it to the network.

The transaction envelope also contains the total fee to be paid, the address of the account that pays the fee, memo information, a list of signers, etc. For example, Mary's Bank could initiate a payment between their customers, Bob and Kelly, and foot the transaction fee. (This requires multisignature accounts, which we'll learn about in Chapter 4.)

To learn more about transactions read [the topic on the Stellar devlopers site](https://www.stellar.org/developers/guides/concepts/transactions.html).

## Onward

Now that we now how to work with aliases and make XLM payments, lets get to the fun stuff: [issuing assets](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md).

[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md)
