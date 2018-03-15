[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md)

# Chapter 3. Issuing Assets

In the last two chapters, we created accounts and made payments in Stellar's native currency, the lumen. We also learned that the native currency is typically used as fees to fund the Stellar platform.

In this chapter, we'll learn about **Credit Assets**, which are primitives that can hold and track any kind of real-world asset, such as fiat currencies, land, commodities, other cryptocurrencies, etc. A credit asset doesn't really hold a real-world asset -- it's more like an IOU that is used to *redeem* a real-world asset -- like how I issued five units of `AXE` for my five guitars in Chapter 1.

Entities that issue assets are called **Anchors**, and Anchors can be institutions, enterprises, or even individuals. Every Anchor has one or more associated **issuing accounts** from which they distribute assets.

## Creating a new Asset

Every asset has an **issuer** and a **code**. The issuer is the Stellar address of the account that initially issues the asset on the network, and the code is a short string representing the asset. For example, The *Bank of Canada* could create and issue Canadian Dollars on the Stellar network, with the code `CAD`.

Let's try that. First create a new account for the Bank of Canada on the test network, and define an alias for our new asset.

```sh
$ lumen set config:network test
$ lumen account new BankOfCanada
$ lumen friendbot BankOfCanada

# Create an asset called CAD issued by BankOfCanada (with the alias CAD-BoC)
$ lumen asset set CAD-BoC BankOfCanada --code CAD
```

Note that we've only just defined an asset with the code `CAD` -- we haven't issued any yet. Stellar has no knowledge of this asset. We'll issue some `CAD` soon, but first let's suppose that the *People's Bank of Canadian Separatists* want to issue `CAD` too.

```sh
$ lumen account new PBCS
$ lumen friendbot PBCS
$ lumen asset set CAD-PBCS PBCS --code CAD
```

We now have two assets with the code `CAD` on the network. How do we know which is which? And how does poor old Bob ensure that the only `CAD` he holds is from the Bank of Canada?

This is where [trustlines](https://www.stellar.org/developers/guides/concepts/assets.html#trustlines) come into the picture. For a credit asset to be held by an account, the account must create a *trustline* to the asset. And since assets are defined by both their code *and* their issuer, there is no way for Bob to mistakenly (or even maliciously) end up holding the wrong `CAD`.

So, before Bob can hold any `CAD`, he must specify create a trustline to the `CAD` he wants to hold.

```sh
# Create a trustline from Bob to the CAD issued by the Bank of Canada
$ lumen trust create bob CAD-BoC
```

Now, the Bank of Canada can issue some CAD assets and send it to Bob.

```sh
$ lumen pay 10 CAD-BoC --from BankOfCanada --to bob
$ lumen balance bob CAD-BoC
# 10.0000000
```

What if PBCS tried to issue their CAD to Bob?

```sh
$ lumen pay 10 CAD-PBCS --from PBCS --to bob
ERRO[0008] payment failed: 400: Transaction Failed (&{tx_failed [op_no_trust]})  cmd=pay
$ lumen balance bob CAD-PBCS
0
```

Guess that didn't work. The only `CAD` that Bob can hold is from the Bank of Canada. This model of explicitly trusting asset issuers is very different from, say Ethereum's ERC20 model, where issuers can distribute tokens to anyone on the network without their permission.

If Bob decides that he never wants to hold `CAD` again, he can return the assets back to the issuer and revoke his trustline.

```sh
$ lumen pay 10 CAD-BoC --from bob --to BankOfCanada
$ lumen trust remove bob CAD-BoC
```

## Concepts

Issuing accounts don't hold balances for the assets they issue -- they can techincally issue an infinite supply. So sending an asset back to an issuer is equivalent to destroying the assets.

To create a fixed supply, the typical strategy is to create a new *distributor* account, also managed by the anchor, issue it a fixed number of assets, and then permanently disable the issuer's account. The public can always look up the total supply by checking the issuer's ledger, and also confirm that the supply is fixed by verifying that the issuer's account is permanently disabled.

As an asset holder, one can limit the total amount of an asset that they hold. This is typically a preventitive measure against malice or errors.

```sh
# Don't hold more than 5000 CAD
$ lumen trust create bob CAD-BoC 5000

# This will fail
$ lumen pay 7000 CAD-BoC --from BankOfCanada --to bob
```

## Managing asset aliases

In the above examples, we defined aliases for the new assets that we created. Remember that these aliases are a construct of the Lumen tool, and not recognized by the Stellar network.

To look up assets by their alias, you can use the `lumen asset` command.

```sh
# Who issues CAD-BoC?
$ lumen asset issuer CAD-BoC
# GC6C225I4VIKCLUWJNAFRTTUN5UAMK7JRTCRUN3KSVXULVZ6OEH2WQRH

$ lumen asset code CAD-BoC
# CAD

# Remove the alias for an asset
$ lumen asset del CAD-BoC
```

## Onward

To learn more about assets, read the section in the [Stellar developer guide](https://www.stellar.org/developers/guides/concepts/assets.html). For now, lets move on to Chapter 4, where we discuss multisignature accounts and transactions.


https://www.stellar.org/laboratory/#explorer?resource=assets&endpoint=single&network=test

[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md)
