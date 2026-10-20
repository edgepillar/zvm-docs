# Who pays for what on NoM

**Written 2026-09-16.** The plasma fairness audit
[D8](../spec/DECISIONS.md#d8--chunks-belong-to-their-author-and-expire) asks for:
every message type and every block a ZVM service publishes, what it costs, who
pays it, who benefits — and anything where those last two are not the same
account.

Plasma is NoM's rate limit rather than a fee: it is rented with fused QSR,
costs nothing to use, and is returned when the QSR is unfused. So "who pays"
means "whose fused QSR is occupied, and whose account is rate-limited", which
is what decides who can be starved and by whom.

A send costs `21,000 + 68 × bytes` of base plasma. Under dynamic plasma a block
must actually pay `ceil(base × fusionPrice ÷ 1000)`, so every number below is a
v1 figure and a floor for v2.

---

## Every block the ZVM puts on NoM

| Block | Bytes | Base plasma | Published by | Who benefits | Same account? |
|---|---:|---:|---|---|---|
| **`0x01` native call** (ERC-20 transfer) | 89 | 27,052 | the caller | the caller | **yes** |
| **`0x02` relayed batch** (full block) | 16,384 | 1,135,112 | the relayer | the wallets inside it | **no** |
| **`0x03` chunk** | ≤ 16,384 | ≤ 1,135,112 | the author | the author | **yes** |
| **`0x04` assemble** (2 chunks) | 69 | 25,692 | the author | the author | **yes** |
| **Deposit**, a send to the vault | 0–90 | 21,000+ | the depositor | the depositor | **yes** |
| **Vault receive**, one per deposit | 0 | 21,000 | **the committee** | the depositor | **no** |
| **Payout**, `ZVMW ‖ id` | 36 | 23,448 | **the committee** | the withdrawer | **no** |
| **Burn mirror**, `ZVMB` | 4 | 21,272 | **the committee** | every holder | **no** |
| **Faucet** sends and fusions | — | — | the faucet | the recipient | no, and deliberately |

Two structural facts make the list shorter than it looks:

- **Nothing receives an inbox message.** The inbox's type byte is one no key
  and no embedded contract can produce, so no receive block is ever published
  for a `0x01`–`0x04` send and nobody pays receive plasma for one. This is the
  same property that makes value sent there burned, and it is why the message
  lane costs exactly one block.
- **Every ZVM block is a *user send*.** Derivation ignores contract blocks, so
  the ZVM never causes an embedded-contract call and never pays embedded
  plasma.

---

## The four places the payer is not the beneficiary

### 1. The batch lane — the relayer pays, the wallets ride

A relayer spends up to 1,135,112 plasma on a block carrying other people's
transactions, and those transactions pay it nothing on NoM. At the measured 89
compressed bytes per transfer that is about 6,100 plasma of someone else's
capital per transaction.

**This is the subsidy the whole design rests on**, and it is the one that now
has an answer: from the gas fork a relayed transaction's priority fee is split,
half to the account that posted it and half burned
([D9](../spec/DECISIONS.md#d9--a-native-calls-gas-is-the-plasma-its-nom-block-paid-times-16),
`contracts/src/ZVMFees.sol`). The relayer is paid on the ZVM for the plasma it
spends on NoM. It is not a refund in the same asset, and it does not have to
be: what a relayer needs is a reason to keep running, not an exact rebate.

Until that fork activates on a chain, the relayer is a volunteer. That is
D20's position and it is fine for a devnet; it is not fine for a chain anyone
depends on.

**Under dynamic plasma that 1,135,112 is a floor, not a price** (measured
2026-09-16, chain 71). A block pays `ceil(base × nextFusionPrice / 1000)`, and
the rollup's own traffic is what moves that price: driven by the benchmark it
went from 1× to **23.6×**, so the same full block wanted 26.8 M plasma. An
account cannot hold more than **10.5 M** however much it fuses, so past about
**9.25×** a full 16 KB block cannot be published by anyone, and the relayer's
accounts start being refused outright:

```
err="ledger.publishRawTransaction: rpc error -32000: not enough plasma on account"
```

Three consequences worth stating plainly:

- **The subsidy is not a fixed cost.** A relayer budgeting fused QSR against the
  base price is budgeting for the quiet case; the busy case, which is the one
  that matters, can cost twenty times more per block.
- **Capacity is bought with accounts, not QSR.** The 10.5 M ceiling is per
  account, so the only way to publish more under a high price is more accounts —
  each of which is more fused QSR standing idle when the chain is quiet.
- **It recovers slowly.** The price decays about 5 % per momentum, so a burst
  that pushes it to 23× leaves the chain charging a multiple for roughly
  fourteen minutes after the load stops.

None of this is a defect in dynamic plasma — it is dynamic plasma working, on a
user whose whole product is data. It is an argument about who should hold the
fused QSR, and how much.

### 2. The vault's receive — the committee pays per deposit

`bridge.receiveDeposits` publishes a receive for every send into the vault, so
the deposit can be spent later. That is ~21,000 plasma per deposit, paid by the
committee, benefiting the depositor.

It is small per deposit and unbounded in aggregate: **the peg's NoM running
cost scales with the number of deposits, and nothing charges for it.** A
deposit of one ZNN and a deposit of a million cost the committee the same. At
the v1 ceiling of 100 account blocks a momentum, a sustained flood of dust
deposits would occupy the vault's plasma and delay payouts behind them — not
theft, but a denial of service against the peg, paid for by the attacker at
21,000 plasma a block and by the committee at the same rate.

**The cheap mitigation is built (2026-09-16).** `zvm bridge -min-receive
ZNN:1,QSR:10` sets a floor per token, below which a deposit is not received
promptly; dust is received as soon as no withdrawal is outstanding, and
`/status` reports `deferredDeposits`. It is off by default.

Deferring is not refusing, and the distinction is the whole reason this is
safe: **the ZVM credits a deposit from the send**, which every node derives
independently, so the depositor has their balance whether or not the vault has
received anything. An unreceived send simply sits on NoM. What waits is only the
vault's ability to re-spend those coins — which is exactly the resource a dust
flood was competing for, so the floor points the delay back at the party causing
it.

**The honest fix is built (2026-09-16): the bridge now has revenue, and spends
it on exactly this.** A withdrawal pays a fee — 25 basis points, at least 0.01
ZNN or 0.1 QSR — which the vault keeps instead of paying out. That is the shape
Zenon's own bridge uses: `FeePercentage` per token pair, taken on a wrap and
accumulated in the contract. Here it accumulates in `ZVMBridge` as `feesZNN`
and `feesQSR`, and `zvm bridge -fund-plasma` fuses the QSR part to the vault's
own plasma, which is the resource receives and payouts spend.

Three details decide whether that is safe:

- **What may be fused is ZVM state, not a balance.** `fundableQSR()` is
  `feesQSR − fusedQSR`: fees earned less fees already fused. Reading the
  vault's NoM balance instead would race — a deposit the vault has received but
  the ZVM has not yet credited looks exactly like surplus — so the balance is
  only checked for whether the QSR is actually there.
- **The fusion is derived back.** A vault send to NoM's plasma contract derives
  to `markFused` (spec §7), so `fusedQSR` is a fact every node computes rather
  than a number the committee reports.
- **A committee member can check it.** The signer policy accepts a third shape
  of block — `plasma.Fuse(vault)`, in QSR, for no more than fee revenue allows —
  and works that limit out from its own nodes. Fused QSR stays the vault's, so
  this moves no value out; it is bounded so that a coordinator cannot lock up
  the QSR payouts need.

The floor (§2, above) stays: it bounds the damage while the revenue accrues.
What has changed is that the cost is now paid by the people the peg is for,
rather than by whoever runs the committee.

### 3. Payouts — the committee pays, the withdrawer benefits

23,448 plasma per withdrawal, on the committee. There is no withdrawal fee on
the ZVM side, so this is unpriced too. It is bounded by withdrawal volume
rather than by deposit dust, and the size tiers and outflow limits
([D17](../spec/DECISIONS.md#d17--withdrawals-wait-six-momentums-on-devnet-thirty-on-testnet-and-mainnet),
`docs/COMMITTEE.md`) already bound how fast it can be made to happen.

### 4. The burn mirror — the committee pays for the chain's honesty

21,272 plasma each time the committee destroys, on NoM, ZNN the fee splitter
burned on the ZVM. Nobody in particular benefits; every holder does, because it
is what makes the burn visible in NoM's own supply rather than only in ZVM
state. It is periodic and cheap, and the committee is the only account that
*can* publish it, since only the vault holds the backing ZNN.

---

## What was unfair and is not any more

**Chunks bought nothing.** Before the gas fork, a call that arrived in chunks
was priced by the assemble block's plasma alone: an author paid 1,135,112
plasma for each 16 KB chunk and got gas for 25,692. A 22 KB contract deployment
paid for two full blocks and was budgeted as if it were one small one — which
is why deploying from a Zenon account did not work without a clone factory.

From the gas fork the whole payload's plasma counts, each chunk once
(`spec/SPEC.md` §6). The author now gets what the author paid for, and the
double-spend that rule has to prevent — publish chunks once, assemble twice —
is prevented by consuming each chunk on its first assembly.

---

## The conclusion

Nothing rides completely free: every ZVM block on NoM is paid for by somebody's
fused QSR, and there is no path by which a user gets a NoM block without an
account that paid its plasma.

But **three costs sit on the committee rather than on the accounts that cause
them** — receives, payouts and the burn mirror — and one sits on the relayer.
The relayer's is now priced. The committee's are not, and of those the receive
is the one that scales with something an attacker chooses rather than with
something the committee chooses.

| Cost | Borne by | Scales with | Priced? |
|---|---|---|---|
| Batch blocks | the relayer | relayed volume | **yes**, from the gas fork |
| Vault receives | the committee | **deposit count** | **yes**, from the withdrawal fee — and floored |
| Payouts | the committee | withdrawal count | **yes**, from the withdrawal fee; rate-limited too |
| Burn mirror | the committee | how often it is run | no, and negligible |
| Plasma mirror | the committee | how often the fee is fused | no: it is what the fee pays for |

That is the audit. The one item worth acting on before a chain holds value was
the deposit receive, because it is the only unpriced cost an outsider can run up
on somebody else's account. It now has a floor (§2) *and* a source of funds: a
withdrawal pays for the plasma that receives deposits and publishes payouts.

The remaining asymmetry is deliberate and worth naming: **depositors do not pay,
withdrawers do.** A deposit costs the committee a receive and pays nothing; the
fee is collected on the way out. That is the trade every fee schedule of this
shape makes — it is cheap to arrive and it costs something to leave — and it
holds as long as most deposits eventually leave. A flood of dust that never
withdraws still pays nothing, which is what the floor is for.
