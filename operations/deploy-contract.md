# Deploying a contract to a ZVM devnet and verifying it in the explorer

A contract reaches a ZVM devnet as an ordinary EVM creation transaction. It is
published in the explorer by handing the node the source it was compiled from,
which the node recompiles and compares with the code at the address. Any deploy
route can be followed by any verify route.

## Where things are

| | |
|---|---|
| EVM chain id | **7340470** |
| JSON-RPC | `http://localhost:47080/zvm/rpc` |
| Verifier API | `http://localhost:47080/zvm/api` |
| Explorer | `http://localhost:47080/explorer/` |
| Faucet | `POST http://localhost:47080/zvm/api/faucet` with `{"address":"0x…"}`, or `http://localhost:47080/glyphswap/faucet`: 50 ZNN per drip, 10 s cooldown |

These are a local [zvm-devnet](https://github.com/Zenoglyphs-VM/zvm-devnet) at
its default address; another devnet has its own address and chain id, and a
script written for one will not reach the other. The devnet's test accounts
hold ZNN already, so the faucet is only for an address of your own.

### Check the endpoints first

Each of these was run against a fresh zvm-devnet on 2026-10-20, with the
answers shown. If yours differ, stop before spending anything.

```sh
export ZVM_RPC=http://localhost:47080/zvm/rpc
export ZVM_API=http://localhost:47080/zvm/api

rpc() { curl -s -X POST -H 'content-type: application/json' \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"$1\",\"params\":[${2:-}]}" $ZVM_RPC; }

rpc eth_chainId                  # "result":"0x7001b6"  = 7340470
rpc eth_gasPrice                 # "result":"0xee9f278" = 250,212,984 wei
rpc eth_maxPriorityFeePerGas     # "result":"0xf4240"   = 1,000,000 wei
```

The last two are why the fee section below matters: the node suggests a tip
fifty thousand times smaller than the relayer accepts.

The verifier answers on the same host, and WQSR at
`0x4200000000000000000000000000000000000101` is already verified, so it is
something to test against.

```sh
# The compiler versions the node offers (newest 0.8.37 when checked):
curl -s $ZVM_API/verify/compilers
# {"builds":[{"longVersion":"0.8.37+commit.f401782d","version":"0.8.37"},…

# A verified contract's ABI, in Etherscan's shape:
curl -s "$ZVM_API?module=contract&action=getabi&address=0x4200000000000000000000000000000000000101"
# {"message":"OK","result":"[{\"inputs\":[],\"name\":\"InsufficientAllowance\",…","status":"1"}

# A chain id that is not this one is refused:
curl -s "$ZVM_API?module=contract&action=getabi&chainid=1&address=0x4200000000000000000000000000000000000101"
# {"message":"NOTOK","result":"this explorer serves chain 7340470, not 1","status":"0"}

# Submitting source for an address that is already verified is refused:
curl -s -X POST $ZVM_API \
  --data-urlencode module=contract --data-urlencode action=verifysourcecode \
  --data-urlencode chainid=7340470 \
  --data-urlencode contractaddress=0x4200000000000000000000000000000000000101 \
  --data-urlencode codeformat=solidity-single-file --data-urlencode 'sourceCode=contract X{}' \
  --data-urlencode contractname=X --data-urlencode compilerversion=v0.8.30+commit.73712a01
# {"message":"NOTOK","result":"Contract source code already verified","status":"0"}
```

The create2 deployer at `0x4e59b44847b379578588920ca78fbf26c0b4956c` has code on
this chain, so a deploy through it gets the same address here as on any other
chain that has it. A contract created that way is created by a call to the
deployer rather than by a creation transaction of yours, so the node probably
cannot read its constructor arguments and you should pass them to the verifier
explicitly (not tested).

## Before deploying

**Fund the deployer** from the faucet. Admission checks the sender can afford
gas limit × max fee, not what the deploy will actually spend, so a generous fee
cap needs a larger balance than the deploy costs. At a 100 gwei tip, a
1,000,000-gas deploy burns about 0.1 ZNN.

**Set the priority fee yourself, to at least 50 gwei (100 is safe).** This is
the one thing that trips stock tooling. The relayer that carries transactions
refuses any whose effective tip is under 50 gwei, but the node's
`eth_maxPriorityFeePerGas` answers 0.001 gwei, so a tool or wallet that asks
the node for a fee signs a transaction that is then refused:

```
priority fee … is below this relayer's minimum of 50000000000 wei per gas
```

Leave headroom in the max fee as well: the effective tip is
`min(tip, maxFee − baseFee)`, so a cap that only just clears the floor loses its
tip as the base fee rises. A max fee of 150 gwei is comfortable.

**Compile with `evmVersion: prague`.** Contracts on this chain are built for
Prague, and the compiler's default target changes from release to release, so
set it explicitly. A single transaction cannot use more than 16,777,216 gas, so
a creation that needs more is refused.

**Keep solc's standard JSON input** from the build that produced the bytecode.
Verification is a recompile, and this input carries every file and setting, so it
is the one verify route that cannot disagree with your compiler about them.
Foundry, Hardhat and Remix all produce it.

## Deploy

None of the commands below have been run against the chain; they follow from the
fee rules above and the endpoint checks. Nothing has been deployed or verified by
hand for this page.

### Foundry

```sh
export ZVM_RPC=http://localhost:47080/zvm/rpc
export ZVM_API=http://localhost:47080/zvm/api
```

`foundry.toml`:

```toml
[profile.default]
evm_version = "prague"
optimizer = true
optimizer_runs = 200
```

One contract, verified in the same command:

```sh
forge create src/Token.sol:Token \
  --rpc-url $ZVM_RPC --private-key $KEY --broadcast \
  --gas-price 150000000000 --priority-gas-price 100000000000 \
  --constructor-args "Name" "SYM" \
  --verify --verifier etherscan --verifier-url $ZVM_API \
  --etherscan-api-key zvm --chain 7340470
```

A script:

```sh
forge script script/Deploy.s.sol \
  --rpc-url $ZVM_RPC --private-key $KEY --broadcast --slow \
  --with-gas-price 150000000000 --priority-gas-price 100000000000 \
  --verify --verifier etherscan --verifier-url $ZVM_API \
  --etherscan-api-key zvm --chain 7340470
```

`--gas-price` is the max fee for an EIP-1559 transaction. `--slow` waits for each
transaction's receipt before sending the next; a run of back-to-back nonces has
been dropped by the relayer as "nonce too high" or "nonce too low", and
one at a time avoids that. The API key is not checked, but forge wants a value.

### Hardhat

Hardhat 2 has no priority-fee setting, so set `gasPrice`, which sends legacy
transactions; a legacy transaction's tip is `gasPrice − baseFee`, so 150 gwei
clears the floor.

```js
solidity: {version: '0.8.30', settings: {evmVersion: 'prague', optimizer: {enabled: true, runs: 200}}},
networks: {
  zvm: {
    url: 'http://localhost:47080/zvm/rpc',
    chainId: 7340470,
    accounts: [process.env.KEY],
    gasPrice: 150_000_000_000,
  },
},
```

### Remix or a wallet

Add the chain to the wallet (chain id 7340470, the RPC above, ZNN as the coin),
choose *Injected Provider* in Remix and deploy. Before confirming, open the
wallet's advanced fee editor and set the priority fee to at least 50 gwei;
left alone, it offers 0.001 gwei and the transaction is refused.

### `zvm evm-send`

The `zvm` CLI signs an EIP-1559 transaction and submits it as a wallet would,
with a 100 gwei tip and a suitable fee cap already set. With no `-to`, it deploys
`-data` as creation code:

```sh
# the devnet's rollup config, from a zvm-devnet checkout
docker compose cp genesis:/l2/rollup.json .
zvm evm-send -config rollup.json -eth $ZVM_RPC \
  -key 0x… -data @Token.creation.hex
```

`-data @file` reads hex: the creation bytecode with the ABI-encoded constructor
arguments appended. It waits for the receipt and prints
`contract    0x… = z1…`.

## Verify

The verifier needs four things: the address, the solc version, the contract as
`path:Name` (the path being a key in the input's `sources`), and the standard
JSON input. It accepts the submission only if the runtime bytecode it compiles
equals the code at the address. Immutables and linked library addresses are
taken from the chain, so they need no attention. A match on every byte,
including the metadata hash, is **exact**; everything but the metadata hash is
**partial**.

### The explorer page

Open `http://localhost:47080/explorer/verify`, or `/verify/<address>` to arrive
with the address filled in.

- **Standard JSON input** is the reliable mode. It accepts plain standard JSON,
  a Hardhat build-info file, and Remix's `compData.json` export, as a file, a
  paste, or a drop anywhere on the form. Foundry's is printed by
  `forge verify-contract --show-standard-json-input <address> <path:Name>`.
- **Solidity files** takes the `.sol` files, imports included and named as the
  imports name them, and builds the input from an optimizer, runs, EVM-version
  and via-IR form. Set EVM version to `prague`: the default, "the compiler's
  default", is not the same for every compiler.
- **Compiler** is chosen from the pragma when there is one; check it against
  the version you compiled with.
- **Contract** is `path:Name`, offered from the contracts the sources declare.
- **Constructor arguments** are read from the creating transaction. Paste the
  ABI-encoded hex only if the node cannot find it, such as when a factory
  created the contract.

Submitting starts a job that the page polls: queued, compiling, then verified or
failed. A compiler version the node has not used before is downloaded first. On
success the contract page shows a *Verified* badge and Read and Write tabs
generated from the ABI.

### Foundry

```sh
forge verify-contract 0x… src/Token.sol:Token \
  --chain 7340470 --verifier etherscan --verifier-url $ZVM_API \
  --etherscan-api-key zvm --watch
```

Add `--constructor-args $(cast abi-encode "constructor(string,string)" "Name" "SYM")`
if the contract was not created by an ordinary transaction. The `--verify` flags
on `forge create` and `forge script` above make this call after deploying.

### Hardhat

```js
etherscan: {
  apiKey: {zvm: 'zvm'},
  customChains: [{
    network: 'zvm',
    chainId: 7340470,
    urls: {
      apiURL: 'http://localhost:47080/zvm/api',
      browserURL: 'http://localhost:47080/explorer',
    },
  }],
},
```

then `npx hardhat verify --network zvm <address> <constructor args>`.

### Directly over HTTP

The simplest route to script:

```sh
curl -s -X POST $ZVM_API/verify -H 'content-type: application/json' -d '{
  "address": "0x…",
  "compiler": "0.8.30+commit.73712a01",
  "contract": "src/Token.sol:Token",
  "input": { "language": "Solidity", "sources": {…}, "settings": {…} }
}'
# {"id":"…","status":"queued",…}

curl -s $ZVM_API/verify/<id>
# status: queued | running | verified | failed; when verified, match: exact | partial
```

`constructorArgs` (hex, without the creation code) is optional. A leading `v` on
the compiler is accepted. `GET $ZVM_API/verify/compilers` lists the versions the
node offers. Resubmitting an address that is already verified is refused with
"already verified".

The same node also answers Etherscan's contract API at `$ZVM_API` (`/api`), which
is what forge and hardhat-verify call, so any other tool that can point an
Etherscan verifier at a custom URL works with `--verifier-url $ZVM_API`. It
refuses a `chainid` other than the devnet's own.

Verification has limits: sources must be inline (no `urls`, no absolute
imports), the node accepts about sixty submissions a minute across all users, and
a compile that takes more than a few minutes fails. A script verifying many
contracts should retry when told the queue is full.

## When it does not work

| You see | Cause | Do |
|---|---|---|
| `priority fee … is below this relayer's minimum` | the tip is under 50 gwei | set it to 50 gwei or more; 100 is safe |
| `insufficient funds: balance …, transaction costs up to …` | admission checks gas limit × max fee | fund more, or lower the max fee |
| `max fee per gas … is below the base fee` | the cap has not kept up with a busy block | raise the max fee |
| a tx hash, then no receipt | the transaction was dropped after being relayed | send it again; with a script, use `--slow` |
| a creation over 16,777,216 gas is refused | the per-transaction gas cap | split the contract or lower its constructor cost |
| verify: bytecode does not match | solc version, optimizer, runs, EVM version or via-IR differ from the build | submit the standard JSON input from the build itself |
| verify: `already verified` | the address already has a verification | nothing; open the contract page |
| verify: queue full or too many requests | the node's submission limits | retry in a minute |
