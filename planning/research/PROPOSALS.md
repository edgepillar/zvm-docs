# Review register for the 24 future ideas

**All entries: `REQUIREMENTS_ONLY`.** F01-F24 preserve the numbering in the
selected FUTURE.md. Every entry inherits R1-R7 and the proposed decision roles
in the [review framework](README.md#requirements-shared-by-every-proposal).
The cases below are a research agenda, without executed or accepted vectors.
Use the [proposal template](PROPOSAL-TEMPLATE.md) to resolve each entry.

## F01: NoM events and state as L2 input

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L74-L96).
Suggested class: ZVM derivation change.

- **Specify:** A canonical successful embedded-call outcome and historical-state
  source, send/receive association, event order, identifiers and mandatory-queue
  deferral. Decide how missing historical state is reconstructed and verified.
- **Bound and review:** RPC trust, omitted or fabricated outcomes, forked history,
  rebuild cost, storage growth and competition with deposits/payouts. No mutable
  frontier query may silently stand in for state at the selected momentum.
- **Candidate cases:** Successful and failed calls with identical requested
  arguments; missing or delayed receive; duplicate delivery; same height with a
  different hash; queue saturation; interrupted replay and removal/re-inclusion.
  Decide expected state for every observation before exporting a vector.

## F02: HTLC swaps settled by derivation

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L98-L170).
Suggested class: ZVM derivation change plus escrow contract; depends on F01.

- **Specify:** Exact binding of offer, NoM entry, creator, recipient, asset,
  amount, hash algorithm, hashlock, proxy-unlock policy and expiry. Define the
  successful create/unlock/reclaim observations, queue delay and settlement order.
- **Bound and review:** Timeout and observation assumptions, LP option/griefing
  cost, escrow exposure, proxy-unlock restrictions and continued committee backing
  of wrapped assets. A shared timestamp alone does not prove a safe short expiry.
- **Candidate cases:** Two competing creates; wrong amount/recipient/algorithm;
  failed embedded receive; unlock/reclaim at both sides of expiry; withheld or
  reorged unlock; long mandatory-queue delay; crash after one leg is irreversible.
  Resolve whether each case binds, refunds, waits or refuses and why value is safe.

## F03: Committee accountability

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L172-L182).
Suggested class: contract and custody policy, subject to source-fact sufficiency.

- **Specify:** Authenticated attribution, cancellation/slashing evidence,
  withdrawal lifecycle and how a delayed or already signed payout interacts with
  a refund. Prove how evidence identifies responsible members rather than only
  the vault; do not assume an aggregate payout signature identifies its signers.
- **Bound and review:** Bond denomination, price/liquidity changes, correlated
  loss, maximum payable liability, theft versus liveness, withdrawal of collateral
  and committee resharing. Independent cryptographic/custody review is required.
- **Candidate cases:** Refund then late payout; duplicated anomaly; disputed
  history; threshold collusion; missing attribution; bond below liabilities;
  epoch change with in-flight withdrawals; operator loss and restored state.

## F04: A canonical ERC-20 for every ZTS

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L184-L189).
Suggested class: ZVM derivation change plus factory/token contracts; depends on F01.

- **Specify:** Domain-separated token identity, wrapper derivation, issuance and
  mutable metadata rules, decimal conversion, backing, deposits and redemption.
  Resolve how an existing wrapper or unsupported token migrates without duplication.
- **Bound and review:** Factory/metadata authority, malicious token metadata,
  amount overflow and dust, unsupported payout assets, wrapper proliferation and
  custody exposure. Specify fees and costs for creating otherwise unused wrappers.
- **Candidate cases:** Same metadata with distinct token IDs; repeated deposit;
  decimal extremes; metadata update; wrapper-address collision; reorg after mint;
  unsupported redemption and restart during factory creation.

## F05: Minting on L2 from a NoM burn

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L191-L195).
Suggested class: ZVM derivation change plus mint authority; depends on F01.

- **Specify:** Successful irreversible burn evidence, destination authorization,
  exact supply/decimal rules and chain domain. Decide how a burn names one
  destination rollup and how a reorg after mint affects accounting.
- **Bound and review:** Accidental burns, unavailable destination, issuer rights,
  multiple rollups accepting the same burn, griefing and loss when the L2 halts.
  State explicitly whether any reverse redemption exists and who can honor it.
- **Candidate cases:** Failed burn receive; missing/wrong destination; same burn
  on two configured rollups; duplicate replay; amount extremes; restored mint
  journal; conflicting burn history after the destination effect.

## F06: Fair ordering within a momentum

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L197-L202).
Suggested class: ZVM derivation change.

- **Specify:** Exact seed, canonical eligible input set, permutation algorithm,
  tie rules and interaction with mandatory/optional messages and chunk assembly.
  Preserve old ordering below the accepted activation boundary.
