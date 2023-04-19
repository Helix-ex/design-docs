# High-level design

![helix high-level design](docs/images/helix-design.png)

The Helix hybrid exchange consists of the following components

* helix web UI: will be used by humans for authentication, funding and trading
* "[funding](https://matrix.fandom.com/wiki/The_Landlord)" proxy: exposes the user management and funding services performed by the "landlord" smart contract to the web UI; shields the rest of the system from the [complexity of interacting with an ICP smart contract](https://internetcomputer.org/docs/current/references/ic-interface-spec/#http-call-overview).
* "[keymaker](https://matrix.fandom.com/wiki/The_Keymaker)": a service that provides funding use case integration between the "funding" proxy and the "V12" off-chain exchange
* "V12": high-speed, low latency exchange running off the chain.

# User flows

1. [user registration](docs/helix-user-registration-flow.md)

# Design principles / ideas

1. all backend services will be coded in rust
1. all APIs must be rate limited to prevent abuse
1. amounts shall be passed as strings across APIs and be handled as decimal types in code (e.g. using a [package like this](https://pkg.go.dev/github.com/shopspring/decimal)) in order to avoid rounding issues

# Users and accounts

Users will register with a `PrincipalId` they control. The `landlord` canister will keep user funds in an [account](https://internetcomputer.org/docs/current/references/ledger#_accounts) that is derived from the canister's `PrincipalId` and a user identifier (`uid`). The latter is allocated to the user (to his `PrincipalId` to be more precise) upon registration.

IOW, the `landlord` canister will maintain a

* `PrincipalId -> uid` mapping
* `NextUID` variable ([local state](https://internetcomputer.org/docs/current/references/security/rust-canister-development-security-best-practices/#use-thread_local-with-cellrefcell-for-state-variables-and-put-all-your-globals-in-one-basket))

in the [canister stable memory](https://internetcomputer.org/docs/current/references/security/rust-canister-development-security-best-practices/#consider-using-stable-memory-version-it-test-it).

Please note that:

1. the `funding-proxy` service will have a copy of the data above in its database. The canister's copy will be regarded as the master record however.
1. any funds owned by a user will be kept in the corresponding `landlord` canister account or user-specific deposit addresses for non-ICP assets
1. all accounts that hold user funds are controlled by the canister's `PrincipalId`

In order to facilitate canister upgrades both the mapping as well as the `NextUID` variable above must be committed to stable memory [as explained here](https://internetcomputer.org/docs/current/developer-docs/build/languages/motoko/upgrades/#preupgrade-and-postupgrade-system-methods).

# Canister action

The `landlord` canister API functions will be called by the `funding-proxy` and `keymaker`. The latter only requires the `tx_sign()` method in order to

* move funds between the funding and the trading wallet
* execute withdrawals

Functions on the canister's API (via a https POST requests) are invoked in an asynchronous fashion and the proxy needs to [monitor the state](https://internetcomputer.org/docs/current/references/ic-interface-spec/#http-call-overview) of the respective request.

# Request states

Whenever a client calls the [`funding-proxy` API](api/landlord_proxy.yaml) and gets back an [`async_result`](api/landlord_proxy.yaml#L426) the

* associated request will be executed in asynchronous fashion
* client is expected to [check on the request condition](api/landlord_proxy.yaml#L203) until it reaches a final state

The request states are as follows:

* `new`: the request was received but is not being processed
* `open`: the request is being worked on
* `success`: the request has been completed successfully
* `failure`: the request has failed

# Configuring and testing

At the moment, to launch the environment for development and testing, you need to start 6 services:

* Postgresql database
* Local ICP replica
* Vault (encryption key storage)
* funding_proxy service (works with Postgresql and local ICP replica)
* keymaker_trade service (works with Postgresql database)
* keymaker_fund service (works with Postgresql database)
* seraph service (works with Postgresql database)

To prepare the environment for running the project, you need to execute the `make fetch_canisters` command, (what the command does - see the section on useful utilities in the makefile).
To run the entire project, you need to run the `make vault_init` and `make helix_build_init` commands.
    - The command `make vault_init` will start Vault.
    - The command `make helix_build_init` will start Postgresql, ICP, funding_proxy, keymaker_trade, keymaker_fund, seraph.
There are `make helix_halt` and `make vault_destroy` commands to stop all running services.

To run the tests, you need to run the command:

* `make fnp_test` - to run tests for funding_proxy
* `make kt_test` - to run tests for keymaker_trade
* `make sr_test` - to run tests for keymaker_trade

The test launch command will also start all the necessary services for this (databases, ICP, etc.) and run the tests.

How to write and run a new test, for example, for the funding_proxy service:

* Go to directory [funding_proxy/tests](./funding_proxy/tests)
* Create a new test file or use an existing one
* Writing a new test following the example of existing ones
* Run the `make fnp_test` command to run the test

Useful makefile utilities

* The operation of launching a local ICP replica also includes the deployment of canisters, the methods of which will be called by the funding_proxy service. Canisters are in a separate [repository](https://github.com/Helix-ex/canisters). To correctly launch ICP, all canisters must be placed in the `startup/local_dfx/canisters` folder. To do this, there are auxiliary commands `make fetch_canisters` to load canisters from the repository into the `startup/local_dfx/canisters` folder and the `make clean_canisters` command to delete the `startup/local_dfx/canisters` folder
* The `make db_logs` and similar commands allow you to see the logs of the database services, local ICP replica and helix services, respectively
* The `make fnp_sqlx_prepare` and similar commands allow you to generate a verification file for checking sql queries. They must be called after adding a new sql query to the project or changing an existing one.
* The `make clippy_fmt` command will clean up the services code
* The `make db_init` command allows you to run databases separately from the project
* The `make dfx_init` command allows you to run a local ICP replica separately from the project

## funding-proxy

Deployment configuration:

* APP_ENDPOINT - address to serve REST API
* DATABASE_URL - database connection URL
* DB_CONNECTION_TIME_S - database connection timeout in seconds
* SERAPH_DATABASE_URL - seraph database connection url
* SERAPH_DB_CONNECTION_TIME_S - seraph database connection timeout in seconds
* USER_INACTIVITY_THRESHOLD_DAYS - number of days after which a user is considered inactive
* VAULT_URL - address of `vault` REST API
* VAULT_TOKEN - token to connect to `vault`
* ICP_URL - address of `ICP` API
* ICP_IDENTITY_PRIVATE_KEY - the key that signs all requests to the ICP
* ICP_LANDLORD_CANISTER_ID - Canister ID LANDLORD
* ICP_REQUEST_STATUS_UPDATE_INTERVAL_S - interval in seconds after which the status of all requests processed in the ICP is requested
* ICP_CONNECTION_ATTEMPTS - Number of attempts to connect to the ICP
* ICP_PAUSE_BETWEEN_CONNECTION_ATTEMPTS_S - interval in seconds between attempts to connect to the ICP
* LLP_SUPPORTED_CHAINS - comma-separated list of chains for which a wallet will be created in the landlord canister. Default value: eth
* LOG_LEVEL - Minimum logging level. Possible values(from lowest to highest): trace, debug, info, warn, error. Default value: info

## seraph

Deployment configuration:

* APP_ENDPOINT - address to serve REST API
* DATABASE_URL - database connection URL
* DB_CONNECTION_TIME_S - database connection timeout in seconds
* VAULT_URL - address of `vault` REST API
* VAULT_TOKEN - token to connect to `vault`
* FUNDING_PROXY_URL - address of `funding-proxy` REST API
* LOG_LEVEL - Minimum logging level. Possible values(from lowest to highest): trace, debug, info, warn, error. Default value: info

## keymaker-fund

The following optional variables are supported in the `keymaker-fund` Makefile:

* DB_NAME - database name ("keymaker_fund" by default).
* DB_URL - Database connection URL ("postgresql://postgres:postgres@localhost:27505/${DB_NAME}" by default).
* BTC_NETWORK - Bitcoin network to be used. The possible values are "bitcoin", "testnet", "signet" and "regtest" ("bitcoin" by default).
* BTC_NUM_CONFIRMATIONS - Number of Bitcoin confirmations (6 by default).
* BTC_START_BLOCK - Bitcoin start block.
* BTC_URL - Bitcoin JSON RPC URL.
* ETH_NATIVE_ASSETS - Comma-separated list of "<asset_name>=<url>" mappings.
* ETH_NUM_CONFIRMATIONS - Number of Ethereum confirmation (6 by default).
* ETH_START_BLOCK - Ethereum start block.
* FUNDING_PROXY_ADDRESS - Address of `funding-proxy` REST API ("http://localhost:27503" by default).
* ICP_ASSETS - Comma-separated list of "<asset_name>=<ledger_address>" mappings.
* ICP_IDENTITY_PEM - ICP identity private key PEM content.
* ICP_NUM_CONFIRMATIONS - Number of Internet computer confirmations (20 by default).
* ICP_START_BLOCK - Internet Computer start block.
* ICP_URL - Internet computer URL.
* KEYMAKER_TRADE_ADDRESS - Address of `keymaker-trade` REST API ("http://localhost:27504" by default).
* LANDLORD_CANISTER_ID - Principal ID of landlord canister.
* MAX_TRADING_WALLET_VALUE_USD - Maximum USD value allowed for trading wallet (infinite by default).
* NIOBE_CANISTER_ID - Principal ID of niobe canister.
* REST_API_ADDRESS - Address to serve REST API ("http://localhost:27507" by default)
* TRADING_ENGINE_URL - Trading engine URL.
* SERAPH_DB_URL - Seraph database connection URL.

## new system design - featuring the V12 matching engine

![new helix system design](docs/images/helix-design-2.png)

### security design objectives

The questions we need to answer fall into the following areas:

  1. authentication

* end users (humans/bots) must authenticate towards the helix exchange
* requests to the REST/websocket APIs (made by end users or internal services) must have proper authentication
* internal services must verify the authentication of requests made by end users or other internal services

  2. authorization

* certain APIs may only be used by end users or internal service and vice versa
* end users may have different roles/ranks (e.g retail, market maker etc.) and the availability of APIs, rate limits etc. may be limited depending on these
* internal services must verify the authorization of requests made by end users or other internal services

  2. secrets management: we need to create, share, store, rotate

* service account credentials and/or roles
* (postgres) database encryption keys
* backup encryption keys

  3. encryption of data at rest

* sensitive postgres database columns
* backups

    must be encrypted

### threats

Which threats do we defend (confidentiality, integrity, availability) against?

* NO: attack by a three letter agency e.g. NSA
* NO: police raids the data center and executes [cold boot attacks](https://en.wikipedia.org/wiki/Cold_boot_attack) ([attack update](https://www.zdnet.com/article/new-cold-boot-attack-affects-nearly-all-modern-computers/))
* YES: malicious insiders e.g. sabotage or theft of secrets / funds
* YES: [advanced persistent threat](https://en.wikipedia.org/wiki/Advanced_persistent_threat) e.g. North-Korean hackers
* YES: catastrophic failures e.g.
  * large scale data corruption over a protracted period of time
  * loss of keys/credentials
  * disclosure of keys/credentials
* YES: unauthorized access to exchange infrastructure, whether physical or remote
* NO: `landlord` smart contract is DDOS'ed
* NO: dfinity/ICP subnet running the `landlord` smart contract loses/divulges the smart contract's private key
* NO: off-chain servers/infrastructure is DDOS'ed

### general principles / considerations

* all APIs (except for the market data API) will be authenticated
* backend services will use service accounts (key/secret) to authenticate to each other (example: `keymaker-fund` to `funding-proxy`)
* service account credentials (key/secret) will be rotated frequently (every N hours) (N=4h?)
* key material (or any other sensitive data) must be
  * encrypted at rest
  * zeroized immediately after use (to limit exposure / damage from memory dumps)
* we don't want to traverse the public internet in order to obtain key material or service account credentials
* REST APIs calls need to protect against replay attacks by using a timestamp (5 seconds in the past or shorter)
* we apply [defense in depth](https://en.wikipedia.org/wiki/Defense_in_depth_(computing)) to protect our systems
* every backend service will have its own database encryption key that is rotated every K hours (K=24h?)
* we want to use proven / open source tools if they exist and minimize building/maintaining such tools ourselves

**Please see** [docs/security-design.md](docs/security-design.md) for details.
