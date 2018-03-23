[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)

# Chapter 3. Issuing Assets

In the last two chapters, we created accounts and made payments in Stellar's native currency, the lumen. We also learned that the native currency is typically used as fees to fund the Stellar platform.

In this chapter, we'll learn about **Credit Assets**, which are primitives that can hold and track any kind of real-world asset, such as fiat currencies, land, commodities, other cryptocurrencies, etc. A credit asset doesn't really hold a real-world asset -- it's more like an IOU that is used to *redeem* a real-world asset -- like how I issued five units of `AXE` for my five guitars in Chapter 1.

Entities that issue assets are called **Anchors**, and Anchors can be institutions, enterprises, or even individuals. Every Anchor has one or more associated **issuing accounts** from which they distribute assets.

## Creating a new asset

Every asset (except the native asset) has an **issuer** and a **code**. The issuer is the Stellar address of the account that initially issues the asset on the network, and the code is a short string representing the asset. For example, The *Bank of Canada* could create and issue Canadian Dollars on the Stellar network, with the code `CAD`.

Let's try that. First create a new account for the Bank of Canada on the test network, and define an alias for our new asset.

```sh
$ lumen set config:network test
$ lumen account new BankOfCanada
$ lumen friendbot BankOfCanada

# Create an asset alias called CAD issued by BankOfCanada. Lumen uses the alias as the
# as the asset code unless you explicitly specify it with the --code flag.
$ lumen asset set CAD BankOfCanada
```

## Trustlines

Stellar doesn't let an anchor simply issue a new asset to anyone, nor does it allow accounts to hold arbitrary assets. Accounts must explicitly create **trustlines** to assets they want to hold. [Trustlines](https://www.stellar.org/developers/guides/concepts/assets.html#trustlines) protect users from trading the wrong type of asset, by explicitly authorizing assets from only trusted anchors and issuers.

So, before Bob can hold any `CAD`, he must specifically create a trustline to the `CAD` he wants to hold with the `lumen trust` command. Since assets are defined by both their code *and* their issuer, there's no way for Bob to end up with the wrong `CAD` in his account.

```sh
# Create a trustline from Bob to the CAD issued by the Bank of Canada
$ lumen trust create bob CAD

# You can also refer to the asset without an asset alias.
$ lumen trust create bob CAD:BankOfCanada
```

## Issuing the asset

Now, the Bank of Canada can issue some CAD assets and send it to Bob. To issue an asset, simply make a payment from the issuer to the recipient.

```sh
$ lumen pay 10 CAD --from BankOfCanada --to bob
$ lumen balance bob CAD
# 10.0000000
```

Great! Bob now has some spending money.

So lets suppose that there's an Anchor run by a rogue entity called the *People's Bank of Canadian Separatists*, and they're issuing `CAD` too.

```sh
$ lumen account new rogue-bank
$ lumen friendbot rogue-bank
$ lumen asset set CAD-rogue rogue-bank --code CAD
```

