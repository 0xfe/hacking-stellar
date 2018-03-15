[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md)

# Chapter 5. The Decentralized Exchange

One of the coolest features of Stellar is the decentralized distributed exchange, where users can buy and sell any of the assets on the network. Stellar actually steps this up a notch, allowing you to do instant cross-asset payments, so you can always transact with your preferred currency.

## Trading on the exchange

To buy and sell assets on the Stellar exchange, use the `lumen dex trade` command. It allows you to make an offer to sell a certain amount of one asset for a price in terms of another asset.

Let's have Bob make an offer to sell 10 XLM for 5 USD each. For the arithmetically challenged, it's an offer to buy 50 USD for 10 XLM.

```sh
$ lumen dex trade bob --buy USD --sell XLM --amount 10 --price 5
```

Once this offer is made, it remains in Stellar's order book until either someone else takes it, or Bob revokes it.

To see all of Bob's offers, use `lumen dex list`. You'll notice that every offer is associated with an Offer ID (in parenthesis.)

```sh
$ lumen dex list bob
# (141452) selling 10.0000000 XLM for USD at 5.0000000 USD/XLM
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
    "asset_type": "credit_alphanum4",
    "asset_code": "XLM",
    "asset_issuer": "GDELI4BPSO7SZGNNIDJ33N2HMJDQKB6PDD6P633U6LKGM26BYDVPRXU3"
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
$ lumen dex trade mary --buy XLM --sell USD --amount 50 --price 0.2
```

## Managing offers

If nobody takes Bob's offer, he can choose to reduce the price of the existing offer with the `--update` flag, passing in the offer ID.

```sh
$ lumen dex trade bob --buy USD --sell XLM --price 4 --update 141452
```

Alternatively, Bob can simply revoke his offer with the `--delete` flag.

```sh
$ lumen dex trade bob --buy USD --sell XLM --price 4 --delete 141452
```

## Cross-asset Payments


[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md)