- **Bound and review:** Producer seed bias, grinding and withholding, censorship,
  economic extractable value, queue priority and worst-case sorting/resource cost.
  Define the fairness property and adversary before claiming improvement.
- **Candidate cases:** Permuted RPC enumeration; duplicate keys; empty/single
  input; grinding alternatives; full mandatory queue; deferred/chunked messages;
  old/new rules at the activation height and deterministic cross-client replay.

## F07: Onboarding by fusing for a new address

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L204-L210).
Suggested class: contract and committee policy; reassess any missing source facts.

- **Specify:** Recipient authorization, exact request/fusion identity, accepted
  observation of usable plasma, funding/reclaim rights and cancellation rules.
  Distinguish a requested fusion from completed and usable capacity.
- **Bound and review:** QSR budget, sybil demand, capacity delay, stranded capital,
  committee discretion and double funding. Accepted quotas and fees must have an
  accountable operator owner rather than an undocumented service default.
- **Candidate cases:** Duplicate/concurrent onboarding; wrong recipient; failed
  or delayed fusion; depleted budget; reorg and restart; reclaim while the user
  still relies on capacity; committee outage and withdrawal of funding.

## F08: Rollup fees to pillars and delegators

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L212-L217).
Suggested class: distribution contract; historical weight input depends on F01.

- **Specify:** Fee source, entitlement period, canonical historical weights,
  recipient mapping, claim/nullifier domain, rounding and unclaimed-funds policy.
  Define how source reorgs and fee-distribution changes affect accrued claims.
- **Bound and review:** Weight manipulation, last-moment delegation, sybil claims,
  admin diversion, dust, computation/storage growth and claim cost versus reward.
  Establish that every allocation is covered by collected distributable fees.
- **Candidate cases:** Weight change at period boundary; duplicate or cross-period
  claim; restored claim state; reorged snapshot; zero total weight; dust and
  maximum amount; concurrent claims and operator-selected snapshot disagreement.

## F09: Airdrops through L2

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L219-L224).
Suggested class: funded distribution contract and application policy.

- **Specify:** Eligibility authority and snapshot, chain-bound claim bytes,
  Zenon/L2 recipient mapping, funding and redemption, root replacement and expiry.
  Separate distribution entitlement from proving control of the intended recipient.
- **Bound and review:** Misallocation, sybil eligibility, stolen claims, relayer
  availability/cost, gasless-user access and committee-backed withdrawal exposure.
  An L2 claim must not promise free or immediate spendability on NoM.
- **Candidate cases:** Wrong chain/root/address encoding; duplicate/front-run
  claim; changed root with prior claims; insufficient funding; interrupted claim;
  reverted transfer; withdrawal fee exceeding a small entitlement.

## F10: Pillar-weighted scheduling of ZVM forks

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L226-L231).
Suggested class: governance contract plus node/derivation change; depends on F01.

- **Specify:** Voting authority, immutable weight snapshot, quorum/timelock,
  proposal identity, conflicting notices, release/configuration binding and exact
  activation height semantics. Define stale-client halt and replay from genesis.
- **Bound and review:** Governance capture, purchased weights, censorship, a vote
  scheduling unavailable code, emergency powers and committee disagreement. A
  contract vote cannot by itself establish legitimate upgrade authority.
- **Candidate cases:** Competing heights; vote/snapshot reorg; late or missing
  notice; changed client artifact; unknown fork; minority old client; restored
  notice state; cancellation immediately before and after activation.

## F11: Rollups messaging each other through NoM

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L233-L237).
Suggested class: ZVM derivation change; depends on F01 and each peer's rules.

- **Specify:** Source rollup/genesis/version authentication, message commitment,
  destination domain, source finality, delivery ordering and cross-rollup receipts.
  Decide whether NoM carries an authenticated execution result or merely input data.
- **Bound and review:** Dishonest source execution, duplicate relay, incompatible
  forks, recursive message loops, destination outage, locked value and gas funding.
  Shared L1 ordering alone does not authenticate another rollup's executed state.
- **Candidate cases:** Spoofed source; same message on another rollup; differing
  source rules; reorg on either derived history; replay after restore; failed
  destination call; partial delivery and upgrade while a message is in flight.

## F12: Registering the ZVM with the embedded bridge

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L239-L248).
Suggested class: bridge administrator decision plus ZVM derivation/contract change.

- **Specify:** Accepted network/token registry, bridge event authentication,
  native-gas versus ERC-20 mint/burn handling, decimals, fees, redeem delays and
  transition from existing custody. Keep the orchestrator threshold trust explicit.
- **Bound and review:** Registry/admin compromise, unsupported token mappings,
  parallel bridge backing, replay across network registrations and refund/loss
  ownership. Separate an operator registration from consensus proof enforcement.
