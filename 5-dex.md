[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)

# Chapter 5. The Decentralized Exchange

One of the coolest features of Stellar is the decentralized distributed exchange, where users can buy and sell any of the assets on the network. Stellar actually steps this up a notch, allowing you to do instant cross-asset payments, so you can always transact with your preferred currency.

## Trading on the exchange

To buy and sell assets on the Stellar exchange, use the `lumen dex trade` command. It allows you to make an offer to sell a certain amount of one asset for a price in terms of another asset.

Let's have Bob make an offer to sell 10 lumens (XLM) for 5 USD each. For the arithmetically challenged, it's an offer to buy 50 USD for 10 lumens.

```sh
# Recall that `native` is an internal alias for the native asset (lumens/XLM)
$ lumen dex trade bob --buy USD --sell native --amount 10 --price 5
```

Once this offer is made, it remains in Stellar's order book until either someone else takes it, or Bob revokes it.

To see all of Bob's offers, use `lumen dex list`. You'll notice that every offer is associated with an Offer ID (in parenthesis.)

```sh
$ lumen dex list bob
# (141452) selling 10.0000000 lumens for USD at 5.0000000 USD/lumen.
```

You can get more detail about Bob's offers with the `--format json` flag.

```json
$ lumen dex list bob --format json
{
  "_links": {
    "self": {
      "href": "https://horizon-testnet.stellar.org/offers/143297"
    },
    "offer_maker": {
      "href": "https://horizon-testnet.stellar.org/accounts/GDELI4BPSO7SZGNNIDJ33N2HMJDQKB6PDD6P633U6LKGM26BYDVPRXU3"
    }
  },
  "id": 141452,
  "paging_token": "141452",
  "seller": "GDELI4BPSO7SZGNNIDJ33N2HMJDQKB6PDD6P633U6LKGM26BYDVPRXU3",
  "selling": {
    "asset_type": "native",
    "asset_code": "XLM",
  },
  "buying": {
    "asset_type": "credit_alphanum4",
    "asset_code": "USD",
    "asset_issuer": "GC6C225I4VIKCLUWJNAFRTTUN5UAMK7JRTCRUN3KSVXULVZ6OEH2WQRH"
  },
  "amount": "10.0000000",
  "price_r": {
    "n": 5,
    "d": 1
  },
  "price": "5.0000000"
}
```

Mary can take Bob's offer by making an inverse offer on the DEX. Again, that's an offer to buy 50 units of XLM for 20¢ each.

```sh
$ lumen dex trade mary --buy native --sell USD --amount 50 --price 0.2
```

## Managing offers

If nobody takes Bob's offer, he can choose to reduce the price of the existing offer with the `--update` flag, passing in the offer ID.

```sh
$ lumen dex trade bob --buy USD --sell native --price 4 --update 141452
```

Alternatively, Bob can simply revoke his offer with the `--delete` flag.

```sh
$ lumen dex trade bob --buy USD --sell native --price 4 --delete 141452
```

## Cross-asset Payments

Stellar lets you make transparent cross currency payments over the DEX. For example, if Mary only prefers to transact in Canadian Dollars, and Kelly only likes to transact in Euros, Stellar can facilitate an automatic currency conversion using the cheapest prices on the DEX.

Stellar can sometimes do this even if there isn't a direct CAD <-> EUR offer on the DEX, by finding alternate payment paths over multiple currencies. So, if there are offers for CAD <-> USD, and EUR <-> USD, Stellar can trade the CAD for USD, and then the USD for EUR, all in a single atomic step.

Cross-asset payments are simple to execute with Lumen. Use `--with [asset]` to specify the currency you want to make the payment with, and `--max [price]` to specify the maximum amount you want to spend for the payment.

```sh
# Kelly deposits 10 USD into Mary's account, and pays for it with her EUR balance. She
# also instructs Stellar not to spend more than 8 EUR on the transaction.
$ lumen pay 10 CAD --from kelly --to mary --with EUR --max 8
```

