# Zeal — a consumer Zcash wallet that spends like a bank account

> Working codename. The product thesis, not the name, is the thing to react to.

**One line:** Hold your money as shielded ZEC. Spend it as local currency. The
conversion to stablecoins over NEAR Intents happens in the middle and the user
never learns the word "swap."

---

## 1. The problem worth solving

Zcash has a store-of-value story and no last mile.

The asset is structurally hard to exit. OKX cut ZEC pairs, Bit2Me stopped ZEC
trading, and Binance Dubai dropped it because privacy tokens fail VARA listing
rules. Every year the set of venues that will take ZEC and hand back fiat gets
thinner, and the ones that remain are exactly the ones that demand a transparent
address and full KYC — which defeats the point of holding it.

So a ZEC holder today has three bad options: keep it and never spend it, sell it
on a shrinking list of exchanges, or route it through a chain of manual steps
(shield → unshield → exchange → bank) that leaks the privacy they paid for.

Meanwhile the *other* asset class solved the last mile completely. Stablecoin
off-ramp infrastructure is now boring, licensed, and available as an API: Bridge
(inside Stripe since 2024) converts USDC/USDT to fiat and plugs into Stripe
Treasury and Issuing, and is shipping stablecoin-linked Visa cards across 100+
countries with Visa by end of 2026. Rain does direct-to-bank payouts on domestic
rails with embedded compliance. ZeroHash covers regulated custody and payouts.

**The arbitrage is obvious: ZEC has the privacy, stablecoins have the rails.**
Nobody has connected them in a way a normal person can use.

## 2. Why this is buildable now, and wasn't 18 months ago

The connecting piece shipped. NEAR Intents is cross-chain settlement where
market makers compete to fill a user's stated intent, and it now handles ZEC in
both directions — including into fresh *shielded* addresses.

This is not speculative; it is live and load-bearing for real Zcash apps:

