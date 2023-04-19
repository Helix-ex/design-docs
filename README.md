# High-level design

![helix high-level design](docs/images/helix-design.png)

The Helix hybrid exchange consists of the following components

* helix web UI: will be used by humans for authentication, funding and trading
* "[funding](https://matrix.fandom.com/wiki/The_Landlord)" proxy: exposes the user management and funding services performed by the "landlord" smart contract to the web UI; shields the rest of the system from the [complexity of interacting with an ICP smart contract](https://internetcomputer.org/docs/current/references/ic-interface-spec/#http-call-overview).
* "[keymaker](https://matrix.fandom.com/wiki/The_Keymaker)": a service that provides funding use case integration between the "funding" proxy and the "V12" off-chain exchange
* "V12": high-speed, low latency exchange running off the chain.

# funding proxy

Users will register with a `PrincipalId` they control. The `landlord` canister will keep user funds in an [account](https://internetcomputer.org/docs/current/references/ledger#_accounts) that is derived from the canister's `PrincipalId` and a user identifier (`uid`). The latter is allocated to the user (to his `PrincipalId` to be more precise) upon registration.

The funding proxy is in charge of
- user registration
- allocating segregated deposit addresses for each user and blockchain supported

Please note that:

1. the `funding-proxy` service will have a copy of the user data in its encrypted database. The canister's copy will be regarded as the master record however.
1. any funds owned by a user will be kept in the corresponding `landlord` canister account or user-specific deposit addresses for non-ICP assets
1. all accounts that hold user funds are controlled by the canister's `PrincipalId`

[API docs](https://app.swaggerhub.com/apis/MUHAREM_2/funding-proxy_api/1.0.14)

# keymaker

Registered users hold 2 wallets with the exchange, a
- funding wallet: deposits arrive here and funds in this wallet can be withdrawn at any time
- trading wallet: users need to transfer the funds they want to trade to this wallet; funds in this wallet cannot be withdrawn

The `keymaker` service is responsible for
- monitoring the blockchains supported for arriving deposits
- withdrawals
- transfer of funds between the funding and trading wallet

[API docs](https://app.swaggerhub.com/apis/MUHAREM_2/keymaker-fund_api/1.0.4)


# V12 trading engine

The V12 trading engine was built from scratch to facilitate low-latency and high-volume order management for retail as well as institutional users and trading bots.
It offers websockets based APIs for
- [market data distribution](https://helix-ex.github.io/apidocs/docs/market-data/#market-data-api)
- [order management](https://helix-ex.github.io/apidocs/docs/order-management/#order-management-api)

# general principles / considerations

* high-volume APIs use websockets whereas low-volume APIs use https+REST
* all APIs (except for the market data API) will be authenticated
* backend services will use service accounts (key/secret) to authenticate to each other (example: `keymaker` to `funding-proxy`)
* service account credentials (key/secret) will be rotated frequently (every N hours) (N=4h?)
* key material (or any other sensitive data) must be
  * encrypted at rest
  * zeroized immediately after use (to limit exposure / damage from memory dumps)
* we don't want to traverse the public internet in order to obtain key material or service account credentials
* REST APIs calls need to protect against replay attacks by using a timestamp (5 seconds in the past or shorter)
* we apply [defense in depth](https://en.wikipedia.org/wiki/Defense_in_depth_(computing)) to protect our systems
* every backend service will have its own database encryption key that is rotated every K hours (K=24h?)
* we want to use proven / open source tools if they exist and minimize building/maintaining such tools ourselves
* all backend services will be coded in rust
* all APIs must be rate limited to prevent abuse
* amounts shall be passed as strings across APIs and be handled as decimal types in code (e.g. using a [package like this](https://pkg.go.dev/github.com/shopspring/decimal)) in order to avoid rounding issues