Lumen uses automatic path finding to facilitate this payment, but you can specify your own asset path if you'd like.

```sh
$ lumen pay 10 USD --from kelly --to mary --with EUR --max 8 --through native
```

## Trying it out

Let's try an end-to-end example where we create a bunch of assets, offer them on the DEX, and then make a path payment through them.

First, lets create some accounts.

```sh
# Switch to the test network
$ lumen set config:network test

# Create and fund our asset issuer.
$ lumen account new issuer
$ lumen friendbot issuer

# Create and fund Bob and Kelly, our end-users.
$ lumen account new kelly
$ lumen friendbot issuer
$ lumen account new bob
$ lumen friendbot issuer

# Create and fund Mary and Mike, active traders.
$ lumen account new mary
$ lumen friendbot mary
$ lumen account new mike
$ lumen friendbot mike
```

Now, let's create some assets for currencies `USD`, `INR`, and `EUR` all issued by `issuer`. We'll also create trustlines for Bob, Kelly, Mary, and Mike, and seed their accounts with some funds.

```sh
$ lumen asset set USD issuer
$ lumen asset set INR issuer
$ lumen asset set EUR issuer

# Kelly's preferred currency is INR, she doesn't hold anything else.
$ lumen trust create kelly --to INR
$ lumen pay 1000 INR --from issuer --to kelly

# Bob's preferred currency is USD.
$ lumen trust create bob --to USD
$ lumen pay 1000 USD --from issuer --to bob

# Mary's likes to trade USD and EUR.
$ lumen trust create mary --to USD
$ lumen pay 1000 USD --from issuer --to mary
$ lumen trust create mary --to EUR
$ lumen pay 1000 EUR --from issuer --to mary

# Mike likes to trade EUR and INR.
$ lumen trust create mike --to EUR
$ lumen pay 1000 EUR --from issuer --to mike
$ lumen trust create mike --to INR
$ lumen pay 1000 INR --from issuer --to mike
```

Since Mary and Mike are active traders, have them put some offers up on the DEX. Let's keep the prices at `1` to keep things simple.

```sh
$ lumen dex trade mary --buy USD --sell EUR --amount 50 --price 1
$ lumen dex trade mike --buy EUR --sell INR --amount 50 --price 1
```

So Mary is trading USD for EUR, and Mike's trading EUR for INR. These offers hang out on the network until someone makes an inverse offer.

Some time later, Kelly fixes Bob's Internet, and charges him `10 INR`, which is her preferred currency. Bob only has `USD`, which is his preferred currency.

No big deal. Thanks to the DEX, Bob can pay Kelly `10 INR` with his `USD`, by making a path payment.

```sh
# Bob knows that the USD <-> INR exchange rate is 1, so he sets the max spend to 10, which implies that
# he spends no more than 10 USD on this transaction.
$ lumen pay 10 INR --from bob --to kelly --with USD --max 10
```

That's it! Let's check their balances to verify that the transaction went through.

```sh
$ lumen balance bob USD
# 9990.0000000

$ lumen balance kelly INR
# 1010.0000000
```

To see what happened, let's take a look at the orderbook.

```sh
$ lumen dex list mary
# (141452) selling 40.0000000 EUR for USD at 1.0000000 USD/EUR

$ lumen dex list mike
# (143297) selling 40.0000000 INR for EUR at 5.0000000 USD/XLM
```

The path payment filled 10 units of both Mary's and Mike's offers. Effectively, Bob sold his `USD` to Mary, who in turn sold her `EUR` to Mike, who ended up paying Kelly the `10 INR`. Furthermore, Stellar ensured that of this happend in one atomic step, i.e., there's no way that this payment flow would only be partially executed, leaving both Bob and Kelly hanging.

# Onward

As you can see, Stellar's DEX is a fantastic way to facilitate cross-asset payments, but also path payments help increase the amount of liquidity in the DEX. In the next chapter, we'll work on building an Anchor.

[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)