- **Candidate cases:** Wrong network/class/token; changed token pair; duplicate
  wrap; halted bridge; reorged request; gas-token amount mismatch; old/new routes
  active together; in-flight redemption during registration or key rotation.

## F13: Client-side ZK sync and proof-gated signing

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L250-L258).
Suggested class: prover, client and signer policy; potentially separate designs.

- **Specify:** Proof statement/public inputs, trusted start/checkpoint, complete
  input commitment, execution rules, finality and data availability. Separate a
  wallet sync proof from the exact withdrawal fact a signer must verify.
- **Bound and review:** Proof-system/setup assumptions, verifier and witness bugs,
  proving cost, withheld data, stale proofs, signer fallback and quorum collusion.
  A signer policy does not make NoM enforce a proof or constrain a colluding quorum.
- **Candidate cases:** Valid proof for wrong chain/range/rules; omitted inbox
  input; wrong withdrawal recipient/amount; unavailable witness; restored signer
  state; revoked verifier; delayed prover and disagreement between independent nodes.

## F14: Ed25519 verification on L2

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L263-L269).
Suggested class: execution-client change plus application signature domains.

- **Specify:** Exact verification variant, input/output bytes, malformed-input
  behavior, canonical scalar/point and subgroup handling, gas and activation.
  Applications separately specify chain/contract/nonce/expiry-bound messages.
- **Bound and review:** Cryptographic interoperability, parser divergence, denial
  of service and dependency provenance. State which user flow needs a precompile
  rather than the existing on-chain approval or ERC-1271 paths.
- **Candidate cases:** Valid chosen-variant signature; noncanonical scalar/point;
  small-order inputs; invalid lengths; wrong variant/domain; gas boundary; replay
  on another application; old/new execution clients at activation.

## F15: Trustless peg-out

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L274-L298).
Suggested class: NoM consensus change plus bridge/prover or execution integration.

- **Specify:** Choose and justify execution commitments versus proof enforcement;
  bind genesis, ordered complete inbox, rule version, accepted state transition,
  withdrawal identity/recipient/asset/amount and persistent nullifiers on NoM.
  Define verifier replacement, delays, halts and old-custody migration.
- **Bound and review:** Soundness/setup, data availability, verifier resource
  bounds, compromised governance, unavailable prover/challenger and maximum
  liabilities during migration. Pin cryptographic code; a dependency containing
  pairing operations does not establish a complete, safe consensus verifier.
- **Candidate cases:** Valid proof of unrelated state; omitted batch; alternate
  genesis/rules; repeated/nullifier-rolled-back withdrawal; invalid point/proof;
  unavailable data; verifier upgrade; L1 reorg and both sides of activation.

## F16: L2 contracts that act on NoM

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L300-L315).
Suggested class: NoM consensus change, or a separately trusted committee service.

- **Specify:** Authenticated intent, originating contract/chain/version, target
  account and method, asset/amount/nonce, execution authority and success receipt.
  Define failure/compensation and authorization revocation while an intent waits.
- **Bound and review:** Contract compromise, confused authority, recursive calls,
  drained service budgets and double execution. The committee variant must state
  its signing trust, keys, limits and operator recovery independently of F15.
- **Candidate cases:** Forged origin; unauthorized method; replay after restore;
  partial embedded execution; stale revocation; reordered intents; source reorg
  after external execution; committee loss and duplicate success receipts.

## F17: Pillar preconfirmations

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L317-L331).
Suggested class: unresolved between contract/node and NoM consensus change.

- **Specify:** Scheduled producer authority, registered key, chain-bound promise,
  transaction/order/deadline, inclusion evidence, equivocation and slashing proof.
  Separate a promise from NoM inclusion, execution success and finality.
- **Bound and review:** Bond liquidity, cumulative promised exposure, censorship,
  scheduling changes, key compromise, false accusation and under-collateralization.
  Resolve the authenticated producer/schedule source before choosing a change class.
- **Candidate cases:** Two conflicting promises; producer change/reorg; inclusion
  with reverted execution; missing or withheld evidence; promise after deadline;
  cross-chain replay; bond withdrawal with promises outstanding; clock boundary.

## F18: A separate budget for batch data

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L333-L337).
Suggested class: NoM consensus/resource and fee change.

- **Specify:** Data-lane encoding, admission, byte/resource limits, pricing,
  retention/availability, interaction with ordinary account blocks and recovery
  on missing data. Bind any ZVM inbox changes to the same activation specification.
- **Bound and review:** Worst-case bandwidth/storage/verification, spam subsidy,
  fee manipulation, ordinary-user starvation and validator hardware requirements.
  Capacity and fee values require representative measured workloads and economics.