- **Zashi** (Electric Coin Company's own mobile wallet) runs on NEAR Intents,
  and its activity spike was visible enough to make headlines. Users convert
  BTC, SOL, USDC into ZEC in-app and shield it.
- **Zodl** converts dozens of assets into shielded ZEC, depositing to a fresh
  shielded address per swap.
- **SwapKit** integrated Zcash across 20+ chains on the same rails.

Integration is a REST API, not a protocol research project. **1Click** requests
quotes from market makers, returns a quote-specific `depositAddress`, and the
user just sends the input asset there; a solver picks it up and settles, usually
in about a second on the NEAR side. 1Click does not take custody — assets sit in
the quote-specific address and settle on-chain. Fee collection is a single
parameter on the quote request, which is the monetization hook.

**But everyone built the same half.** Zashi, Zodl, SwapKit all point *into* ZEC.
Zashi's CrossPay lets you spend shielded ZEC and have the recipient get crypto
on any NEAR-supported chain — still crypto, not money. ECC has been *evaluating*
Banxa, Coinbase, and debit card support for fiat ramps; it is not shipped.

The exit to fiat is the unbuilt half. That is the product.

## 3. The product

### Design rule

The user has one mental model: **"I have $X."** Everything else — shielding,
intents, solvers, quotes, USDC, chains, gas — is implementation detail that
never reaches the screen. If a screen contains the word *bridge*, *slippage*,
*network*, or *seed phrase*, it is a bug.

### Onboarding

- Passkey/biometric account creation. No twelve words shown at signup.
- Keys generated on device, wrapped by the passkey, encrypted backup to
  iCloud/Google. Non-custodial, but recoverable the way a normal app is.
  (Advanced users can export the seed; it is in Settings, not in the funnel.)
- A unified address, shielded-by-default, generated silently.
- Time to first screen: under 30 seconds, no KYC yet.

KYC is deferred until the user first tries to cash out or order a card — the
moment where it's obviously necessary and the user has already seen value. This
matters: it keeps the "receive and hold privately" path completely KYC-free.

### Home screen

```
        $ 2,418.60
        ≈ 3.04 ZEC · private

   [ Receive ]  [ Send ]  [ Spend ]
```

Balance is denominated in the user's local currency by default, with ZEC as the
secondary line. Tapping the number flips the primary/secondary. "private" is a
live status word, not decoration — it turns amber if any funds are sitting
transparent and offers one tap to fix it.

### The four things it does

| Action | What the user thinks | What actually happens |
|---|---|---|
| **Receive** | "Got paid." | Funds land at a unified address; anything transparent auto-shields on arrival. |
| **Send** | "Sent Maria $40." | Z→Z if Maria is on Zeal. Otherwise a 1Click intent settles in whatever she can receive. |
| **Spend** | "Tapped my card." | Just-in-time shielded ZEC → USDC via 1Click, settled into the card program's balance. |
| **Cash out** | "Money's in my bank Tuesday." | ZEC → USDC → partner off-ramp → local rails (SEPA, ACH, PIX, SPEI). |

**Spend is the centerpiece.** A card that draws on a shielded ZEC balance, with
conversion happening per-transaction, is a thing that does not exist today. It
turns ZEC from an asset you have to *decide to liquidate* into an asset you just
*use* — which is the entire difference between a store of value and money.

## 4. How it actually works

### The spend path

```
shielded ZEC  ──1Click quote──►  quote depositAddress  ──solver──►  USDC
                                                                      │
                                                          partner ramp balance
                                                                      │
                                                    ┌─────────────────┴──────┐
                                              Visa authorization       bank payout
```

Two implementation notes that matter more than they look:

1. **The swap's destination is the ramp partner's deposit address for that
   user**, not the user's own wallet. USDC never sits in a user-controlled hot
   balance, which removes a step, a gas problem, and a support burden.

2. **Card authorization cannot wait on a swap.** Even a fast intent is too slow
   for a terminal. So the card runs against a **pre-funded float** the program
   maintains, and the ZEC→USDC conversion happens as *replenishment* behind the
   authorization. The user's ZEC is what backs the float; the float is what
   answers Visa in 200ms. This is a balance-sheet design decision, not a
   protocol one, and it is where the real engineering is.

### Stack

- **Client:** native iOS/Android. Zcash keys and note management via the mobile
  wallet SDK (librustzcash under it). Shielded scanning is the hard client
  problem — budget for it properly; it is what makes or breaks perceived speed.
- **Swap:** NEAR Intents 1Click. Quote → deposit → status poll → settle, with
  its built-in retry and refund handling.
- **Ramp + card:** Rain or Bridge as primary (Bridge if the Stripe Issuing
  integration is wanted; Rain if local payout rails matter more), ZeroHash as
  the regulated-custody fallback. Build the integration behind an internal
  interface from day one — you *will* switch providers, probably under duress.
- **Backend:** quote orchestration, float management, screening pre-flight,
  refund state machine. Deliberately thin; it holds no user keys.

## 5. The three things that can kill this

Everything above is the easy part. These are the parts that decide whether the
product exists.

### 5.1 Compliance holds are the existential risk, not a footnote

In July 2026 a user swapped 1,120 shielded ZEC for roughly $589k USDT through
Zodl on NEAR Intents. The deposit was recorded on-chain but not credited. A
compliance clearance was issued on July 23 and then *revoked* on August 26 under
a new regulatory review. Fifty days later the funds were still stuck. Zodl's
position was that the app has no control over assets routed through external
infrastructure — which is true, and which is exactly the problem.

NEAR Intents runs real-time screening through third-party providers (TRM Labs,
AMLBot, PureFi, Binance AML). Screening can block a swap, refunds then require
manual review, and that review can run past the stated window. **And
shielded-ZEC-derived funds are precisely the profile these systems flag.** This
isn't an edge case for this product; it's the main case.

If a consumer wallet's "cash out" button can eat someone's money for 50 days
with no recourse and a shrug about external infrastructure, the product is dead
on its first bad week.

Non-negotiable design responses:

- **Pre-flight screening before the user commits.** Screen the intended route
  and amount *before* the ZEC leaves the shielded pool, and refuse up front
  rather than failing mid-flight. A declined quote is a bad minute; a stuck
  swap is a lost customer and a forum thread.
- **Bounded exposure per intent.** Split large cash-outs into multiple smaller
  intents over time. This caps how much can be frozen in any single event and
  happens to be better for privacy too (see 5.2). Large amounts get a different,
  explicitly-slower, explicitly-explained path — not the consumer button.
- **The program eats the timing risk, not the user.** If an intent is held, the
  user is made whole from float on a published SLA and the company pursues the
  release. This is an underwriting cost and it must be priced into the spread
  from day one. It is also the single strongest reason a user picks this over
  doing it themselves.
- **A refund path that is a product surface**, with a real status, a real
  timeline, and a human. Not a support email.

### 5.2 This product trades privacy-of-spending for privacy-of-holding. Say so.

Be honest about what happens: shielded ZEC → USDC on a transparent chain → a
KYC'd off-ramp → a named bank account is a **fully deanonymized exit**. The
privacy protects the holding period, not the exit.

The chain analysis is already good at this. Arkham has labeled 53% of ZEC
transactions and attributed $420B of volume by identifying exchanges, which use
transparent addresses almost exclusively. Academic work on shielded-pool
round-trips — transparent → shielded → transparent with the same or similar
amount, close in time — matched 31.5% of all coins sent into the shielded pool.
Under a quarter of ZEC supply sits shielded, so the anonymity set is thinner
than the marketing suggests.

A wallet doing frequent, amount-matched, temporally-tight shielded exits is
*generating* that exact heuristic's signal. Mitigations, in order of value:

- **Never round-trip a matching amount.** Convert non-round amounts, offset from
  the spend, drawing on float so the on-chain amount and the user-visible amount
  need not agree.
- **Decouple timing from the spend.** Replenishment batches on its own schedule,
  not on the user's tap. This falls out of the float design for free.
- **Fresh shielded addresses per flow**, as Zodl already does on the inbound side.
- **Keep the Z→Z path first-class.** Zeal-to-Zeal payments never touch any of
  this and should be the fastest, cheapest, most-promoted path in the app. The
  best privacy outcome is the money never leaving.

And then tell the user, in one plain sentence at the moment of cash-out:
*"Cashing out links this money to your identity. Money you keep in Zeal, or send
to other Zeal users, stays private."* Users who chose a privacy coin will
respect being told the truth far more than they'll respect a privacy badge that
overclaims.

### 5.3 The program, not the user, is what gets deplatformed

A card issuer's BIN sponsor and a banking partner have to knowingly accept
ZEC-sourced volume. Many will not. This is the diligence to do *before* writing
client code, because it determines whether the product is legal-and-fundable or
a demo.

Practical posture: partner-licensed rather than own-licensed at the start; pick
the ramp partner on their willingness to underwrite privacy-coin-sourced flow,
not on API quality; expect to be asked for transaction monitoring beyond the
partner's baseline; assume the answer differs per corridor and launch where it's
yes. Also assume the answer can be withdrawn — which is the real argument for
the provider-abstraction layer in §4.

### 5.4 (Minor, but constant) volatility

ZEC moved enough in 2026 — a June collapse after the Orchard counterfeiting
vulnerability disclosure, then a climb toward $800 in August, the highest since
2018 — that a balance denominated in fiat on the home screen will visibly move.
Float absorbs the intra-transaction risk. The user-facing answer is an optional
**"lock to dollars"** toggle: a portion of the balance is held as USDC instead
of ZEC, for users who want a spending pocket that doesn't swing. Volatility is
also the honest argument for the whole product — the way to make a volatile
private asset spendable is to convert at the edge, not to ask merchants to
accept it.

## 6. Money

Three lines, all usage-based, none of them a subscription:

- **Spread on conversion.** 1Click takes a fee parameter on the quote; set it,
  show the all-in rate, take 0.3–0.7% depending on corridor. This is the
  primary line and it also funds the make-whole SLA in §5.1.
- **Interchange on card spend.** The standard card-program economics, and the
  reason "Spend" beats "Cash out" as the flagship action — it monetizes per
  transaction instead of per exit.
- **FX on local payouts.** Where the corridor supports it.

Free: receiving, holding, shielding, and Z→Z sends. Those are the retention
surface; charging for them would be strategically illiterate.

## 7. Competitive position

| | Shielded custody | Buy ZEC | Spend as fiat | Cash to bank |
|---|---|---|---|---|
| Zashi | ✅ | ✅ (NEAR Intents) | ❌ (CrossPay → crypto) | ⏳ evaluating Banxa/Coinbase |
| Zodl | ✅ | ✅ (NEAR Intents) | ❌ | ❌ |
| Exchange (Kraken etc.) | ❌ transparent + KYC | ✅ | ❌ | ✅ |
| **Zeal** | ✅ | ✅ | ✅ | ✅ |

The defensible column is **Spend**. Buying ZEC via NEAR Intents is now a
commodity — three products do it identically because it's the same API. The
moat is the float, the compliance underwriting, and the card program, none of
which are an afternoon's integration.

The risk is that ECC ships the Banxa/Coinbase ramp into Zashi and takes the
obvious half of this. That's survivable: a third-party ramp bolted onto a wallet
is a "sell" button, not a card, and it will hand the user the deanonymized exit
with none of the mitigations in §5.2. The bet is that the last mile is a
payments-operations problem and ECC is a cryptography org.

## 8. What to build, in order

**Phase 0 — before any code.** Get a written answer from Rain, Bridge, and
ZeroHash on ZEC-sourced volume. If all three say no, the product is a different
product. This is two weeks and it gates everything.

**Phase 1 — the wallet (8–10 weeks).** Shielded-by-default Zcash wallet,
passkey onboarding, encrypted backup, auto-shield on receive, Z→Z send, fiat-
denominated balance. Ship this alone; it's already the nicest consumer Zcash
wallet if the scanning performance is right.

**Phase 2 — cash out (6–8 weeks).** 1Click integration, pre-flight screening,
refund state machine, KYC at first exit, one corridor end-to-end. Pick one
corridor where ZEC is delisted but stablecoin rails are strong — Argentina,
Turkey, Nigeria, Brazil. Delisting pressure is the customer-acquisition engine:
the users with the sharpest pain are the ones whose local exchange just dropped
ZEC.

**Phase 3 — the card (12+ weeks, partner-gated).** Float management, JIT
replenishment, authorization path. This is the actual product; Phases 1–2 are
how you earn the right to build it.

## 9. Open questions

1. **Float size and funding.** How much pre-funded USDC does a card program
   backed by a volatile asset need, and who capitalizes it? This is the largest
   unknown and it's financial, not technical.
2. **Does the make-whole SLA survive a bad month?** §5.1 promises the company
   absorbs held-funds risk. Model it against a correlated event — a regulatory
   review that holds many intents at once, which is exactly what happened in
   August 2026.
3. **Solver liquidity depth for ZEC.** Consumer cash-out is many small intents,
   not one large one. Does the market-maker side price small ZEC→USDC fills well
   enough to leave a spread, at volume, at 3am?
4. **Is non-custodial survivable through KYC?** Holding is non-custodial; the
   ramp partner will want an account relationship. Where exactly does the line
   sit, and can the "receive and hold privately, KYC only to exit" promise hold
   up legally in each corridor?
5. **Single point of failure.** Everything routes through one intents network
   whose screening layer has already frozen a user for 50 days. What is the
   second route, and is there one?

---

## Sources

- [NEAR Intents Activity Spikes as Zcash's Zashi Wallet Taps It for Private Swaps — CoinDesk](https://www.coindesk.com/markets/2025/10/09/near-intents-activity-spikes-as-zcash-s-zashi-wallet-taps-it-for-private-swaps)
- [1Click Swap API — NEAR Intents docs](https://docs.near-intents.org/integration/distribution-channels/1click-api/about-1click-api)
- [SwapKit Zcash Integration: Cross-Chain Swaps](https://swapkit.dev/blog/swapkit-zcash-integration/)
- [Swapping into ZEC — Zodl Support](https://support.zodl.com/article/26-swapping-into-zec)
- [Zashi Wallet — Electric Coin Company](https://electriccoin.co/zashi/)
- [Zcash Holder Says $589K USDT Stuck on NEAR Intents 50 Days After Zodl Swap — Crypto Times](https://www.cryptotimes.io/2026/09/11/zcash-holder-says-589k-usdt-stuck-on-near-intents-50-days-after-zodl-swap/)
- [Help Needed — NEAR intent swap on Zashi stuck — Zcash Community Forum](https://forum.zcashcommunity.com/t/help-needed-near-intent-swap-on-zashi-stuck/52429)
- [Zcash Delisting Exchanges 2026 Pressures ZEC Coin — CoinGabbar](https://www.coingabbar.com/en/crypto-currency-news/zcash-delisting-exchanges-2026-zec-price-pressure)
- [The Not-So-Private Privacy Coin – 53% of Zcash Transactions Deanonymized](https://finance.yahoo.com/news/not-private-privacy-coin-53-121554795.html)
- [An Empirical Analysis of Anonymity in Zcash — USENIX Security '18](https://www.usenix.org/sites/default/files/conference/protected-files/security18_slides_kappos.pdf)
- [On the linkability of Zcash transactions — arXiv:1712.01210](https://arxiv.org/pdf/1712.01210)
- [Privacy Recommendations and Best Practices — Zcash Documentation](https://zcash.readthedocs.io/en/master/rtd_pages/privacy_recommendations_best_practices.html)
- [Bridge — Stablecoin Infrastructure and APIs for Developers](https://bridge.xyz/)
- [Stablecoin offramps for global fiat payouts — Rain](https://www.rain.xyz/product/offramps)
- [Best Stablecoin Offramp APIs for Fintech 2026](https://eco.com/support/en/articles/15699532-best-stablecoin-offramp-apis-for-fintech-2026)
- [How Zcash Became The Test Case For Privacy Crypto's Comeback](https://yellow.com/research/zcash-privacy-crypto-2026-comeback)
