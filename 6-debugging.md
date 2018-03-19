[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)

# Chapter 6. Debugging Stellar

In this chapter, we'll explore the many ways to debug and troubleshoot your Stellar applications. To follow along with the examples in this chapter, you'll need the command-line tools, [jq](https://stedolan.github.io/jq/), [curl](https://curl.haxx.se/), and [Lumen](http://github.com/0xfe/lumen) (which you probably already have.)

## The Stellar Network

The Stellar organization manages two publically-accessible networks, called *live* and *test*. (These are sometimes also referred to as *public* and *testnet*.) Both consist of a peer-to-peer network of nodes running the [Stellar Core](https://www.stellar.org/developers/stellar-core/learn/admin.html) software, which validate and process transactions, and commit them to the global ledgers of their specific networks.

The live network is where real value is transacted. Lumens on the live network cost real money, and can be bought at many cryptocurrency exchanges. The test network on the other hand is

### Horizon

## Ledgers, transactions, and operations

Every few seconds, the Stellar network commits a ledger to its global database. The ledger consists of a set of transactions, each containing one or more operations.

Every transaction is charged a fee, derived from the number of operations in the transaction, and billed to its respective *source account*. The operations could be anything from payments, to offers, to administrative tasks such as managing signers. You can see the full list of Stellar operations [here](https://www.stellar.org/developers/guides/concepts/list-of-operations.html).

## Investigating a ledger

Let's start off with streaming the global database for ledger entries. You can use `lumen watch ledger` to do this. The `lumen watch` commands run forever (or until you explicitly terminate them), and are a really handy way to troubleshoot your applications.

```json
$ lumen watch ledger --network public
{
  "_links": {
    "self": {
      "href": "https://horizon.stellar.org/ledgers/16889095"
    },
    "transactions": {
      "href": "https://horizon.stellar.org/ledgers/16889095/transactions{?cursor,limit,order}",
      "templated": true
    },
    "operations": {
      "href": "https://horizon.stellar.org/ledgers/16889095/operations{?cursor,limit,order}",
      "templated": true
    },
    "payments": {
      "href": "https://horizon.stellar.org/ledgers/16889095/payments{?cursor,limit,order}",
      "templated": true
    },
    "effects": {
      "href": "https://horizon.stellar.org/ledgers/16889095/effects{?cursor,limit,order}",
      "templated": true
    }
  },
  "id": "6ecef75a9baabb85c4600ad2a12bd15a08b71e3410bb4b4780e25cec7a32ddf6",
  "paging_token": "72538110684037120",
  "hash": "6ecef75a9baabb85c4600ad2a12bd15a08b71e3410bb4b4780e25cec7a32ddf6",
  "prev_hash": "99beda19d3f8df030c12dfc057fb9f403b3d987ead561db866ea4450e55edc2b",
  "sequence": 16889095,
  "transaction_count": 3,
  "operation_count": 29,
  "closed_at": "2018-03-18T22:45:33Z",
  "total_coins": "103768249377.9199212",
  "fee_pool": "1417457.4202712",
  "base_fee_in_stroops": 100,
  "base_reserve_in_stroops": 5000000,
  "max_tx_set_size": 50,
  "protocol_version": 9
}

# ... streams forever ...
```

There are a few interesting things in the output above (which shows the most recent ledger at the time of this writing). Firstly, we used the `--network public` flag to specify that we want the live network. You could also have done `lumen set config:network public`, but it's good practice to leave the default network at `test`.

You can see that there were 3 transactions and 29 operations in total -- we'll dig into those shortly. You can also see that there are abou 103 billion coins (lumens) in circulation.

Okay, let's look at the first transaction in that ledger. Query the URL in the `href` field of the `transactions` struct under `_links`. From here on we'll refer to JSON paths using dot notation, so the last path would be `_links.transactions.href`.

```sh
$ curl https://horizon.stellar.org/ledgers/16889095/transactions | jq '._embedded.records[0]'
```

...which outputs:

```json
{
  "_links": {
    "self": {
      "href": "https://horizon.stellar.org/transactions/20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54"
    },
    "account": {
      "href": "https://horizon.stellar.org/accounts/GDMIF55WK2LUUQ77TRE4RDDC5325I2GY555HRQWI5IE7MGG7662OSBWS"
    },
    "ledger": {
      "href": "https://horizon.stellar.org/ledgers/16889095"
    },
    "operations": {
      "href": "https://horizon.stellar.org/transactions/20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54/operations{?cursor,limit,order}",
      "templated": true
    },
    "effects": {
      "href": "https://horizon.stellar.org/transactions/20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54/effects{?cursor,limit,order}",
      "templated": true
    },
    "precedes": {
      "href": "https://horizon.stellar.org/transactions?order=asc&cursor=72538110684041216"
    },
    "succeeds": {
      "href": "https://horizon.stellar.org/transactions?order=desc&cursor=72538110684041216"
    }
  },
  "id": "20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54",
  "paging_token": "72538110684041216",
  "hash": "20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54",
  "ledger": 16889095,
  "created_at": "2018-03-18T22:45:33Z",
  "source_account": "GDMIF55WK2LUUQ77TRE4RDDC5325I2GY555HRQWI5IE7MGG7662OSBWS",
  "source_account_sequence": "72537633942667267",
  "fee_paid": 100,
  "operation_count": 1,
  "envelope_xdr": "AAAAANiC97ZWl0pD/5xJyIxi7vXUaNjvenjCyOoJ9hjf97TpAAAAZAEBtJgAAAADAAAAAAAAAAAAAAABAAAAAAAAAAMAAAAAAAAAAVJNVAAAAAAAq2nN4Bi7gPQYGpIp0xISfxU8xOHn/rUZdRnF/rGcukUAAAACMXnLAAABhqAAABeRAAAAAAAAAAAAAAAAAAAAAd/3tOkAAABAkQwyGKYer821RJ4Wr1vtkU4qa6EGcY8lgr8y7HkiPuUv6uiptgrUZzCxodMN44smFPW6f850XRSAqv1N+rGlCg==",
  "result_xdr": "AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAADAAAAAAAAAAAAAAAAAAAAANiC97ZWl0pD/5xJyIxi7vXUaNjvenjCyOoJ9hjf97TpAAAAAAAyLRIAAAAAAAAAAVJNVAAAAAAAq2nN4Bi7gPQYGpIp0xISfxU8xOHn/rUZdRnF/rGcukUAAAACMXnLAAABhqAAABeRAAAAAAAAAAAAAAAA",
  "result_meta_xdr": "AAAAAAAAAAEAAAADAAAAAAEBtQcAAAACAAAAANiC97ZWl0pD/5xJyIxi7vXUaNjvenjCyOoJ9hjf97TpAAAAAAAyLRIAAAAAAAAAAVJNVAAAAAAAq2nN4Bi7gPQYGpIp0xISfxU8xOHn/rUZdRnF/rGcukUAAAACMXnLAAABhqAAABeRAAAAAAAAAAAAAAAAAAAAAwEBtQcAAAAAAAAAANiC97ZWl0pD/5xJyIxi7vXUaNjvenjCyOoJ9hjf97TpAAAAAjNAgBQBAbSYAAAAAwAAAAEAAAABAAAAAMRxxkNwYslQaok0LlOKGtpATS9Bzx06JV9DIffG4OF1AAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAQEBtQcAAAAAAAAAANiC97ZWl0pD/5xJyIxi7vXUaNjvenjCyOoJ9hjf97TpAAAAAjNAgBQBAbSYAAAAAwAAAAIAAAABAAAAAMRxxkNwYslQaok0LlOKGtpATS9Bzx06JV9DIffG4OF1AAAAAAAAAAABAAAAAAAAAAAAAAAAAAAA",
  "fee_meta_xdr": "AAAAAgAAAAMBAbT1AAAAAAAAAADYgve2VpdKQ/+cSciMYu711GjY73p4wsjqCfYY3/e06QAAAAIzQIB4AQG0mAAAAAIAAAABAAAAAQAAAADEccZDcGLJUGqJNC5TihraQE0vQc8dOiVfQyH3xuDhdQAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAEBAbUHAAAAAAAAAADYgve2VpdKQ/+cSciMYu711GjY73p4wsjqCfYY3/e06QAAAAIzQIAUAQG0mAAAAAMAAAABAAAAAQAAAADEccZDcGLJUGqJNC5TihraQE0vQc8dOiVfQyH3xuDhdQAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAA==",
  "memo_type": "none",
  "signatures": [
    "kQwyGKYer821RJ4Wr1vtkU4qa6EGcY8lgr8y7HkiPuUv6uiptgrUZzCxodMN44smFPW6f850XRSAqv1N+rGlCg=="
  ]
}
```

You can see that the transaction consists of 1 operation (hence the fee of 100 stroops, charged to the address in `source_account`.) Let's grab the operations link from `_links.operations.href` and extract the first operation.

```sh
curl https://horizon.stellar.org/transactions/20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54/operations | jq '._embedded.records[0]'
{
  "_links": {
    "self": {
      "href": "https://horizon.stellar.org/operations/72538110684041217"
    },
    "transaction": {
      "href": "https://horizon.stellar.org/transactions/20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54"
    },
    "effects": {
      "href": "https://horizon.stellar.org/operations/72538110684041217/effects"
    },
    "succeeds": {
      "href": "https://horizon.stellar.org/effects?order=desc&cursor=72538110684041217"
    },
    "precedes": {
      "href": "https://horizon.stellar.org/effects?order=asc&cursor=72538110684041217"
    }
  },
  "id": "72538110684041217",
  "paging_token": "72538110684041217",
  "source_account": "GDMIF55WK2LUUQ77TRE4RDDC5325I2GY555HRQWI5IE7MGG7662OSBWS",
  "type": "manage_offer",
  "type_i": 3,
  "created_at": "2018-03-18T22:45:33Z",
  "transaction_hash": "20dbabf0fbdcadbd4e96848fd4a251791af1b4cef9c35ba614b180a6d6d79b54",
  "amount": "942.0000000",
  "price": "16.5755014",
  "price_r": {
    "n": 100000,
    "d": 6033
  },
  "buying_asset_type": "credit_alphanum4",
  "buying_asset_code": "RMT",
  "buying_asset_issuer": "GCVWTTPADC5YB5AYDKJCTUYSCJ7RKPGE4HT75NIZOUM4L7VRTS5EKLFN",
  "selling_asset_type": "native",
  "offer_id": 0
}
```

Looks like it's an offer to sell 942 lumens for `RMT` on Stellar's distributed exchange. How can you tell? Well, the `type` of operation is `manage_offer`, which means that this is related to a trade on the DEX, the `selling_asset_type` is `native`, which is lumens (XLM), the `amount` is `942`, and `buying_asset_code` is `RMT`.

Now, let's look at the account in `source_account`. This time, we'll use `lumen`.

```sh
$ lumen info GDMIF55WK2LUUQ77TRE4RDDC5325I2GY555HRQWI5IE7MGG7662OSBWS --network public
{
  "address": "GDMIF55WK2LUUQ77TRE4RDDC5325I2GY555HRQWI5IE7MGG7662OSBWS",
  "balances": [
    {
      "asset": {
        "code": "RMT",
        "issuer": "GCVWTTPADC5YB5AYDKJCTUYSCJ7RKPGE4HT75NIZOUM4L7VRTS5EKLFN",
        "type": "\"credit_alphanum4\""
      },
      "amount": "13750.3198383",
      "limit": "922337203685.4775807"
    }
  ],
  "signers": [
    {
      "public_key": "GDMIF55WK2LUUQ77TRE4RDDC5325I2GY555HRQWI5IE7MGG7662OSBWS",
      "weight": 1,
      "key": "GDMIF55WK2LUUQ77TRE4RDDC5325I2GY555HRQWI5IE7MGG7662OSBWS",
      "type": "ed25519_public_key"
    }
  ],
  "flags": {
    "auth_required": false,
    "auth_revocable": false,
    "auth_immutable": false
  },
  "native_balance": {
    "asset": {
      "code": "XLM",
      "issuer": "",
      "type": "\"native\""
    },
    "amount": "2.8002402",
    "limit": ""
  },
  "home_domain": "",
  "thresholds": {
    "high": 0,
    "medium": 0,
    "low": 0
  },
  "data": {},
  "seq": "72537633942667287"
}
```

You can tell that the account holder has about 2.8 lumens and over 13000 `RMT`. The account has a single signer, which is simply the master key (see chapter 4.)

## Watching an account

Knowing how to dig into individual ledgers is useful when you know which ledger some transacation occurred in, however a lot of the time, you just want to debug the transactions on a single account. You can use the `lumen watch transactions` and `lumen watch payments` commands with the address of the account to follow a stream of transactions or payments.

Let's move back to the test network and try it out. Open up a new command-line terminal and type:

```sh
$ lumen set config:network test
$ lumen account new mary
$ lumen watch payments mary --cursor start
# ... this blocks ...
```

Then, on a different terminal, type:

```sh
$ lumen friendbot mary
```

Switch back to the first terminal -- if friendbot succeeded in funding Mary's account, you should see this:

```json
{
  "id": "34287651636912129",
  "type": "create_account",
  "paging_token": "34287651636912129",
  "_links": {
    "transaction": {
      "href": "https://horizon-testnet.stellar.org/transactions/d97481dddb1fc7d4164beaf868d7a222f3dbf6ffe7fd6dd5d599681a9a262195"
    }
  },
  "account": "GCLPPDSQNXTS2OAXPELZHDWT6TB36SPOZVY7QZZDCXXVUM4CZWOXR4GA",
  "starting_balance": "10000.0000000",
  "from": "",
  "to": "",
  "asset_type": "",
  "asset_code": "",
  "asset_issuer": "",
  "amount": "",
  "Memo": {
    "memo_type": "none",
    "memo": ""
  }
}
```

There was a new `create_account` operation funding Mary's account with `10000` lumens. Let's pay her again.

```sh
$ lumen pay 10 --from bob --to mary --memotext "hi mary"
```

You should now see a `payment` operation show up on the watching terminal:

```json
{
  "id": "34287978054422529",
  "type": "payment",
  "paging_token": "34287978054422529",
  "_links": {
    "transaction": {
      "href": "https://horizon-testnet.stellar.org/transactions/d546ef6942625e628e5c80f7a60ad887674bcfb9a4109c845a922afa947b8504"
    }
  },
  "account": "",
  "starting_balance": "",
  "from": "GBGMPKS4JY6SKHSZQ5KHM4GRSJNRXINH57HMJ4T3UXDE2WIU6PYFPRPC",
  "to": "GCLPPDSQNXTS2OAXPELZHDWT6TB36SPOZVY7QZZDCXXVUM4CZWOXR4GA",
  "asset_type": "native",
  "asset_code": "",
  "asset_issuer": "",
  "amount": "10.0000000",
  "Memo": {
    "memo_type": "text",
    "memo": "hi mary"
  }
}
```

Notice the memo text on the transaction?

* streaming
* offers
* info
* sequence number