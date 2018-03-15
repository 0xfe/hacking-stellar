[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md)

# Chapter 4. Managing Signers (in-progress)

Transactions must be cryptographically signed before it can be submitted to Stellar for processing. And typically, it's the account that initiates the transaction (e.g., a payer, or a trustor) that signs it. Transactions that are not signed, or signed by the wrong parties are immediately rejected by the network.

Stellar lets you setup some sophisticated signature requirements. You can add and remove multiple signers to and from an account, and set their authorization levels based on the types of transactions you want to allow.

For example, Bob and Mary could share a joint Stellar account, out of which either one could spend independently. But they can set it up such that adding another person requires both Bob and Mary to approve the transaction.

## Adding a signer

The Lumen `signer` command lets you manage the signature requirements of an account.

```sh
$ lumen signer --help
manage signers on account

Usage:
  lumen signer [add|remove|thresholds|masterweight] [flags]
  lumen signer [command]

Available Commands:
  add          add signer_address as a signer on [account] with key weight [weight]
  masterweight set the weight of the accounts master key
  remove       remove signer_address as a signer from [account]
  thresholds   set low, medium, and high thresholds for [account]

Flags:
  -h, --help   help for signer

Global Flags:
      --network string   network to use (test) (default "test")
      --ns string        namespace to use (default) (default "default")
      --store string     namespace to use (default) (default "file:/Users/mo/.lumen-data.yml")
  -v, --verbose          verbose output (false)

Use "lumen signer [command] --help" for more information about a command.
```

Let's say that Mary, Bob, and Kelly want to create a pizza fund, into which they regularly contribute some money for pizza night. They elect Kelly to create the fund for them.

```sh
# Kelly first creates a new account for the pizza fund and seeds it with 5 XLM
$ lumen account new pizzafund
$ lumen pay 5 --from kelly --to pizzafund --fund

# The pizza company accepts USD from various common anchors. Let the fund
# hold no more than 200 USD.
$ lumen asset set USD citibank
$ lumen trust create pizzafund USD 200

# Kelly then adds Mary and Bob as signers on the fund. She only needs their public addresses for
# this, and creates some aliases for convenience.
$ lumen account set bob GCJTY3FCRCBPAPKG4NPWRLVLIL6EU4UHPBYPH75K4SNK4ELSTLS6JFDX
$ lumen account set mary GAFAP7HEY3Q7YQNNXKRIMVSTDGOP7PHMRPN5UZ4C655HARYV7KHSXEAQ

# Add bob, mary, and kelly as signers, with the weight of 1 each.
$ lumen signer add bob 1 --to pizzafund --memotext "pizza for bob"
$ lumen signer add mary 1 --to pizzafund --memotext "pizza for mary"
$ lumen signer add kelly 1 --to pizzafund --memotext "pizza for mary"
```

That's it! Now Bob, Kelly, and Mary can spend from the pizza fund and never be hungry again! Since Kelly added herself as a signer, she can throw away the pizza fund's seed.

Bob and Mary don't need the seed either. They'd make a payment using thier own seeds.

```sh
$ lumen pay 10 USD --from pizzafund --to pizzahut --signers bob
```

## Weights and Thresholds

When we added signers to the pizza fund with the `lumen signer` command, we set their weights to `1`. To understand what that did, we first need to understand **thresholds**.

Every Stellar operation is lumpted into one of three security categories: *low*, *medium*, and *high*. Each of these categories has a signing threshold, which is a number between 0 and 255. For a transaction of a specific category to be approved by the network, it must have enough signatures with weights summing up to the category's threshold.

For example, payments fall into the *medium* category. Suppose we wanted to require that the pizza fund (above) can only be spent from if at least two members approve, we would need to set the account's medium threshold to 2.

You can use the `signer thresholds` command to set the *low*, *medium*, and *high* thresholds for an account.

```sh
# Set thresholds: low=1, medium=2, high=2
$ lumen signer thresholds pizzafund 1 2 2 --signers kelly
```

Now, payments from `pizzafund` must have at least two signers (each signer has a weight of 1, and we need a minimum total of 2.) Notice that we also set the *high* threshold to 2? We needed to do this because operations to manage signers and thresholds fall into the *high* category. If we didn't also set *high* to 2, any one of them could set the *medium* threshold back to 1, and make a payment.

If we wanted to require that all three of them approve any threshold or signer operations, we could set *high* to 3.

```sh
$ lumen signer thresholds pizzafund 1 2 3 --signers kelly,mary
```

## The Master Key

When you first create an account you generate an address and a seed. The address is the public-facing identifier for the account, and the seed is the **master key**. As we explored in this chapter, you can always add and remove new keys to and from an account, but the master key is forever.

The default weight of the master key is `1`, and the only way to disable a master key, is by setting its weight to `0`. You can do this with `lumen masterweight`.

```sh
$ lumen masterweight pizzafund 0 --signers kelly,mary,bob
```

If you set the master weight to 0, and there are no other signers left, the account is permanently disabled. There's no getting it back. This is, in fact, the recommended way to lock an anchor's issuing account for fixed supply assets.

## Onward

To learn more about multisignature support in Stellar, read the [developer guide](https://www.stellar.org/developers/guides/concepts/multi-sig.html).