We now have two assets with the code `CAD` on the network (with the alises `CAD` and `CAD-rogue`. How does poor old Bob ensure that the only `CAD` he holds is from the Bank of Canada? Simple -- he does nothing.

Let's see what happens when the People's Bank of Canadian Separatists try to issue their CAD to Bob.

```sh
$ lumen pay 10 CAD-rogue --from rogue-bank --to bob
ERRO[0008] payment failed: 400: Transaction Failed (&{tx_failed [op_no_trust]})  cmd=pay
$ lumen balance bob CAD-rogue
# 0
```

Guess that didn't work. The only `CAD` that Bob can hold is from the Bank of Canada. This model of explicitly trusting asset issuers is very different from, say Ethereum's ERC20 model, where issuers can distribute tokens to anyone on the network without their permission.

If Bob decides that he never wants to hold `CAD` again, he can return the assets back to the issuer and revoke his trustline.

```sh
$ lumen pay 10 CAD --from bob --to BankOfCanada
$ lumen trust remove bob CAD
```

### Trustline fees

Adding a trustline to an account raises the required minimum balance by the *base reserve* fee, which is 0.5 XLM. So Bob's account with a trustline to the Bank of Canada's CAD asset, needs to maintain at least 1.5 XLM to be able to transact on the network. Any transaction that reduces the balance to below the minimum will be rejected with `INSUFFICIENT_BALANCE`

You can read more about Stellar fees in the [developer guide](https://www.stellar.org/developers/guides/concepts/fees.html).

## Distributing assets

Issuing accounts don't hold balances for the assets they issue -- they can techincally issue an infinite supply. So sending an asset back to an issuer is equivalent to destroying the assets.

To create a fixed supply, the typical strategy is to create a new *distributor* account, also managed by the anchor, issue it a fixed number of assets, and then permanently disable the issuer's account. The public can always look up the total supply by checking the issuer's ledger, and also confirm that the supply is fixed by verifying that the issuer's account is permanently disabled.

To permanently disable an account, use `lumen signers masterweight`. This only works if there are no other signers on the account, and we'll discuss this in the next chapter.

```sh
# Create a distributor account for BoC CAD
$ lumen account new distributor
$ lumen friendbot distributor

# Add a trustline for the distributor and distribute a fixed supply of CAD
$ lumen trust create distributor CAD
$ lumen pay 10000000000 CAD --from BankOfCanada --to distributor

# Now kill BoC's issuer's account
$ lumen signers masterweight BankOfCanada 0
```

Bank of Canada can no longer create new CAD assets from that issuer. Their only option now is to create a new issuer account, and get customers to create a new trustline to the new asset for that account.

### Setting asset limits

As an asset holder, one can limit the total amount of an asset that they hold during trustline creation. This is typically a preventitive measure against malice or errors.

```sh
# Don't hold more than 5000 CAD
$ lumen trust create bob CAD 5000

# This will fail
$ lumen pay 7000 CAD-BoC --from BankOfCanada --to bob
```

As an anchor, you can also choose to pre-approve your asset holders. By marking the issuing account as `AUTH_REQUIRED`, you can require trustlines to be approved by the issuer before being created.

```sh
$ lumen flags BankOfCanada auth_required

# BankOfCanada must sign this transaction
$ lumen trust create bob CAD-BoC 1000 --signers bob,BankOfCanada
```

## Managing asset aliases

In the above examples, we defined aliases for the new assets that we created. Remember that these aliases are a construct of the Lumen tool, and not recognized by the Stellar network.

To look up assets by their alias, you can use the `lumen asset` command.

```sh
# Who issues CAD-BoC?
$ lumen asset issuer CAD
# GC6C225I4VIKCLUWJNAFRTTUN5UAMK7JRTCRUN3KSVXULVZ6OEH2WQRH

$ lumen asset code CAD
# CAD

$ lumen asset code CAD-rogue
# CAD

# The alias `native` is reserved by Lumen and designates the
# network's native asset.
$ lumen balance mo native
```

Delete an asset alias with `lumen del`.

```sh
$ lumen asset del CAD-BoC
```

If you don't want to create an asset, you can use colon-syntax to reference one without making an alias. The format `CODE:ISSUER` or `CODE:ISSUER:TYPE` can be used in place of any of the other commands in this chapter.

```sh
$ lumen trust create bob CAD:BankOfCanada

# Incase you want to specify the asset type
$ lumen pay 1 USD:BankOfCanada:credit_alphanum4 --from mary --to bob

# No aliases at all
$ lumen balance bob CAD:GC6C225I4VIKCLUWJNAFRTTUN5UAMK7JRTCRUN3KSVXULVZ6OEH2WQRH
```

You can also use federated addresses (see [Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md)) when specifying your assets.

```sh
$ lumen pay 1 CAD:issuer*bankofcanada.com --from mary --to bob

$ lumen asset set CAD anchor*citibank.com
```

## Onward

To learn more about assets, read the section in the [Stellar developer guide](https://www.stellar.org/developers/guides/concepts/assets.html). You can explore assets in the [Stellar lab](https://www.stellar.org/laboratory/#explorer?resource=assets&endpoint=single&network=test).

For now, lets move on to [Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md), where we discuss multisignature accounts and transactions.

[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)
