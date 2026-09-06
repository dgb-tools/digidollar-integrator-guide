# DigiDollar Integrator's Guide — what you can safely promise

**Version 0.1.2 · 2026-09-06.** Describes DigiByte Core **v9.26.5** (the current
mainnet release). Every source file cited below was checked byte-identical on the
`v9.26.4` and `v9.26.5` tags and on `develop` on 2026-09-05; if a later release touches
one, the pin here is what this text describes. DigiDollar has been active on DigiByte mainnet since block
**23,869,440** (17 July 2026). Everything here is *current behavior*, not a
permanent guarantee — the founder characterizes DigiDollar as still under active
development, and no change described as possible should be treated as committed
until it is implemented, tested, and given an activation path.

**Sources, with attribution.** The design rulings in §§2–7 are the written answers
of Jared Tate (DigiByte founder and maintainer) to this author's integrator
questions, September 2026, quoted with permission on the condition that this
document pins the release it describes and does not present current behavior as a
mainnet guarantee. The output-floor rules in §1 are from Shen (DigiByte Core) in
[Core Discussion #425](https://github.com/orgs/DigiByte-Core/discussions/425), July
2026. Chain-level facts are from this author's decoding of mainnet transactions
([week-one census, v3](https://dgbinsights.com/census/DD_MAINNET_CENSUS_W1_v3.md) — exact reconciliation with the
node; raw records: [redemptions JSON](https://dgbinsights.com/census/redemptions-week1-v3.json), [186-record JSONL](https://dgbinsights.com/census/census-mainnet-week1.jsonl)) and from operating oracle slot 29 since activation. Where a code reference is
given, it is the file the founder cited; verify against the release you build on.

If you find an error, treat it as ours until shown otherwise, and open an issue.

---

## 0. Read this first: the one-paragraph model

DigiDollar V1 is an **overcollateralized, synthetic, USD-denominated asset native to
DigiByte consensus**. Anyone can mint DD by locking DGB collateral in a time-locked
vault; the DD is fungible and transfers like any UTXO asset. **It is not a bearer
claim on collateral.** Only the vault's owner — after the vault matures, by burning
the required DD, with the owner key's signature — can release collateral. Peg
support, collateral backing, redemption authority, and market liquidity are four
different properties, and the single most common integrator error is treating them
as one. In the founder's words: *"Holding DD does not provide a direct claim
against any specific vault or an unconditional protocol redemption at $1."*

**The recommended public disclosure, verbatim from the founder:**

> DigiDollar is an experimental, overcollateralized DigiByte-native asset. Holding
> DD does not confer a protocol-level claim against arbitrary collateral.
> Redemption requires a matured vault and authorization from that vault's owner.
> Peg maintenance and secondary liquidity are market-dependent. Oracle outages can
> suspend minting and redemption, while volatility protection can temporarily
> suspend all DigiDollar operations, including ordinary transfers.

Put that, or something substantially like it, wherever you show a DD balance.

---

## 1. Consensus facts you hit on day one

**Amounts and the on-chain ledger.** DD lives as zero-DGB taproot outputs whose
dollar value is declared in an `OP_RETURN` that begins `DD <type>`. Amounts are
**integer cents**, minimal-length little-endian. Three transaction types, dispatched
on the type byte — never on transaction shape:

| type | meaning | the payload declares | what is burned |
|---|---|---|---|
| `01` | mint | the one DD output created, plus lock height and tier | nothing |
| `02` | transfer | each DD output created, in order | nothing (Σ in = Σ out) |
| `03` | redeem | the one *change* output created | consumed − change |
| *(none)* | redeem with **no DD record in the transaction at all** | nothing — no DD output can be created | everything consumed |

**The rule that unifies them:** the payload is a ledger of *created* outputs. A
redemption's amount field is its change, not its burn — burns are implicit. Two
invariants let you check your decoder: transfers conserve value exactly, and every
week-one burn equalled the mint it closed, to the cent.

**Decoder traps** (each one broke a real implementation; details in the census):
walk pushes, never byte offsets (minimal-length amounts shift every later field) ·
`0x00` is `OP_0`, a zero-length push meaning zero (how lock tier 0 is encoded — 16% of
early mints) · `OP_1`–`OP_16` are single-byte opcodes (`0x51`–`0x60`) that push the numbers 1–16 · a type-3 redemption looks
exactly like a transfer by shape · the payload declares change, not burn.

**Floors, from Core Discussion #425 (Shen):**

- **Minimum mint: $100** — the only *gated* floor (activation-height parameter).
- **Minimum DD output: $1** — an **ungated, unconditional** consensus rule
  (`ValidateOutputAmount` applies at all heights; changeable only by a coordinated
  consensus upgrade). Rationale: dust/UTXO-set protection for zero-DGB outputs.
- **Change must be exactly $0 or ≥ $1.** Paying $1.00 from a $1.50 output is
  *invalid* — it would create $0.50 of change. **Your coin selection must produce
  exact payment or leave at least $1 of change.** This is the sharpest edge in DD
  integration and the one most likely to fail silently in testing. Every zero-change
  redemption on mainnet so far has used the no-record form; whether a type-3 record
  declaring $0 change is accepted is not verified here — do not rely on it.
- Consequently: DGB is the micro-payment tier; DD is ≥ $1 on-chain settlement;
  sub-dollar DD flows are an application-layer concern (tabs, prepaid balances).

**Standardness.** DigiByte's relay limit for an `OP_RETURN` output is
`MAX_OP_RETURN_RELAY = 83` bytes of *script* (`src/policy/policy.h`): the `OP_RETURN`
opcode, the push prefix, and up to **80 bytes of data**. An 80-byte payload relays;
an 83-byte *payload* does not (this author's July test). Standard policy also allows
**one** `OP_RETURN` output per transaction (`multi-op-return`, `src/policy/policy.cpp`),
and in a DD transaction that output *is* the DD record — so there is no separate canvas
for a commitment of your own inside a DD transaction. Put commitments in a separate DGB
transaction, within 80 data bytes.

**Three more rules you hit on day one:**

- **DD inputs must be confirmed.** Consensus — not merely relay policy — rejects
  spending or redeeming DD outputs created by unconfirmed transactions
  (`TX_CONSENSUS`, reject reason `dd-input-amounts-unknown`; messages `"Cannot spend unconfirmed DD
  inputs"` and `"Cannot redeem unconfirmed DD inputs"`, `src/digidollar/validation.cpp`).
  Chained unconfirmed DD payments — normal for DGB — do not work for DD.
- **Meaning comes from the record, not from labels.** Classify by the type byte and
  the declared outputs; do not infer from an explorer's caption. The public explorer
  currently captions a redemption's declared *change* as the burn
  ([Discussion #446](https://github.com/orgs/DigiByte-Core/discussions/446)). An
  unknown type byte is undecodable — refuse to classify rather than guess.
- **Fees are DGB.** DD outputs carry zero DGB, so every DD transaction needs a DGB
  input for fees. A fee-sponsorship design ("Paymaster", Shen) exists as a concept
  draft with a regtest prototype — [Discussion #430](https://github.com/orgs/DigiByte-Core/discussions/430)
  — and is not a proposed consensus change.
- **Reconcile from the chain, not a wallet UI.** Operators reported DD change not
  appearing in the Core wallet (DigiDollar Gitter room, Aug 2026).

A reference decoder with test vectors exists: `dgb-digidollar-codec` 0.2.0 on npm
(186 week-one census records plus live fixtures; MIT), and `dd-explain-mcp` 1.0.0 for
agents. Neither is authoritative; both are cross-checks.

**Collateral.** Vault collateral is a taproot output whose internal key is a
NUMS point (`COLLATERAL_NUMS_POINT_BYTES`, `src/digidollar/scripts.h`) — no known
key-path private key exists, so the construction is intended to disable key-path
spending — and both script paths begin with `<lockHeight> OP_CHECKLOCKTIMEVERIFY`.
The source comment reads *"NO early redemption, NO forced liquidation, NO
exceptions."* Lock tiers observed on mainnet: 0 (~1 hour, test), 1 (30 days), 2 (90
days), 3 (180 days), 4 (1 year), 5 (2 y), 6 (3 y), 7 (5 y), 8 (7 y), 9 (10 y).

**Units traps in public feeds** (the dgbinsights feeds and, likely, others'): supply figures in cents;
the oracle price in dollars only in `price_usd` (`price_cents` rounds sub-cent DGB
to 0); 24-hour range fields in cents. Divide before you chart.

---

## 2. Who can redeem — holder rights (founder answer 1)

- An unrelated DD holder has **no protocol-level right** to redeem anyone's
  collateral. Both the normal and emergency (ERR) paths require vault-timelock
  expiry, destruction of the required DD, and a valid signature from the owner key
  committed to that vault (`CreateNormalRedemptionPath()`, `CreateERRPath()` in
  `src/digidollar/scripts.cpp`; accounting in `ValidateRedemptionTransaction()` and
  `ValidateCollateralReleaseAmount()`, `src/digidollar/validation.cpp`).
- The wallet's "Position not found" is a wallet-level restriction, but a custom
  transaction builder cannot bypass the consensus-required owner signature either.
- DD is fungible *for burning*: an owner may acquire DD from anyone to redeem.
  Transferring DD **never** transfers ownership of the originating vault.
- Par for a non-owner is supported only **indirectly**: new minting when minting
  above par is attractive, secondary-market liquidity and arbitrage, demand from
  matured owners who need DD to unlock, and supply contraction through completed
  redemptions. The protocol enforces valid mints, transfers, burns, and releases. It
  does **not** guarantee liquidity, owner participation, a standing $1 bid, or
  direct holder redemption.

**Disclosure (founder's wording, answer 1):** *Holding DD does not provide a direct claim against any specific
vault or an unconditional protocol redemption at $1.*

---

## 3. Backing is not liquidity — abandoned vaults (founder answers 2 & 3)

- A matured vault can remain unspent **indefinitely** if its owner loses the key,
  disappears, or declines to buy the required DD. Maturity makes redemption
  *eligible*; it creates no anyone-can-spend path, no forced liquidation, no
  automatic settlement.
- While unspent, that vault stays inside aggregate collateral, supply, and position
  counts (`SystemHealthMonitor::ScanUTXOSet()`, `src/digidollar/health.cpp`;
  `DigiDollarStatsIndex::CustomAppend()`, `src/index/digidollarstatsindex.cpp`).
  Under v9.26.5, collateral whose owner key is unavailable stays locked *while still
  appearing in backing figures*; the current rules provide **no salvage,
  reassignment, liquidation, or timeout-cleanup path.**
- Therefore, in the founder's words (answer 2): *"aggregate collateral is not
  equivalent to economically actionable redemption liquidity."* Abandoned vaults make backing statistics look stronger
  than the collateral actually available to settle current holders.
- Fungibility means every DD output is the same cents-denominated unit; transfers
  carry no vault provenance, maturity, seniority, or collateral rights forward. A
  common unit of account — not a pro-rata claim on the pool.

**Disclosure (founder's wording, answers 2–3):** *The system health ratio is an aggregate solvency indicator, not a
guarantee of holder liquidity or immediate redeemability.*

---

## 4. Emergency redemption (ERR) — incentives and ordering (founder answer 4)

- When system health is poor, a vault owner must burn **more** DD than the position
  minted to release full collateral — capped at **1.25× the original DD amount**
  (`EmergencyRedemptionRatio::GetRequiredDDBurn()`, `src/consensus/err.cpp`;
  enforced by `ValidateCollateralReleaseAmount()`).
- The owner's incentive to redeem exists only when recovered collateral is worth
  more than the market cost of acquiring and burning the required DD. **A below-par
  DD price is part of the intended recovery mechanism** — discounted DD makes
  redemption profitable while the enlarged burn contracts supply. That mechanism is
  market-dependent; a rational owner may decline to redeem.
- There is **no consensus-level redemption queue or holder priority.** Valid
  redemptions are ordinary transactions competing for block inclusion.
- Because ERR can burn DD that originated from other positions, supply can contract
  faster than vault count. V1 does **not** guarantee that enough DD remains for every
  later owner to redeem, and provides no terminal global-settlement or
  loss-socialization mechanism.
- Observed: in DigiDollar's first mainnet week, all 13 redemptions burned exactly
  their positions' mint amounts — no ERR multiplier was active.
- **The mint-side counterpart, DCA** (dynamic collateral adjustment), raises the
  collateral required per new DD as system health falls. Tiers from the
  `HEALTH_TIERS` table in `src/consensus/dca.cpp`: health ≥ 150% → 1.0×;
  120–149% → 1.25×; 110–119% → 1.5×; below 110% → 2.0×. (The doc comment atop
  `dca.h` still lists older bands and multipliers; the table is what runs.) The
  founder stated the same top three bands in the DigiDollar Gitter room, 27 May 2026.

**Disclosure (founder's wording, answer 4):** *ERR is a supply-contraction and collateral-recovery mechanism, not a
guarantee that every holder or vault owner can exit.*

---

## 5. The oracle dependency (founder answer 5, plus operator data)

- Price comes from a **MuSig2 quorum of 7 signatures from 35 configured oracle
  keys**, aggregated into one Schnorr signature plus a participation bitmap that
  miners embed in the coinbase (`OP_ORACLE` bundle, version byte `0x03`); every full node validates
  it on `CheckBlock`. Quotes are valid for a bounded number of blocks (20 at the time
  of writing — read `validity_blocks` from `getoracleprice`).
- The threshold favors **liveness** over collusion resistance: seven valid
  participants keep the feed alive while many are dark; seven cooperating or
  compromised keys can satisfy it. Signatures prove agreement by an authorized
  quorum, **not** that operators used economically independent data — correlated
  exchange feeds (the stock Core binary initializes six exchange fetchers and needs
  three; operators *can* patch the set, and nothing on-chain shows whether any do),
  shared software,
  shared infrastructure, thin-market manipulation, and supply-chain compromise
  remain live risks. The founder: *"I cannot point to a published formal security
  analysis proving that seven is the uniquely correct threshold. It is the
  implemented availability-versus-collusion trade-off and should be documented as
  such."*
- The roster is rooted in chain parameters. **Adding, removing, rotating, or
  replacing keys is a coordinated consensus upgrade with an activation plan** — not
  an operator-local change. If quorum is lost for days, **there is no automatic
  fallback oracle**; once the last accepted quote is no longer current
  (`validity_blocks`, 20 at the time of writing), mints and redemptions stop until
  quorum recovers or an upgrade changes the roster.
- **Transfers do not need an oracle quote.** Mempool admission requires a recent
  quote for mint and redeem only (`DigiDollarMempoolTxRequiresOracleQuote()`,
  `src/validation.cpp`). Ordinary transfers continue through a pure oracle outage —
  unless a volatility freeze (§6) is separately active.
- **Observed, 26–27 August 2026:** a malformed network message crashed daemons
  across the oracle set (the founder's diagnosis: a segfault in the message-handler
  thread, `txmempool.cpp`); reporting participants fell from ~31 to **11** for roughly
  a day. Quorum held. Minting continued throughout — four new positions and ~1.3M
  DGB of collateral were added *during* the outage. Recovery depended on operators
  noticing by hand, which is why a free operator kit now exists
  ([dgb-tools/oracle-ops](https://github.com/dgb-tools/oracle-ops)).

**Disclosure (founder's wording, answer 5):** *Oracle availability is a dependency for minting and redemption,
while ordinary transfers may separately be suspended by volatility protection.*

---

## 6. Volatility freezes and what "settled" means (founder answer 7)

Consensus itself rejects the affected DD transaction types during volatility
protection — which types depends on the tier reached. Thresholds in
`src/consensus/volatility.h`, enforced by `ValidateMintTransaction()`,
`ValidateTransferTransaction()`, `ValidateRedemptionTransaction()`:

| measured volatility | effect |
|---|---|
| ≥ 20% over one hour | new **minting** frozen |
| ≥ 30% over 24 hours | **all** DD operations frozen — including ordinary transfers |
| ≥ 50% over seven days | **emergency** all-operations freeze |
| release | no earlier than the block *after* the **8,640-block cooldown (~36 hours)** — `currentHeight > cooldownEndHeight` — *and* only once every window is back under a **stricter** bar: 1h < 10%, 24h < 20%, 7d < 30% (`VolatilityMonitor::UpdateState()`, `src/consensus/volatility.cpp` — the recovery checks reuse the next-lower threshold constants — the 1h bar is the 10% *warning* level — so a freeze never lifts at the level that caused it) |

State derives deterministically from accepted on-chain oracle prices, heights, and
block order (`VolatilityMonitor::UpdateState()`), and follows reorgs
(`RemovePriceForHeight()` on disconnect). The founder notes this remains
consensus-sensitive code that should be supported by explicit restart, replay,
disconnect, and competing-chain tests before being treated as finalized.

**The founder's definition (answer 7): "settled" means a confirmed transaction
established ownership of a valid DD UTXO.** That is ownership under the current
chain tip, not irreversible finality — reorgs apply as for any UTXO. It does not mean the balance will remain continuously
transferable or redeemable: during an all-operations freeze it is recorded but
temporarily non-transferable and non-redeemable.

**A settlement policy that respects this** (the one this author's checkout uses):

- Quote DD amounts with a short TTL (15 minutes) from a price you cross-check
  against a second source; refuse to quote on divergence.
- Show "confirming" from first sight; **fulfil at 6 confirmations** (~90 seconds).
  Core itself only requires DD inputs to be confirmed (≥ 1); six is this author's
  margin, not a protocol number.
- Treat received DD as *held*, not *spendable*: before promising any outbound DD
  transfer (refund, payout), read the volatility-protection state from the
  `getdigidollarstats` RPC, and be prepared for a hold of at least ~36 hours — longer
  until every recovery bar is met.
- Never let a valid DD balance in your UI imply a $1 redemption right (§2).

---

## 7. What testing does and does not prove (founder answer 6)

The repository carries substantial consensus and accounting tests — among them
`digidollar_burn_enforcement_tests`, `digidollar_redeem_tests`,
`digidollar_volatility_tests`, `digidollar_rh16_reorg_attacks_tests`,
`digidollar_rh31_consensus_fork_tests`, `digidollar_rh33_mempool_relay_tests`,
`oracle_bundle_manager_tests`, `digidollar_oracle_roster_tests`, the MuSig2
orchestration and network-attack suites (`src/test/`), and the functional tests under
`test/functional/`. They establish transaction validity, conservation, burn
enforcement, arithmetic boundaries, oracle signatures, reorg behavior, and freeze
rules.

They do **not** establish that aggregate collateral always exceeds supply under
arbitrary drawdowns, that DD returns to par, that every owner can eventually exit,
or that every holder has an exit. The founder is *"not aware of a published,
reproducible market or agent-based simulation"* covering severe DGB drawdowns, absent
DD liquidity, owner non-participation, concentrated maturities, ERR activation, and
prolonged below-par trading together. Those are separate economic propositions.

**Disclosure (founder's wording, answer 6):** *Current testing demonstrates protocol correctness under tested
conditions, not guaranteed peg recovery or universal redemption under market
stress.*

---

## 8. Canonical metrics, and where to get them

**Exposed today** via the `getdigidollarstats` RPC (backed in Core by
`SystemHealthMonitor` and `DigiDollarStatsIndex`): total outstanding DD supply · total unspent DGB collateral ·
oracle-valued system collateral ratio · active vault count · oracle price and
availability · DCA and ERR state · volatility-protection state.

**Recommended for merchant-grade settlement risk, not yet canonical RPC metrics:**
maturity ladder and concentration · collateral-tier distribution · matured-but-
unredeemed vaults · underwater-position count and value · oracle quorum size and
quote age · volatility-freeze status · DD market depth and deviation from par · the
share of collateral that is *economically actionable* — which cannot be proven from
the chain, because owner willingness and key availability are not on it.

**Public instruments** (independent, non-authoritative — reconcile against your own
node): [dgbinsights.com](https://dgbinsights.com) — a 5-minute snapshot feed since
activation, a daily archive retained since activation (as of 2026-09-05), the
[week-one census](https://dgbinsights.com/census/DD_MAINNET_CENSUS_W1_v3.md) with raw records
([redemptions JSON](https://dgbinsights.com/census/redemptions-week1-v3.json), [186-record JSONL](https://dgbinsights.com/census/census-mainnet-week1.jsonl)), and a
documented API with the units traps spelled out. A position scanner producing the
maturity ladder, matured-unredeemed set, and an oracle participation history from
the coinbase bitmaps is in progress there.

**Reconciliation discipline that has worked:** derive every figure two ways (chain
walk and node RPC) at the same height, publish the raw records, and treat a
discrepancy as your error until shown otherwise. It took five corrections to make
the week-one census exact; the raw records made the last one possible without a
new chain scan.

---

## 9. What you may and may not say

| you may say | you may not say |
|---|---|
| "DD is a USD-denominated synthetic asset minted against overcollateralized vaults on DigiByte." | "DD is redeemable for $1." |
| "Aggregate collateralization is X% — a system metric, not actionable solvency." | "Every DD is backed by $X of accessible collateral." |
| "Redemption releases collateral to the vault owner after maturity, against the required DD burn and the owner's signature." | "Holders can redeem." |
| "Settled = a confirmed DD UTXO you control (Core accepts spends at 1 confirmation; choose your own fulfilment depth)." | "Settled = always transferable." |
| "Transfers can continue through a pure oracle outage; once the last valid quote expires, mint and redeem stop. A volatility freeze can separately halt transfers." | "DD never freezes." |
| "Consensus tests cover validity and conservation rules under tested conditions." | "The peg is proven." |
| "Behavior described for Core v9.26.5, as of Sept 2026." | anything as a permanent guarantee |

---

## 10. Changelog and terms

- **0.1.2 — 2026-09-06.** §1: a DD transaction's single `OP_RETURN` is the DD record,
  so commitments need their own transaction; the no-record redemption row made
  explicit; zero-change redemptions noted as unverified for type 3; `OP_1`–`OP_16`
  called opcodes; wallet-reconciliation bullet separated; first-person slips removed.
  §5 bundle notation. §6: the 1h recovery bar named as the warning level. Census raw
  records (JSON and 186-record JSONL) published and linked.
- **0.1.1 — 2026-09-05.** Review corrections on the published text: header section
  pins; confirmed-only DD inputs stated as consensus, not policy; Paymaster cited to
  Discussion #430 as a concept draft; NUMS wording; the quote-validity grace before
  mint/redeem stop; §3 lock wording without permanence claims; §6 hold duration and
  the founder's "settled" definition attributed; §8 archive wording; §9 rows; the
  week-one census linked to its published record.
- **0.1 — 2026-09-05.** First release. Founder answers incorporated with permission;
  #425 floors; census v3 decoding rules; incident observation; DCA tiers and
  operator reports from the DigiDollar Gitter room; corrections from pre-release
  review (§1 relay cap and confirmed-input rule, §6 recovery bars, §9 caveats).

This guide is © its author and released under **CC BY 4.0**; quoted founder
statements remain his and are reproduced under the terms stated above. Part of
[dgb-tools](https://github.com/dgb-tools). Corrections welcome — issues and PRs.