- **Candidate cases:** Exact limit and one byte over; incompressible/adversarial
  input; unavailable payload; simultaneous ordinary/data saturation; fee rounding
  and extremes; history replay and old/new node behavior at activation.

## F19: Paying sentinels for real work

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L339-L342).
Suggested class: NoM consensus/reward change, subject to an accepted work definition.

- **Specify:** Eligible worker identity, independently verifiable work,
  unpredictably selected challenges if needed, result freshness, reward period
  and domain-separated claim accounting. Define exactly what service is rewarded.
- **Bound and review:** Sybil identities, outsourced/replayed responses, colluding
  challengers, work performed only at challenge time, issuance budget, capital
  concentration and the cost of independently verifying the claimed service.
- **Candidate cases:** Same work claimed twice/by two identities; stale response;
  unavailable data between challenges; challenge bias; epoch/key changes; restored
  reward state; no useful work and reward exhaustion.

## F20: Momentum commitments for light wallets

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L344-L348).
Suggested class: NoM consensus/header change plus light-client protocol.

- **Specify:** Which state is committed, exact tree/key/value/hash encodings,
  inclusion and absence proofs, header authentication, finality/checkpoint
  authority and state/history update rules. Distinguish NoM state from L2 state.
- **Bound and review:** Stale but authentic roots, eclipse/dishonest RPC, omitted
  keys, proof-size/verification limits, archival cost and incompatible header
  versions. A Merkle proof alone cannot authenticate the containing momentum.
- **Candidate cases:** Correct proof under an unaccepted header; wrong height,
  chain or key; absence versus zero; malformed/deep proof; changed root after
  reorg; wallet restore with stale checkpoint and header-version activation.

## F21: Encrypted mempool on the pillars' threshold key

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L350-L355).
Suggested class: NoM consensus change plus independently reviewed cryptography.

- **Specify:** Encryption/threshold scheme, authenticated epochs, ciphertext
  domain, order commitment, release timing, verifiable shares, decryption failure
  and rotation. Assess new key generation and key separation before any reuse.
- **Bound and review:** Early threshold collusion, withholding, chosen ciphertexts,
  invalid shares, cross-protocol key exposure and denial of service. State the
  precise ordering/confidentiality property; do not assume front-running is eliminated.
- **Candidate cases:** Early or invalid shares; fewer than threshold available;
  epoch change with pending ciphertext; replay on another chain; malformed input;
  partial decryption and reorg; colluding threshold learns plaintext before ordering.

## F22: Synchronous L1/L2 calls

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L357-L361).
Suggested class: NoM consensus and execution integration; depends on a chosen F15 design.

- **Specify:** The common transition and commit point, cross-layer call/return
  encoding, authorization, gas/resource accounting, reentrancy and exception
  propagation. Resolve whether the chosen design can enforce atomicity at all.
- **Bound and review:** Coupled liveness, circular calls, inconsistent rollback,
  client divergence, block validity/capacity and effects external to the common
  transition. A committee or asynchronous proof path cannot silently stand in.
- **Candidate cases:** L1 succeeds/L2 fails and the reverse; nested cross-layer
  calls; budget exhaustion; producer/client interruption; conflicting execution;
  reorg and old/new clients. Specify every resulting state before calling it atomic.

## F23: ZK sync for NoM itself

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L363-L366).
Suggested class: prover/light-client design; NoM consensus change if on-chain enforcement changes.

- **Specify:** The complete NoM validity/finality statement, trusted genesis,
  consensus/validator evolution, spork rules, recursive composition, data
  availability and authenticated current head. Pin every verifier and circuit version.
- **Bound and review:** Incomplete circuits, invalid recursive composition,
  setup/key compromise, historic rule changes, stale proofs, prover outage and
  resource cost. A proof of valid old history does not prove a fresh canonical head.
- **Candidate cases:** Valid prefix with a stale tip; skipped rule/validator
  transition; proof splice across chains/epochs; omitted account block; malformed
  recursive proof; withheld data; verifier migration and restore from an old proof.

## F24: NoM accounts controlled by L2 contracts

[Source idea](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md#L368-L372).
Suggested class: NoM consensus/account change plus accepted L2-state enforcement.

- **Specify:** Account encoding, contract/chain/genesis identity, accepted state
  proof, authorization call and nonce, spend limits, upgrades/recovery and NoM
  send/receive semantics. Define who controls the account during L2 unavailability.
- **Bound and review:** Compromised contract/admin, stale authorization, recovery
  capture, recursive authority, lost proofs and irreversible L1 effects after an
  L2 reorg. State custody and escape assumptions without inventing a master key.
- **Candidate cases:** Permission revoked after proof creation; wrong contract or
  L2 version; repeated authorization; concurrent sends; contract/key rotation;
  halted L2; recovery dispute; restored account state and changed proof verifier.
