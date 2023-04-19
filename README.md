# High-level design

![helix high-level design](docs/images/helix-design.png)

The Helix hybrid exchange consists of the following components

* helix web UI: will be used by humans for authentication, funding and trading
* "[funding](https://matrix.fandom.com/wiki/The_Landlord)" proxy: exposes the user management and funding services performed by the "landlord" smart contract to the web UI; shields the rest of the system from the [complexity of interacting with an ICP smart contract](https://internetcomputer.org/docs/current/references/ic-interface-spec/#http-call-overview).
* "[keymaker](https://matrix.fandom.com/wiki/The_Keymaker)": a service that provides funding use case integration between the "funding" proxy and the "V12" off-chain exchange
* "V12": high-speed, low latency exchange running off the chain.

# funding proxy

The funding proxy is in charge of
- user registration
- allocating segregated deposit addresses for each user and blockchain supported

Please note that:

1. the `funding-proxy` service will have a copy of the user data in its encrypted database. The canister's copy will be regarded as the master record however.
1. any funds owned by a user will be kept in the corresponding `landlord` canister account or user-specific deposit addresses for non-ICP assets
1. all accounts that hold user funds are controlled by the canister's `PrincipalId`


# keymaker

Registered users hold 2 wallets with the exchange, a
- funding wallet: deposits arrive here and funds in this wallet can be withdrawn at any time
- trading wallet: users need to transfer the funds they want to trade to this wallet; funds in this wallet cannot be withdrawn

The `keymaker` service is responsible for
- monitoring the blockchains supported for arriving deposits
- withdrawals
- transfer of funds between the funding and trading wallet
