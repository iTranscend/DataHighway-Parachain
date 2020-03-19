# Data Highway [![GitHub license](https://img.shields.io/github/license/paritytech/substrate)](LICENSE) <a href="https://github.com/DataHighway-DHX/node/actions?query=workflow%3ACI+branch%3Adevelop" target="_blank"><img src="https://github.com/DataHighway-DHX/node/workflows/CI/badge.svg?branch=develop"></a>

The Data Highway Substrate-based blockchain node.

__WARNING__: This implementation is a proof-of-concept prototype and is not ready for production use.

# Table of contents

* [Contributing](#chapter-cb8b82)
* [Documentation](#chapter-888ccd)
* [Example "dev" development PoS testnet with single nodes](#chapter-ca9336)
* [Example "local" PoS testnet with multiple nodes](#chapter-f21efd)
* [Live (Alpha) "testnet-latest" PoS testnet (with multiple nodes)](#chapter-f0264f)
* [Interact with blockchain using Polkadot.js Apps UI](#chapter-6d9058)

Note: Generate a new chapter with `openssl rand -hex 3`

## Documentation <a id="chapter-888ccd"></a>

Refer to [CONTRIBUTING.md](./CONTRIBUTING.md) for contributing instructions.

Relevant part of contributing instructons will progressively be moved to [DataHighway Developer Hub](https://github.com/DataHighway-DHX/documentation).

## Example "dev" development PoS testnet (with single node) <a id="chapter-f21efd"></a>

### Intro

The development testnet only requires a single node to produce and finalize blocks.

### Run on Local Machine

* Fork and clone the repository

* Install or update Rust and dependencies. Build the WebAssembly binary from all code

```bash
curl https://getsubstrate.io -sSf | bash -s -- --fast && \
./scripts/init.sh && \
cargo build --release
```

* Purge the chain (remove relevant existing blockchain testnet database blocks and keys)

```bash
./target/release/datahighway purge-chain --dev --base-path /tmp/polkadot-chains/alice
```

* Connect to development testnet (`--chain development` is abbreviated `--dev`)

```bash
./target/release/datahighway \
  --base-path /tmp/polkadot-chains/alice \
  --name "Data Highway Development Chain" \
  --dev \
  --telemetry-url ws://telemetry.polkadot.io:1024
```

## Example "local" PoS testnet (with multiple nodes) <a id="chapter-f21efd"></a>

### Intro

Run a multiple node PoS testnet on your local machine with built-in keys (Alice, Bob, Charlie) using a custom Substrate-based blockchain configuration using multiple terminals windows.
* Configure and import custom raw chain definition
* Use default accounts Alice, Bob, and Charlie as the three initial authorities of the genesis configuration that have been endowed with testnet units that will run validator nodes
* **Important**: Since we are using GRANDPA where you have authority set of size 4, it means you need 3 nodes running in order to **finalize** the blocks that are authored. (Credit: @bkchr Bastian Köcher)

### Run on Local Machine (without Docker)

#### Fetch repository and dependencies

* Fork and clone the repository

* Install or update Rust and dependencies. Build the WebAssembly binary from all code

```bash
curl https://getsubstrate.io -sSf | bash -s -- --fast && \
./scripts/init.sh
```

#### Build runtime code

```bash
cargo build --release
```

#### Create custom blockchain configuration

* Create latest chain specification code changes of <CHAIN_ID> "local"

> Other chains are specified in src/chain_spec.rs (i.e. dev, local, testnet, or testnet-latest).

* Generate the chain specification JSON file from src/chain_spec.rs

```bash
mkdir -p ./src/chain-spec-templates
./target/release/datahighway build-spec \
  --chain=local > ./src/chain-spec-templates/chain_spec_local_latest.json
```

* Build "raw" chain definition for the new chain from it

```bash
mkdir -p ./src/chain-definition-custom
./target/release/datahighway build-spec \
  --chain ./src/chain-spec-templates/chain_spec_local_latest.json \
  --raw > ./src/chain-definition-custom/chain_def_local_v0.1.0.json
```

> Remember to purge the chain state if you change anything (database and keys)

```bash
./target/release/datahighway purge-chain --chain "local" --base-path /tmp/polkadot-chains/alice
./target/release/datahighway purge-chain --chain "local" --base-path /tmp/polkadot-chains/bob
./target/release/datahighway purge-chain --chain "local" --base-path /tmp/polkadot-chains/charlie
```

#### Terminal 1

Run Alice's bootnode using the raw chain definition file that was generated

```bash
./target/release/datahighway --validator \
  --unsafe-ws-external \
  --unsafe-rpc-external \
  --rpc-cors=all \
  --base-path /tmp/polkadot-chains/alice \
  --keystore-path "/tmp/polkadot-chains/alice/keys" \
  --chain ./src/chain-definition-custom/chain_def_local_v0.1.0.json \
  --node-key 88dc3417d5058ec4b4503e0c12ea1a0a89be200fe98922423d4334014fa6b0ee \
  --alice \
  --rpc-port 9933 \
  --port 30333 \
  --telemetry-url ws://telemetry.polkadot.io:1024 \
  --ws-port 9944 \
  --execution=native \
  -lruntime=debug
```

When the node has started, copy the libp2p local node identity of the node, and paste in the `bootNodes` of chain_def_local_v0.1.0.json if necessary.

* Notes:
  * Alice's Substrate-based node on default TCP port 30333
  * Her chain database stored locally at `/tmp/polkadot-chains/alice`
  * Bootnode ID of her node is `Local node identity is: QmWYmZrHFPkgX8PgMgUpHJsK6Q6vWbeVXrKhciunJdRvKZ` (peer id), which is generated from the `--node-key` value specified below and shown when the node is running. Note that `--alice` provides Alice's session key that is shown when you run `subkey -e inspect //Alice`, alternatively you could provide the private key that is necessary to produce blocks with `--key "bottom drive obey lake curtain smoke basket hold race lonely fit walk//Alice"`. In production the session keys are provided to the node using RPC calls `author_insertKey` and `author_rotateKeys`. If you explicitly specify a `--node-key` (i.e. `--node-key 88dc3417d5058ec4b4503e0c12ea1a0a89be200fe98922423d4334014fa6b0ee`) when you start your validator node, the logs will still display your peer id with `Local node identity is: Qxxxxxx`, and you could then include it in the chain_spec_local_latest.json file under "bootNodes". Also the peer id is listed when you go to view the list of full nodes and authority nodes at Polkadot.js Apps https://polkadot.js.org/apps/#/explorer/datahighway

#### Terminal 2

Run Bob's Substrate-based node on a different TCP port of 30334, and with his chain database stored locally at `/tmp/polkadot-chains/bob`. We'll specify a value for the `--bootnodes` option that will connect his node to Alice's bootnode ID on TCP port 30333:

```bash
./target/release/datahighway --validator \
  --unsafe-ws-external \
  --unsafe-rpc-external \
  --rpc-cors=all \
  --base-path /tmp/polkadot-chains/bob \
  --keystore-path "/tmp/polkadot-chains/bob/keys" \
  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/QmWYmZrHFPkgX8PgMgUpHJsK6Q6vWbeVXrKhciunJdRvKZ \
  --chain ./src/chain-definition-custom/chain_def_local_v0.1.0.json \
  --bob \
  --rpc-port 9933 \
  --port 30334 \
  --telemetry-url ws://telemetry.polkadot.io:1024 \
  --ws-port 9944 \
  --execution=native \
  -lruntime=debug
```

> Important: Since in GRANDPA you have authority set of size 4, it means you need 3 nodes running in order to **finalize** the blocks that are authored. (Credit: @bkchr Bastian Köcher)

#### Terminal 3

Run Charlie's Substrate-based node on a different TCP port of 30335, and with his chain database stored locally at `/tmp/polkadot-chains/charlie`. We'll specify a value for the `--bootnodes` option that will connect his node to Alice's bootnode ID on TCP port 30333:

```bash
./target/release/datahighway --validator \
  --unsafe-ws-external \
  --unsafe-rpc-external \
  --rpc-cors=all \
  --base-path /tmp/polkadot-chains/charlie \
  --keystore-path "/tmp/polkadot-chains/charlie/keys" \
  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/QmWYmZrHFPkgX8PgMgUpHJsK6Q6vWbeVXrKhciunJdRvKZ \
  --chain ./src/chain-definition-custom/chain_def_local_v0.1.0.json \
  --charlie \
  --rpc-port 9933 \
  --port 30335 \
  --telemetry-url ws://telemetry.polkadot.io:1024 \
  --ws-port 9944 \
  --execution=native \
  -lruntime=debug
```

* Check that the chain is finalizing blocks (i.e. finalized is non-zero `main-tokio- INFO substrate  Idle (2 peers), best: #3 (0xaede…b8d9), finalized #1 (0x4c69…f605), ⬇ 3.3kiB/s ⬆ 3.7kiB/s`)

* Generate session keys for Alice
```bash
$ subkey --ed25519 inspect "//Alice"
Secret Key URI `//Alice` is account:
  Secret seed:      0xabf8e5bdbe30c65656c0a3cbd181ff8a56294a69dfedd27982aace4a76909115
  Public key (hex): 0x88dc3417d5058ec4b4503e0c12ea1a0a89be200fe98922423d4334014fa6b0ee
  Account ID:       0x88dc3417d5058ec4b4503e0c12ea1a0a89be200fe98922423d4334014fa6b0ee
  SS58 Address:     5FA9nQDVg267DEd8m1ZypXLBnvN7SFxYwV7ndqSYGiN9TTpu

$ subkey --sr25519 inspect "//Alice"//aura
Secret Key URI `//Alice//aura` is account:
  Secret seed:      0x153d8db5f7ef35f18a456c049d6f6e2c723d6c18d1f9f6c9fbee880c2a171c73
  Public key (hex): 0x408f99b525d90cce76288245cb975771282c2cefa89d693b9da2cdbed6cd9152
  Account ID:       0x408f99b525d90cce76288245cb975771282c2cefa89d693b9da2cdbed6cd9152
  SS58 Address:     5DXMabRsSpaMwfNivWjWEnzYtiHsKwQnP4aAKB85429ZQU6v

$ subkey --sr25519 inspect "//Alice"//babe
Secret Key URI `//Alice//babe` is account:
  Secret seed:      0x7bc0e13f128f3f3274e407de23057efe043c2e12d8ed72dc5c627975755c9620
  Public key (hex): 0x46ffa3a808850b2ad55732e958e781146ed1e6436ffb83290e0cb810aacf5070
  Account ID:       0x46ffa3a808850b2ad55732e958e781146ed1e6436ffb83290e0cb810aacf5070
  SS58 Address:     5Dfo9eF9C7Lu5Vbc8LbaMXi1Us2yi5VGTTA7radKoxb7M9HT

$ subkey --sr25519 inspect "//Alice"//imonline
Secret Key URI `//Alice//imonline` is account:
  Secret seed:      0xf54dc00d41d0ea7929ac00a08ed1e111eb8c35d669b011c649cea23997f5d8d9
  Public key (hex): 0xee725cf87fa2d6f264f26d7d8b84b1054d2182cdcce51fdea95ec868be9d1e17
  Account ID:       0xee725cf87fa2d6f264f26d7d8b84b1054d2182cdcce51fdea95ec868be9d1e17
  SS58 Address:     5HTME6o2DqEuoNCxE5263j2dNzFGxspeP8wswenPA3WerfmA

$ subkey --ed25519 inspect "//Alice"//grandpa
Secret Key URI `//Alice//grandpa` is account:
  Secret seed:      0x03bee0237d4847732404fde7539e356da44bce9cd69f26f869883419371a78ab
  Public key (hex): 0x6e2de2e5087b56ed2370359574f479d7e5da1973e17ca1b55882c4773f154d2f
  Account ID:       0x6e2de2e5087b56ed2370359574f479d7e5da1973e17ca1b55882c4773f154d2f
  SS58 Address:     5EZAkmxARDqRz5z5ojuTjacTs2rTd7WRL1A9ZeLvwgq2STA2
```

#### Terminal 4 (Optional)

* Add session keys for account(s) to be configured as authorities (validators). Run cURL to insert session key for each key type (i.e. "aura"), by providing the associated secret key, and associated Public key (hex) 
```bash
curl -vH 'Content-Type: application/json' --data '{ "jsonrpc":"2.0", "method":"author_insertKey", "params":["aura", "", "0x408f99b525d90cce76288245cb975771282c2cefa89d693b9da2cdbed6cd9152"],"id":1 }' 127.0.0.1:9933
curl -vH 'Content-Type: application/json' --data '{ "jsonrpc":"2.0", "method":"author_insertKey", "params":["babe", "//Alice//babe", "0x46ffa3a808850b2ad55732e958e781146ed1e6436ffb83290e0cb810aacf5070"],"id":1 }' 127.0.0.1:9933
curl -vH 'Content-Type: application/json' --data '{ "jsonrpc":"2.0", "method":"author_insertKey", "params":["imon", "//Alice//imonline", "0xee725cf87fa2d6f264f26d7d8b84b1054d2182cdcce51fdea95ec868be9d1e17"],"id":1 }' 127.0.0.1:9933
curl -vH 'Content-Type: application/json' --data '{ "jsonrpc":"2.0", "method":"author_insertKey", "params":["gran", "//Alice//grandpa", "0x6e2de2e5087b56ed2370359574f479d7e5da1973e17ca1b55882c4773f154d2f"],"id":1 }' 127.0.0.1:9933
```

* Check that the output from each cURL request is `{"jsonrpc":"2.0","result":null,"id":1}`, since with a successful output `null` is returned https://github.com/paritytech/substrate/blob/db1ab7d18fbe7876cdea43bbf30f147ddd263f94/client/rpc-api/src/author/mod.rs#L47. Also check that the following folder is not empty /tmp/polkadot-chains/alice/keys (it should now contain four keys).

#### Additional Steps (Optional)

* Follow the steps to [interact with blockchain using Polkadot.js Apps UI](#chapter-6d9058)

* View on [Polkadot Telemetry](https://telemetry.polkadot.io/#list/DataHighway%20Local%20PoA%20Testnet%20v0.1.0)

* Distribute the custom chain definition (i.e. chain_def_local_v0.1.0.json) to allow others to synchronise and validate if they are an authority

## Testnet (Alpha) "testnet-latest" PoS testnet (with multiple nodes) <a id="chapter-f0264f"></a>

### Intro

Join the multiple node PoS testnet (alpha), where you will be using the latest custom chain definition for the testnet (i.e. chain_def_testnet_v0.1.0.json).

### Run (with Docker containers)

#### Fetch repository and dependencies

* Fork and clone the repository
* Install and run Docker
* Replace docker-compose.yml with your node information
* Update the relevant ./scripts/docker-entrypoint-<VALIDATOR_NAME>.sh with your node specific information
* Update the ["testnet-latest" chain spec](./src/chain_spec.rs), to be used to generate the raw chain definition
* Start the container (the image will be built on first run based on Dockerfile). It will install dependencies and build chain runtime code
  ```bash
  docker-compose --verbose up -d
  ```
* Check the logs
  ```bash
  docker-compose logs (-f to follow)
  ```
  * Screenshot:
  ![](./assets/images/logs.png)

* Follow the steps to [interact with blockchain using Polkadot.js Apps UI](#chapter-6d9058)

## Interact with blockchain using Polkadot.js Apps UI <a id="chapter-6d9058"></a>

* Setup connection between the UI and the node:
  * Go to Polkadot.js Apps at https://polkadot.js.org/apps
	* Click "Settings" from the sidebar menu, and click its "Developer" tab to be taken to https://polkadot.js.org/apps/#/settings/developer to add Custom Types. Copy the contents of [custom_types.json](./custom_types.json), and pasting it into the input field, then click "Save".
  * Click "Settings" from the sidebar menu again, and click its "General" tab to be taken to https://polkadot.js.org/apps/#/settings. Click the "remote node/endpoint to connect to" selection box, and choose "Local Node (127.0.0.1:9944)" option from the list, then click "Save".
  * Wait for the UI to refresh (i.e. additional sidebar menu items will appear including "Explorer", "Accounts", "Address book", "Staking", "Chain state", etc).
  * Click "Explore" from the sidebar menu to be taken to https://polkadot.js.org/apps/#/explorer/node and shown the "Node info", including connected peers.

Once you've established a connection between the UI and the DataHighway testnet, you may try the following:

* Create accounts and transfer funds:
  * Click "Accounts" from the sidebar menu, then click tab "My accounts", and click button "Add Account"
  * Import Bob's built-in stash account (with 1,000 DHX balance) from the [test keyring](https://github.com/polkadot-js/apps/issues/1117#issuecomment-491020187) by entering: 
    * name: "Bob"
    * mnemonic seed: "bottom drive obey lake curtain smoke basket hold race lonely fit walk"
    * password: "bob"
    * password (repeat): "bob"  
    * secret derivation path: "//Bob//stash"
* Transfer funds between accounts:
  * Click "Transfer" from the sidebar menu
* Stake on the testnet (using testnet DHX that has been endowed to accounts)
  * Click "Stake" from the sidebar menu. Refer to the [Polkadot wiki's collator, validator, and nominator guides](https://wiki.polkadot.network/docs/en/maintain-guides-how-to-validate-kusama)
* Chain state interaction (i.e. roaming, mining, etc):
  * Click "Chain state" from the sidebar menu, then click tab "Storage"
  * Click "selected state query" selection box, and then as an example, choose "dataHighwayMiningSpeedBoostLodgement", and see if it works yet (WIP).
* Extrinsics interaction (i.e. roaming, mining, etc):
  * Click "Extrinsics" from the sidebar menu.

* **Important**:
  * Input parameter quirk: Sometimes it is necessary to modify the value of one of the input parameters to allow you to click "Submit Transaction" (i.e. if the first arguments input value is already 0 and appears valid, but the "Submit Transaction" button appears disabled, just delete the 0 value and re-enter 0 again)
  * Prior to being able to submit extrinics at https://polkadot.js.org/apps/#/extrinsics (i.e. roaming > createNetwork()) or to view StorageMap values, it is necessary to Add Custom Types (see earlier step), otherwise the "Submit Transaction" button will not work.
