[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)

# Chapter 1. The Basics

## What is Stellar?

Stellar is a decentralized payment network. In a nutshell, it lets multiple parties send *unforgeable* digital tokens to each other. These tokens could represent any kind of value, like Dollars and Euros, or even kittens and hugs.

Anyone can create and distribute tokens on the Stellar platform. It's dead simple.

For example, I could create five tokens for a new currency called `AXE`, one for each of my guitars, with the note: *“the bearer of this token is entitled to one of Mo’s guitars.”* I could then sell the tokens to third parties, who could in turn resell them.

Since I’m generally regarded as a trustworthy individual, the current token owners would be comfortable holding these tokens -- they know they could redeem them for real guitars any time.


## On Trust

The magic of Stellar is that it’s *trustless*. Because it’s built on a decentralized ledger (commonly also known as a blockchain), there isn’t a single entity that has overreaching control over it. The platform is run by individuals and organizations all over the world, each contributing some compute, storage, and network capacity.

No single person owns it, and this makes Stellar very difficult for governments, organizations, or rogue entities to compromise.

Obviously, you still need to trust the issuers of the tokens, just like you’d have to trust me to redeem the guitars. Or just like you trust a government that issues real currencies.

With decentralized ledgers however, you have fewer things to trust. For example, when a currency is issued on the Stellar network, you can verify that the supply is consistent with what the issuer claims, or you can move funds quickly and reliably without going through a trusted third-party.

***Thought exercise:*** *Think about how you would buy, say, an expensive guitar, today from a distant seller without a trusted third-party.*

As always, the devil’s in the details, so let’s start exploring by getting our hands dirty.

## Hacking Stellar

To start experimenting, let’s use [Lumen](http://github.com/0xfe/lumen), which is a commandline client for the Stellar platform. Lumen is a really handy tool for working with Stellar, and especially useful while debugging complex Stellar applications.

You can download the latest release of Lumen for your operating system [here](https://github.com/0xfe/lumen/releases).

After downloading Lumen, move it to your search path, and configure it to use the test network. We're not playing with real money for now.

```sh
sudo mv lumen.macos /usr/local/bin/lumen
lumen set config:network test
```

### Create an account
A user must have an account to transact on the network. To create an account, you first need to generate a key pair -- these are the keys to your vault.

A key pair consists of two 28-byte strings. One of them is your public “*address*”, the other your private “*seed*”. Think of one as your username, and the other as your password. Your seed is a secret -- protect it.

Let’s create a new key pair for Bob with Lumen.

```sh
$ lumen account new bob
# Output:
# GDTNQMX5RJ34CWU2XGPJOYHKWSATVALZZ3VCVQOQUXDO7CMHCLH6HDST SAEEOOLWEQDA6H3S5US3IFQ6TE377AR7OS7WUEKJNKVU437FM4KHKJNQ
```

You’ll see that Lumen spit out two strings. The one starting with `G` is Bob’s address, the one starting with `S` is his seed. Lumen stored this key pair in your home directory (`$HOME/.lumen_data.json`) and associated it with the name `bob`.

You can always lookup Bob’s address or seed later.

```sh
$ lumen account address bob
# GDTNQMX5RJ34CWU2XGPJOYHKWSATVALZZ3VCVQOQUXDO7CMHCLH6HDST
$ lumen account seed bob
# SAEEOOLWEQDA6H3S5US3IFQ6TE377AR7OS7WUEKJNKVU437FM4KHKJNQ

```

Great, so now that you have a key pair, let’s turn it into an account. To open an account you need a minimum balance. Since we’re using the test network and all of this is funny money anyway, you can use [Friendbot](http://friendbot.stellar.org), which is a funding bot for test networks run by the Stellar organization.

You can ask Firendbot to fund a new account with the `lumen friendbot` command.

```sh
$ lumen friendbot bob
$ lumen balance bob
# Output: 10000.0000000
```

Okay, you now have a funded account, and you can transact on the Stellar test network. This is no fun to do alone, so lets create another account for Kelly. This time, we’ll have Bob fund her account instead of Friendbot.

```sh
$ lumen account new kelly

# Fund Kelly via Bob's account
$ lumen pay 1000 --from bob --to kelly --fund
$ lumen balance kelly
# Output: 1000.0000000
```

You can always look up detailed account information with `lumen info`. Don't worry about what all this means right now, we'll figure it out over the next few chapters.

```json
$ lumen info bob
{
  "address": "GADFYJK62RVQOU7BNP53TSUFEQ627H4OBZ6TLBVG4T2U25Y3GYR3MFF4",
  "balances": [],
  "signers": [
    {
      "public_key": "GDK5X4BUL3YUDEYNDVTJ4MX6NENGKSHPGHVCQNWX3B4MK633MKRWMJJO",
      "weight": 1,
      "key": "GDK5X4BUL3YUDEYNDVTJ4MX6NENGKSHPGHVCQNWX3B4MK633MKRWMJJO",
      "type": "ed25519_public_key"
    },
    {
      "public_key": "GADFYJK62RVQOU7BNP53TSUFEQ627H4OBZ6TLBVG4T2U25Y3GYR3MFF4",
      "weight": 1,
      "key": "GADFYJK62RVQOU7BNP53TSUFEQ627H4OBZ6TLBVG4T2U25Y3GYR3MFF4",
      "type": "ed25519_public_key"
    }
  ],
  "native_balance": {
    "asset": {
      "code": "XLM",
      "issuer": "",
      "type": "\"native\""
    },
    "amount": "9999.9999800",
    "limit": ""
  },
  "home_domain": "",
  "thresholds": {
    "high": 2,
    "medium": 2,
    "low": 2
  },
  "seq": "34126534528729090"
}
```

## Security PSA

Note that in the real world, you would not keep your seeds lying around in the clear. For personal use it's always better to use a hardware wallet, or store your seeds in a reputable password manager like [KeePassXC](https://keepassxc.org/).

For business and corporate users, you are strongly encouraged to use multisignature accounts -- this is discussed in detail in [Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md).

Finally, [Lumen](https://github.com/0xfe/lumen) is a tool, not a wallet. Don't rely on it to keep your seeds secure.

## Concepts

There's a lot going on in what we just did. What does it mean to "fund" an account? What did we fund it with? Why did we need to fund it in the first place?

The native currency of Stellar is the *lumen*, which has the currency code `XLM`. (Yep, this is also what the tool we're using is named after.) Lumens are used to pay for Stellar transactions, and can be bought and sold in many cryptocurrency exchanges. You check the market price of a lumen [here](https://coinmarketcap.com/currencies/stellar/).

For an account to be valid on the Stellar network, it needs to maintain a minimum balance of 1 XLM. This is based on Stellar's **base reserve** fee, which is 0.5 XLM. Funding an account is a special operation, distinct from typical payment operations -- this is why we used the `--fund` flag when we funded Kelly's account.

Each lumen is divisible into 10 million *stroops*. A stroop is the smallest divisible unit of a lumen, and is how balances are mainatained on the ledger.

There is also a fee to transact on the Stellar network called the *base fee*, which today is 100 stroops per operation. Read more about Stellar fees on the [Stellar developer site](https://www.stellar.org/developers/guides/concepts/fees.html).

## Onward

Move on to [Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) to learn about payments, aliases, and more Stellar concepts.

[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)