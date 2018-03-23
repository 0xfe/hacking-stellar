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

The live network is where real value is transacted. Lumens on the live network cost real money, and can be bought at many cryptocurrency exchanges. The test network on the other uses fake luments that developers can request at any time.

You can see see the status of the networks on the [Stellar Network Dashboard](https://dashboard.stellar.org/).

### Horizon

When you transact on Stellar with the Lumen tool (or with the SDKs), you don't directly interact with the core network. You're actually interacting with [Horizon](https://www.stellar.org/developers/reference/), which is an API gateway into the core network. Horizon actively ingests data from Stellar core, and exports a JSON/HTTP API that can be consumed by any HTTP client (including your browser.)

The Stellar organization runs a set of publically available Horizon servers, which most tools and SDKs default to using. However, if you're deploying applications into production, it's always better to run your own servers. This not only lets you manage load and scale, but also protects you from security compromises of infrastructure you don't fully control.

With Lumen, you can use the `set config:network` command or the `--network` flag to specify a custom Horizon server.

```sh
$ lumen balance bob --network 'custom;http://my.server:8000;networkpassphrase`
```

The passphrase above is for the network you're connecting to (`live` or `test`), not for the Horizon server. You can use any type of firewalling or HTTP proxying to restrict access to your server.

#### XDR Encoding

Messages on the Stellar core network are encoded in a format called XDR (*External Data Representation*), specified in [RFC 4506](https://tools.ietf.org/html/rfc4506.html). XDR is a binary format designed to be both bandwidth friendly and compatible across various machine architectures.

For the most part, Horizon does all the heavy lifting for us, translating messages between XDR and JSON, however the SDKs and libraries do need to work with XDR to sign and submit transactions. XDR structures embedded in JSON are always *base64*-encoded -- Base64 is a text-friendly encoding of binary data, suitable for embedding into strings.

For example, let's generate an unsigned transaction without submitting it to the network:

```sh
$ lumen pay 10 --from issuer --to distributor --nosign --nosubmit >payment.txt
$ cat payment.txt
AAAAADS+VZ+XjxzKscWEqihJhF+xc1iFEHd1g5j+/vIlSphIAAAAZAB6eYwAAAAFAAAAAAAAAAAAAAABAAAAAAAAAAEAAAAA9PBUt4iVNaTh8rzkjyn6nhCtCCZ76N58aTjRCVW/h1cAAAAAAAAAAAX14QAAAAAAAAAAAA==
```

Lumen generated the requested payment transaction and encoded the XDR into base64. To convert it back to human-readable JSON, you can use `lumen tx decode`.

```sh
$ lumen tx decode $(cat payment.txt)
{"Tx":{"SourceAccount":{"Type":0,"Ed25519":[52,190,85,159,151,143,28,202,177,197,132,170,40,73,132,95,177,115,88,133,16,119,117,131,152,254,254,242,37,74,152,72]},"Fee":100,"SeqNum":34473589361082373,"TimeBounds":null,"Memo":{"Type":0,"Text":null,"Id":null,"Hash":null,"RetHash":null},"Operations":[{"SourceAccount":null,"Body":{"Type":1,"CreateAccountOp":null,"PaymentOp":{"Destination":{"Type":0,"Ed25519":[244,240,84,183,136,149,53,164,225,242,188,228,143,41,250,158,16,173,8,38,123,232,222,124,105,56,209,9,85,191,135,87]},"Asset":{"Type":0,"AlphaNum4":null,"AlphaNum12":null},"Amount":100000000},"PathPaymentOp":null,"ManageOfferOp":null,"CreatePassiveOfferOp":null,"SetOptionsOp":null,"ChangeTrustOp":null,"AllowTrustOp":null,"Destination":null,"ManageDataOp":null}}],"Ext":{"V":0}},"Signatures":null}
```

You can sign and submit this transcation to the network with `lumen tx sign` and `lumen tx submit`.

```sh
$ lumen tx sign $(cat payment.txt) --signers issuer >payment.signed.txt
$ lumen tx submit $(cat payment.signed.txt)
```

#### Using namespaces to switch networks

It might become tedious to keep setting the network parameters when you're working across different networks. Lumen lets you segregate your configuration and aliases across namespaces.

```sh
# Create a new namespace called "custom", and set its network
$ lumen ns custom
$ lumen set config:network 'custom:http://my.server:8000;passphrase'
$ lumen account new bob


# Now switch to a different namespace and set its network
$ lumen ns project1
$ lumen set config:network public
$ lumen account address bob
# unknown user: bob

# Switch back to the first namespace
$ lumen ns custom
$ lumen get config:network
# custom:http://my.server:8000;passphrase

$ lumen account address bob
# GVA4EAA...
```

## Ledgers, transactions, and operations

Every few seconds, the Stellar network commits a ledger to its global database. The ledger consists of a set of transactions, each containing one or more operations. The *transaction envelope* is the header that surrounds a transaction and includes common transaction-level fields.

Every transaction is charged a [fee](https://www.stellar.org/developers/guides/concepts/fees.html), derived from the number of operations in the transaction, and billed to its respective *source account*. The operations could be anything from payments, to offers, to administrative tasks such as managing signers. You can see the full list of Stellar operations [here](https://www.stellar.org/developers/guides/concepts/list-of-operations.html).

## Watching the ledger

Let's start off by watching Stellar for new ledger entries with `lumen watch ledger`. All `lumen watch ...` commands run forever, streaming updates to your terminal. They're a handy way to troubleshoot your applications, and can be terminated any time with `Ctrl-C`.

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

There's also a `_links` section that lists a set of URLs relevant to this specific ledger. For example, let's look at the first transaction in that ledger by querying the URL in the `href` field of `transactions` under `_links`. From here on we'll refer to JSON paths using dot notation, so the last path would be `_links.transactions.href`.

```sh
$ curl https://horizon.stellar.org/ledgers/16889095/transactions | jq '._embedded.records[0]'
```

We used `jq '._embedded.records[0]` to filter out just the first transaction. *Exercise:* remove the filter and see what you find.

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

The `operation_count` field tells us that the transaction consists of just one operation, which also explains the fee of 100 stroops (charged to the address in `source_account`.) Let's grab the operations link from `_links.operations.href` and extract the first operation.

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

*Exercise:* Use `lumen dex list [account]` to see all the offers made by an account.

Now, let's dig into at the account in `source_account`.

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

You can tell that the account holder has about 2.8 lumens and over 13000 `RMT`. The account has a single signer, which is simply the master key (see [Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) for more about signing keys.)

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
$ lumen watch transactions mary
```

Now move back to your working terminal and fund Mary's account.

```sh
$ lumen friendbot mary
```

Switch back to the payment terminal -- if friendbot succeeded in funding Mary's account, you should see this:

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

Notice the memo text? It's actually not part of the operation, but pulled in here for convenience. The memo fields are part of the encompassing transaction, so although you can lump many operations into a transaction, they can have only a single memo.

If you switch to the transactions terminal, you should see multiple operations with just the details of the transaction envelope.

#### Lumen Detour: Verbose Logging

Lumen can output detailed logs for an operation with the `-v` flag. This is handy when your Stellar operations aren't behaving as expected.

```sh
$ lumen pay 10 USD --from mo --to mary -v
# DEBU[0000] LUMEN_ENV not set                             type=setup
# DEBU[0000] using storage driver file with /Users/mo/.lumen-data.json  type=setup
# DEBU[0000] using default store                           type=setup
# DEBU[0000] reading file: /Users/mo/.lumen-data.json      method=new type=filestore
# DEBU[0000] getting global:ns                             method=GetGlobalVar type=cli
# DEBU[0000] got val: default (expires: false, expires_on: 2018-03-21 08:00:43.557659 -0400 EDT)  key="global:ns" method=get type=filestore
# DEBU[0000] using namespace from store                    type=setup
# DEBU[0000] namespace: default                            type=setup
# DEBU[0000] getting default:vars:config:network           method=GetVar type=cli
# DEBU[0000] not found, expired: false                     key="default:vars:config:network" method=get type=filestore
# DEBU[0000] getting default:asset:USD:code                method=GetVar type=cli
# DEBU[0000] got val: USD (expires: false, expires_on: 2018-03-21 08:02:49.189346 -0400 EDT)  key="default:asset:USD:code" method=get type=filestore
# DEBU[0000] getting default:asset:USD:issuer              method=GetVar type=cli
# DEBU[0000] got val: GAGUZYRM2G7235EM3C3WY33UHTWORNR7MX2J3OT4F3D46HAFSUEA63LL (expires: false, expires_on: 2018-03-21 08:02:49.188011 -0400 EDT)  key="default:asset:USD:issuer" method=get type=filestore
# DEBU[0000] getting default:asset:USD:type                method=GetVar type=cli
# DEBU[0000] got val: credit_alphanum4 (expires: false, expires_on: 2018-03-21 08:02:49.189837 -0400 EDT)  key="default:asset:USD:type" method=get type=filestore
# DEBU[0000] got asset: &{Code:USD Issuer:GAGUZYRM2G7235EM3C3WY33UHTWORNR7MX2J3OT4F3D46HAFSUEA63LL Type:credit_alphanum4}
# DEBU[0000] getting default:account:mo:seed               method=GetVar type=cli
# DEBU[0000] got val: SAZW22BHCNGEIQNSTWR5OYG3HC44CTPXV4342OEQJDJZWCPHYUQLZCC7 (expires: false, expires_on: 2018-03-06 09:14:24.691781 -0500 EST)  key="default:account:mo:seed" method=get type=filestore
# DEBU[0000] getting default:account:mary:address          method=GetVar type=cli
# DEBU[0000] got val: GD6JJSOKWI7U2YDCMZ3YGPKNOP6W3D7K34HWLC6WHD32CKJJVALV7OBK (expires: false, expires_on: 2018-03-21 08:01:13.923333 -0400 EDT)  key="default:account:mary:address" method=get type=filestore
# DEBU[0000] fund: false, err <nil>                        cmd=pay
# DEBU[0000] paying 10 USD/GAGUZYRM2G7235EM3C3WY33UHTWORNR7MX2J3OT4F3D46HAFSUEA63LL from SAZW22BHCNGEIQNSTWR5OYG3HC44CTPXV4342OEQJDJZWCPHYUQLZCC7 to GD6JJSOKWI7U2YDCMZ3YGPKNOP6W3D7K34HWLC6WHD32CKJJVALV7OBK, opts: &{ctx:<nil> handlers:map[] hasFee:false fee:0 hasTimeBounds:false timeBounds:0 memoType:0 memoText: memoID:0 skipSignatures:false signerSeeds:[] hasCursor:false cursor: hasLimit:false limit:0 sortDescending:false passiveOffer:false sourceAddress: sendAsset:<nil> maxAmount: path:[] isMultiOp:false multiOpSource:}  cmd=pay
# DEBU[0000] signing transaction, seq: 33366067619299340   lib=microstellar method=Tx.Sign
# DEBU[0000] signed transaction, payload: AAAAAPGR63kaYI062wyHd+LARbBzZOCK9pDleNq8UkGhV4sZAAAAZAB2ikMAAAAMAAAAAAAAAAAAAAABAAAAAAAAAAEAAAAA/JTJyrI/TWBiZneDPU1z/W2P6t8PZYvWOPehKSmoF18AAAABVVNEAAAAAAANTOIs0b+t9IzYt2xvdDzs6LY/ZfSdunwux88cBZUIDwAAAAAF9eEAAAAAAAAAAAGhV4sZAAAAQIJduVNXFgBu3/OD6uLLJJlkZD4i8JoHHorCxKi0L0LnbnVvsl2pVuazburcSH43N6AYPHI9kD/M6B03kZaz4gg=  lib=microstellar method=Tx.Sign
# DEBU[0000] submitting transaction to network test        lib=microstellar method=Tx.Submit
# DEBU[0001] transaction submitted to ledger 8026171 with hash abbac2c2906342dff927c7a88075487418c787bc4550fea6353dfc2c2faa75b2  lib=microstellar method=Tx.Submit
```

## Debugging scenario: Token distribution

Let's go through a fixed-supply token distribution scenario. This is where an issuer issues some predetermined number of tokens to a distributor, and then permanently disables their issuer account so no more can be issued.

```sh
# Create a new namespace called `hs` for this scenario.
$ lumen ns hs

# Create accounts for issuer and distributor.
$ lumen account new issuer
$ lumen account new distributor

# Issue 1000 HAK tokens from issuer to distributor.
$ lumen pay 1000 HAK:issuer_account distributor
# ERRO[0000] bad --from address: issuer_account             cmd=pay
```

Okay, we intentionally put the typo in there, but let's pretend that was a mistake and rerun the command with verbose logging.

```sh
$ lumen pay 1000 HAK:issuer --from issuer_account --to distributor -v
# ... snipped ...
DEBU[0000] getting hs:account:issuer:address             method=GetVar type=cli
DEBU[0000] got val: GA2L4VM7S6HRZSVRYWCKUKCJQRP3C42YQUIHO5MDTD7P54RFJKMEQJ7Q (expires: false, expires_on: 2018-03-20 16:23:56.130665 -0400 EDT)  key="hs:account:issuer:address" method=get type=filestore
DEBU[0000] got asset: &{Code:HAK Issuer:GA2L4VM7S6HRZSVRYWCKUKCJQRP3C42YQUIHO5MDTD7P54RFJKMEQJ7Q Type:credit_alphanum4}
DEBU[0000] getting hs:account:issuer_account:seed        method=GetVar type=cli
DEBU[0000] not found, expired: false                     key="hs:account:issuer_account:seed" method=get type=filestore
DEBU[0000] getting hs:account:issuer_account:address     method=GetVar type=cli
DEBU[0000] not found, expired: false                     key="hs:account:issuer_account:address" method=get type=filestore
DEBU[0000] invalid address, seed, or account name: issuer_account  cmd=pay
ERRO[0000] bad --from address: issuer_account            cmd=pay
```

Right away, we'd notice that Lumen could look up the address for `hs:account:issuer` but not `hs:account:issuer_account`. (The `hs` prefix designates the current namespace.) Alright, let's try that again with the right account alias.

```sh
$ lumen pay 1000 HAK:issuer --from issuer --to distributor
# ERRO[0001] payment failed: 404: Resource Missing         cmd=pay
```

Another error. Looking at the [horizon reference](https://www.stellar.org/developers/horizon/reference/errors/not-found.html) for `404`, we can tell that some resource in our request is unknown to Stellar. The only resources in the command are `issuer` and `distributor`. Which one is it? We'll use the `lumen info` command to investigate.

```sh
$ lumen info issuer
ERRO[0000] cant load account: 404: Resource Missing     cmd=balance

$ lumen info distributor
ERRO[0000] cant load account: 404: Resource Missing     cmd=balance
```

Both of them are missing! Ofcourse, we never funded the accounts, so Stellar doesn't know about them.

```sh
# Ask Friendbot to create issuer's account
$ lumen friendbot issuer

# Verify that the resource exists
$ lumen info issuer
{
  "address": "GA2L4VM7S6HRZSVRYWCKUKCJQRP3C42YQUIHO5MDTD7P54RFJKMEQJ7Q",
  "balances": [],
  "signers": [
    {
      "public_key": "GA2L4VM7S6HRZSVRYWCKUKCJQRP3C42YQUIHO5MDTD7P54RFJKMEQJ7Q",
      "weight": 1,
      "key": "GA2L4VM7S6HRZSVRYWCKUKCJQRP3C42YQUIHO5MDTD7P54RFJKMEQJ7Q",
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
    "amount": "10000.0000000",
    "limit": ""
  },
  "home_domain": "",
  "thresholds": {
    "high": 0,
    "medium": 0,
    "low": 0
  },
  "data": {},
  "seq": "34473589361082369"
}

# Use issuer to create distributor's account and verify its existence
$ lumen pay 1000 --from issuer --to distributor --fund
$ lumen info distributor
```

Now that the two accounts exist, lets issue our new `HAK` asset. And to save us the effort of typing in `HAK:issuer` all the time, let's create an alias for it.

```sh
# Create asset alias HAK issued by issuer
$ lumen asset set HAK issuer

# Issue 1000 HAKs
$ lumen pay 1000 HAK --from issuer to --distributor
# ERRO[0005] payment failed: 400: Transaction Failed (&{tx_failed [op_no_trust]})  cmd=pay
```

Tranasaction failed. It's a bit clearer this time because of the result code `op_no_trust` -- the distributor does not have a trustline to the asset. (See [Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) for more on custom assets and trustlines.)

```sh
$ lumen trust create distributor HAK
$ lumen pay 1001 HAK --from issuer --to distributor

# That seemed to have worked, lets verify it.
$ lumen balance distributor HAK
# 1001.0000000
```

Darnit, we created one too many assets. No problem. It turns out that sending an asset back to the issuer destroys it. You can verify this by checking the issuer's `HAK` balance.

```sh
# Issuers never carry balances for assets they issue.
$ lumen balance issuer HAK
0

# Send back one HAK.
$ lumen pay 1 HAK --from distributor --to issuer

# Verify balances.
$ lumen balance distributor HAK
1000.0000000
$ lumen balance issuer HAK
0.0000000
```

Great! It worked. The final step is to permanently disable the issuer's account so that no more new HAK assets can be created.

```sh
# Set the weight of the issuer's master key to 0
$ lumen signer masterweight issuer 0

# Make sure there are no other working keys for issuer
$ lumen signer list issuer
# address:GA2L4VM7S6HRZSVRYWCKUKCJQRP3C42YQUIHO5MDTD7P54RFJKMEQJ7Q weight:0

# Try to issue new assets (this should return an error.)
$ lumen pay 1000 HAK --from issuer --to distributor
# ERRO[0001] payment failed: 400: Transaction Failed (&{tx_bad_auth []})  cmd=pay
```

We now have a fixed supply of 1000 `HAK` tokens to redistribute. To close this off, lets sell a hundred of them on the DEX for 10 XLM each.

```sh
$ lumen dex trade distributor --sell HAK --amount 200 --buy native --price 10
$ lumen dex list distributor
(149231) selling 200.0000000 HAK for lumens at 10.0000000 lumens/HAK
```

# Onward

We learned a bunch of useful Stellar debugging techiques in this chapter, so to summarize:

* [Horizon](https://www.stellar.org/developers/horizon/reference/index.html) is an API gateway to Stellar, and how most applications interact with the core network.
* Leave the `lumen watch ...` commands running on separate terminals as you develop your application.
* Use `curl` to query any relevant `_links` for misbehaving transactions.
* Use Lumen's `-v` flag to debug alias errors or horizon responses.
* One of the most common newbie mistakes is forgetting to create a funded account. A `404` return code is a strong clue that an account does not exist on Stellar's ledger.

This concludes the current edition of Hacking Stellar (so far.) More chapters to come, vote for your favourite on the front page.

[Front](https://github.com/0xfe/hacking-stellar/blob/master/README.md) -
[Chapter 1](https://github.com/0xfe/hacking-stellar/blob/master/1-launch.md) -
[Chapter 2](https://github.com/0xfe/hacking-stellar/blob/master/2-payments.md) -
[Chapter 3](https://github.com/0xfe/hacking-stellar/blob/master/3-assets.md) -
[Chapter 4](https://github.com/0xfe/hacking-stellar/blob/master/4-multisig.md) -
[Chapter 5](https://github.com/0xfe/hacking-stellar/blob/master/5-dex.md) -
[Chapter 6](https://github.com/0xfe/hacking-stellar/blob/master/6-debugging.md)