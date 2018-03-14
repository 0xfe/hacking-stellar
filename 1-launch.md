# Chapter 1.

## What is Stellar?

Stellar is a decentralized payment network. In a nutshell, it lets multiple parties send *unforgeable*
digital tokens to each other. These tokens could represent any kind of value, like Dollars and Euros,
or even kittens and hugs.

Anyone can create and distribute tokens on the Stellar platform. It's dead simple.

For example, *I* could create five tokens for a new currency called `AXE`, one for each of my guitars,
with the note: *“the bearer of this token is entitled to one of Mo’s guitars.”* I could then sell 
the tokens to third parties, who could in turn resell them.

Because I’m generally regarded as a trustworthy individual (I wear a monocle), the current token owners
would be comfortable holding these tokens -- they know they could redeem them for real guitars any time.


## On Trust

The magic of Stellar is that it’s *trustless*. Because it’s built on a decentralized ledger (commonly
also known as a blockchain), there isn’t a single entity that has overreaching control over it. The
platform is run by individuals and organizations all over the world, each contributing some compute,
storage, and network capacity.

No single person owns it, and this makes Stellar very difficult for governments, organizations, or
rogue entities to compromise.

Obviously, you still need to trust the issuers of the tokens, just like you’d have to trust me to
redeem the guitars. Or just like you trust a government that issues real currencies.

With decentralized ledgers however, you have fewer things to trust. For example, when a currency is
issued on the Stellar network, you can verify that the supply is consistent with what the issuer claims,
or you can move funds quickly and reliably without going through a trusted third-party.

***Thought
exercise:*** *Think about how you would buy, say, an expensive guitar, today from a distant seller without
a trusted third-party.*

As always, the devil’s in the details, so let’s start exploring by getting our hands dirty.

## Hacking Stellar

To start experimenting, let’s use [Lumen](http://github.com/0xfe/lumen), which is a commandline client for
the Stellar platform. Lumen is a really handy tool for working with Stellar, and especially useful while
debugging complex Stellar applications.

You can download the latest release of Lumen for your operating system [here](https://github.com/0xfe/lumen/releases).

### Setup

After downloading Lumen, move it to your search path, and configure it to use the test network. We're not
playing with real money for now.

```sh
sudo mv lumen.macos /usr/local/bin/lumen
lumen set config:network test
```

### Create an account
A user must have an account on the network. To create an account, you first need to generate a key pair -- these
are the keys to your vault.

A key pair consists of two 28-byte strings. One of them is your public “address”, the other your private “seed”.
Think of one as your username, and the other as your password. Never share your seed with anyone.

Let’s create a new key pair for Bob with Lumen.

```sh
$ lumen account new bob
# Output:
# GDTNQMX5RJ34CWU2XGPJOYHKWSATVALZZ3VCVQOQUXDO7CMHCLH6HDST SAEEOOLWEQDA6H3S5US3IFQ6TE377AR7OS7WUEKJNKVU437FM4KHKJNQ
```

You’ll see that Lumen spit out two strings. The one starting with `G` is Bob’s address, the one starting with `S`
is his seed. Lumen stored this key pair in your home directory (`$HOME/.lumen_data.json`) and associated it with
the name `bob`.

You can always lookup Bob’s address or seed later.

```
$ lumen account address bob
# GDTNQMX5RJ34CWU2XGPJOYHKWSATVALZZ3VCVQOQUXDO7CMHCLH6HDST
$ lumen account seed bob
# SAEEOOLWEQDA6H3S5US3IFQ6TE377AR7OS7WUEKJNKVU437FM4KHKJNQ

```

Great, so now that you have a key pair, let’s turn it into an account. To open an account you need a minimum balance,
known as the Base Reserve. Since we’re using the test network and all of this is funny money anyway, you can use
Friendbot to give you some funds to open your account.

Let’s do that, and then check our balance

```sh
$ lumen friendbot bob
$ lumen balance bob
# Output: 10000.0000000
```

Okay, you now have a funded account, and you can transact on the Stellar test network. This is no fun to do alone, so
lets create another account for Kelly. This time, we’ll have Bob fund her account instead of Friendbot.

```sh
$ lumen account new kelly

# Fund her with 1000 lumens
$ lumen pay 1000 --from bob --to kelly --fund
$ lumen balance kelly
# Output: 1000.0000000
```
