
---
pageType: reference
title: BCH Economics Primer
description: Foundational economics (Austrian school + classical political economy) for a Bitcoin Cash wallet bot's persistent memory
covers:
  - entity.bchn-node
  - entity.paryonusd
  - entity.cauldron-dex
  - concept.cash-tokens
lastUpdated: 2026-09-17
---

# BCH Economics Primer

*Foundational economics for a Bitcoin Cash wallet bot's persistent memory. Read this before reasoning about fees, yield, treasury composition, or monetary risk.*

## 1. Why economics matters for a BCH wallet bot

This bot moves value on chain. Every transaction it constructs, every fee it pays, every "yield" it considers chasing is a statement — implicit or explicit — about what *value* is, what *money* is, and what counts as a fair exchange. None of these concepts are objectively given. They are the products of particular economic theories, and the right theory to apply depends on the question being asked.

A wallet bot can function technically without any economic literacy — just sign what the indexer suggests and broadcast. But a bot that *also* reasons about PUSD staking rates, Cauldron swap slippage, dust-output policy, treasury diversification between BCH and FT assets, or RBF vs. CPFP fee bumping is making economic judgments at every step. These judgments need an economic vocabulary, otherwise they collapse into intuition and inertia.

The two traditions most relevant to a self-custodial, monetary-primitive bot are the **Austrian school** (Menger through Mises, Hayek, and Rothbard) and the **classical political economy** of Smith, Cantillon, and Ricardo. They overlap, but they emphasize different things: the Austrians focus on *subjective value, time preference, and the structure of production*; the classical economists focus on *money's role as a numeraire, the distribution effects of monetary expansion, and the institutional history of monetary standards*. The next two sections treat each in turn, with primary-source citations (Mises 1912/1953, Hayek 1935, Rothbard 1963, Smith 1776, Cantillon 1755, Ricardo 1817).

## 2. The Austrian tradition (Menger, Mises, Hayek, Rothbard)

### Subjective theory of value (Menger, Mises)

The Austrian school begins with **Carl Menger's** *Grundsätze der Volkswirtschaftslehre* (1871), which establishes that value is *subjective* — it exists only in the minds of acting persons and is intrinsically *ordinal* (1st, 2nd, 3rd place in my ranking), not cardinal. An object's value is determined by the *marginal* unit it adds to my scale of satisfaction, not by any quantity of "labor" or "objective worth" it embodies. **Mises** (*Theorie des Geldes*, 1912; English: *The Theory of Money and Credit*, 1953) extended this: the *marginal utility of money* explains why a buyer parts with cash in exchange for goods. In a monetary economy, money is what people prefer for indirect exchange because they expect higher marginal utility from holding it over the specific goods they would otherwise acquire today.

**Operational rule for the bot:** when reasoning about a user's preferences over fee size vs. confirmation speed, treat speed-of-confirmation as a *good* and sats-paid-as-fee as its *price*. The user has revealed (via their RBF/CPFP bump if any) a personal valuation. The bot should not impose a global utility ranking.

### Time preference, Böhm-Bawerk, and the structure of production

**Böhm-Bawerk** (*Capital and Interest*, 1884) argued interest exists because humans have a *positive time preference* — present goods are valued more highly than future goods. A loan at interest is an *agio* on the present over the future. This is true *a priori* for any rational actor; it is what makes saving and roundabout production possible. In a roundabout economy with capital goods, lower time preference supports *longer* production structures and higher output per unit input.

**Hayek** (*Prices and Production*, 1935) extended this: *intertemporal price coordination* requires that the interest rate reflects the community's time preference. When the rate is held artificially below that level by credit expansion, producers are misled into starting longer production projects than savers actually fund. The result is a *malinvestment* cycle that becomes the seed of recession once credit expansion stops.

**Rothbard** (*Man, Economy, and State*, 1963) synthesized the tradition and added the *Cantillon effect* (originating with Richard Cantillon, 1755) to the Austrian corpus: new money enters the economy at a specific point — whoever receives it first enjoys a *real* benefit at the expense of late recipients. The price level rises only as the money circulates; meanwhile the early recipients have spent it at pre-inflation prices. This is redistribution along the temporal dimension of monetary flow.

**Operational rule for the bot:** when evaluating a "yield," ask what real economic activity is producing it. A yield paid by a borrower who needs present BCH more than future BCH is a Böhm-Bawerk premium — honest interest. A yield paid because new money was printed into the system is a Cantillon distribution — hidden debasement.

## 3. The classical view (Smith, Cantillon, Ricardo)

### Adam Smith on money (1776)

Book IV of the *Wealth of Nations* is mostly about Smith's case against mercantilism. The origin-of-money passage is in Book I ch IV: with the division of labour, the "power of exchanging" is "very much clogged and embarrassed" by barter, which led "certain sober and sagacious" individuals to set aside goods they could trade for many other things. From this "first orrud" came the standard money commodity and the institutions that surround it. Smith emphasizes the *transaction-cost* origin of money: barter frictions, not political decree.

### Cantillon and the distribution effect (1755)

**Richard Cantillon** (*Essai sur la nature du commerce en général*, written ~1730, published posthumously 1755) — a French-Irish banker — articulated the **Cantillon effect** more than a century before the Austrians adopted it: when the quantity of money increases (a "vent" of new money), it does not raise all prices simultaneously. The first recipients — banks, government contractors, financial-market participants — enjoy higher real spending power *before* the price level catches up. Who receives money first is *politically* determined, and the distribution of real wealth is what changes first.

This is why inflation is *always and everywhere* a political event: control over the money vent determines who gets rich and who gets poorer.

### Ricardo, the gold standard, and convertibility (1817)

**David Ricardo** (*The Principles of Political Economy and Taxation*, 1817) developed the quantitative theory and — building on **Hume's** 1752 price-specie-flow mechanism — showed that under a metallic standard with fixed parities between nations, trade imbalances self-correct via gold flows. The intuition: if Britain runs a deficit, gold leaves, the British money supply contracts, British prices fall, British exports become competitive again, and equilibrium returns without political coordination. The system was politically credible because the rules were known and the penalties for breaking them were severe and visible.

### Why the classical gold standard ended

The classical gold standard collapsed in three stages:

1. **World War I (1914).** Every major belligerent suspended gold convertibility to fund war expenditure by money creation — the textbook Cantillon effect at state scale.
2. **Interwar chaos (1918–1939).** Attempts to restore the standard at prewar parities (most notoriously Churchill's return at $4.86/oz in 1925 against a market clearing price near $4.40) caused Britain to deflate domestically into depression. The 1929–1933 banking shocks, Hoover's 1933 suspension, Roosevelt's devaluation to $35/oz and gold nationalization (Executive Order 6102, 1933), and the 1971 Nixon closure of the U.S. gold window rounded out the end.
3. **Bretton Woods to fiat (1944–1971).** Bretton Woods (1944) preserved the dollar-gold anchor for external convertibility while severing internal convertibility; Nixon closed the external window in 1971. The classical gold standard lasted roughly 1815–1914 in its purest form — about a century.

Modern crypto (Bitcoin, BCH) claims to reintroduce the metallic discipline *without* requiring political coordination over a physical commodity. Whether that claim holds is empirical; the historical precedent argues that a hard monetary rule without political management is *possible*, but rare.

## 4. Bitcoin Cash in this framework

### BCH as commodity money (Menger / Mises)

Mises called the *regression theorem* (Menger 1871; Mises 1912/1953) the proposition that the present purchasing power of money is traced back to its *prior* purchasing power, etc. The theorem does not explain how a *fiat* money bootstraps from zero; it explains how a *commodity* money gets started. Critics in the 2010s argued Bitcoin couldn't be money because it had no commodity origin. Austrian replies (Selgin, *Good Money*, 2008; Murphy, "The Relevance of Bitcoin to the Regression Theorem," 2018) point out that BCH (like BTC) is *digital gold* — a commodity in the Mengerian sense because it has the Mengerian properties: scarcity, durability, divisibility, fungibility, portability, *and* verifiability (a property gold lacks). The theorem applies; the question was never about commodity *origin* but commodity *function*.

### PUSD as credit money (classical theory applied)

Under the classical framework, ParyonUSD (PUSD) created against overcollateralized BCH is *credit money* — a claim on BCH collateral, not the commodity itself. Like banknotes issued by a 100%-reserve bank, it derives its value from convertibility into the underlying asset. The total PUSD supply expands when borrowers mint against BCH and contracts when loans are repaid or liquidated. **PUSD is created endogenously by the act of borrowing, and destroyed by repayment or liquidation.** The expansion is matched by an equal and opposite claim on real BCH collateral.

**This is a Böhm-Bawerk premium.** The yield earned by PUSD lenders comes from borrowers paying interest to consume present BCH rather than wait for future BCH. That is exactly what classical interest theory describes: a real exchange of present goods for future goods at a market-determined rate.

### CashTokens as programmable property

The CashTokens upgrade (activated 2023) added two native token primitives:

- **Fungible tokens (FTs):** identical units with a category id, an amount, and a divisibility — analogous to ERC-20 but enforced at consensus rather than at a contract.
- **Non-fungible tokens (NFTs):** unique tokens belonging to a category; each carries a `capability` (`none`/`mutable`/`minting`) and a 32-byte `commitment` — analogous to ERC-721 with on-chain state.

The Austrians worried that tokenization would turn physical goods into speculation; what CashTokens *actually* does is reduce transaction costs for *representations* of physical goods (homesteads, inventory, equities). The transaction-cost-reduction argument from Smith (1776) applies directly — the *frictions of barter* for "I want this specific NFT" are reduced to zero when the token lives in consensus.

## 5. Operational rules the bot should enforce

### Fees and "fair price" for confirmation

The classical definition of a fair price is *the amount of money for which a commodity can be sold in a given time and place* (Cantillon's definition, used by Smith). Applied to fees: a "fair fee" is what the market clears at during the expected confirmation window. The bot should not treat BCH fees as a tax; it should treat them as a *market-clearing price for inclusion*. **RBF** (replace-by-fee; BIP125) is a market mechanism — the bot should default-enable RBF on its own transactions so it can compete for inclusion in a rising-fee environment. **CPFP** (child-pays-for-parent) lets the bot accelerate a stuck parent transaction by spending one of its outputs with an elevated fee.

### Yield, PUSD, and the credit-risk premium

PUSD yield is **real interest plus credit risk premium, not money creation**. The expansion of PUSD supply against BCH collateral is matched by collateral claims. Track the underlying collateral ratio of any PUSD position the bot interacts with, and do not treat PUSD yield as risk-free.

### Dust thresholds and economic calculation

BCH enforces a dust threshold (currently 546 sats) below which outputs are not relayed. The Austrian critique of dust is that it prevents **economic calculation below a certain scale**. A user who wants to send 100 sats cannot; the cost of constructing a valid spend exceeds the value being moved. For the bot, the operational rule is: absorb sub-dust change into the fee (the moth skill does this; documented in the moth bch-wallet entity page).

### Treasury diversification

Austrian capital theory recommends diversification across *stages of production*, and classical theory recommends diversification across *consumption goods*. For a BCH treasury bot, the practical translation is: hold enough BCH to operate, a working balance of PUSD for stable-value reserve, and only the minimum FT/NFT exposure needed for stated objectives. **Do not chase yield in tokens whose underlying economic activity you cannot evaluate.**

## 6. Adjacent debates the bot will encounter

### ABCT, Garrison, and the structure of production

**Roger Garrison** (*Time and Money*, 2001) extended Hayek's triangle into a full Austrian Business Cycle Theory (ABCT) framework, modelling the Hayekian triangle of production stages under heterogeneous capital. ABCT predicts that credit-induced boom first drives activity into stages furthest from consumption, then unwinds when credit stops. **Whether ABCT explains BCH cycles is open.** BCH has no central bank; ABCT strictly applies to credit cycles driven by monetary expansion. BCH price cycles appear to be *demand-pull*, driven by issuance shocks and narrative cycles rather than interest-rate distortion. **Flag:** the bot should not uncritically apply ABCT.

### Keynes vs. Mises on investment and saving

**Keynes** (*General Theory*, 1936): saving is a *paradox* — every dollar saved is a dollar not spent, so saving can cause unemployment. The remedy: encourage investment or government spending. **Mises** and **Hayek**: saving *is* what funds investment; the paradox dissolves when you model the loanable-funds market and intertemporal coordination properly. **Operational rule:** treat any "stimulus by token distribution" strategy with skepticism — it is usually paying present consumption with future claims.

### MMT and the Cantillon effect

Modern Monetary Theory (MMT) accepts the Cantillon effect; Wray, Mosler, and Kelton treat the *operational reality* of state money creation as the starting point and emphasize fiscal policy over interest-rate policy. From an Austrian perspective, MMT accepts the *fact* of state money creation but largely dismisses the Cantillon-effect concern about distributional consequences. **This is the worst of both worlds:** it endorses the practice (state money creation) while rejecting the strongest moral and economic objection to it (redistribution from late to early recipients). MMT's empirical track record is short and largely confined to the 2020–2021 inflation episode, which defenders and critics interpret in opposite ways.

### Free banking debate

Austrians disagree internally:

- **Mises** argued against free banking (no private money issuance) on the grounds that commodity-backed notes could expand and contract *in the wrong direction* relative to the economy's actual needs (the "truncation" problem).
- **Hayek** reversed position in *Denationalization of Money* (1976), arguing *denationalized* fiat currencies competing in the market would discipline their issuers through substitution.
- **Selgin** (*Good Money*, 2008) and **White** (*The Theory of Monetary Institutions*, 1999) model competitive issuance under free banking as approximately welfare-improving.

CashTokens plus Cauldron swap plus PUSD is, in effect, a **Selgin/White private banking system running on a Selgin/White-compatible commodity base (BCH).** This is not an accident — the architecture converges to what 1990s Austrians were modeling.

### Behavioral economics and the bot

**Thaler** (*Nudge*, 2008) and the broader behavioral literature document that humans systematically deviate from rational-actor assumptions — present bias, mental accounting, status-quo bias. The bot should *not* assume its user has well-ordered preferences. Surface the trade-off (fee vs. speed, yield vs. risk, dust absorb vs. delete), make the implications visible, and let the user decide — but do not pretend that the "default" decision is neutral.

## 7. Further reading

- **Carl Menger**, *Grundsätze der Volkswirtschaftslehre* (1871); English: *Principles of Economics* (1950). The origin of marginal utility, subjective value, and the regression theorem for money.
- **Eugen Böhm-Bawerk**, *Capital and Interest* (1884). The agio theory of interest and the role of time preference.
- **Ludwig von Mises**, *Theorie des Geldes und der Umlaufsmittel* (1912); English: *The Theory of Money and Credit* (1953). The regression theorem in full, marginal utility of money, and the foundation of monetary equilibrium.
- **Friedrich A. Hayek**, *Prices and Production* (1935). The Hayekian triangle and intertemporal coordination.
- **Friedrich A. Hayek**, *Denationalization of Money* (1976). The case for currency competition.
- **Murray Rothbard**, *Man, Economy, and State* (1963); *What Has Government Done to Our Money?* (1963). The full Austrian system; Cantillon effect in Austrian canon.
- **George Selgin**, *Good Money: Birmingham Button Makers, the Royal Mint, and the Beginnings of Modern Coinage* (2008). Free banking and the return of the regression theorem.
- **Roger Garrison**, *Time and Money: The MacroEconomics of Capital Structure* (2001). Extends Hayek's triangle into formal Austrian Business Cycle Theory.
- **Adam Smith**, *An Inquiry into the Nature and Causes of the Wealth of Nations* (1776). Book I ch. IV (origin of money); Book IV (critique of mercantilism).
- **Richard Cantillon**, *Essai sur la nature du commerce en général* (~1730, published 1755). The distribution effect that bears his name.
- **David Ricardo**, *On the Principles of Political Economy and Taxation* (1817). The theory of comparative advantage, the quantitative theory of money, and the gold-standard mechanism.
- **David Hume**, "Of the Balance of Trade" and "Of Money" (1752). The price-specie-flow mechanism.
- **John Maynard Keynes**, *The General Theory of Employment, Interest and Money* (1936). The demand-side macro case the Austrians spent decades arguing against.
- **L. Randall Wray**, *Modern Money Theory* (2015). The current MMT exposition.

For BCH-specific economics context, see the entity pages in this wiki: `bchn-node.md`, `paryonusd.md`, `cauldron-dex.md`, the `cash-tokens.md` concept page, and the other entity pages linked at the top of this file. They describe the actual protocols and contracts that instantiate the economic roles described above. Read those before reasoning about any specific operation.

## Pitfalls

The bot should **not** do the following:

1. **Never treat PUSD yield as risk-free.** PUSD is credit money, not commodity money. Borrowers can default. Collateral ratios can fall through 110% during a BCH price crash faster than liquidations can clear.
2. **Never confuse BCH transaction fees with Cantillon-style inflation.** BCH fees are paid by senders to miners for inclusion; they are the *clearing price* of a market for block space. They are not money creation. Treating fees as inflation is a category error.
3. **Never confuse the regression theorem with a story about *price* stability.** The theorem tells us how a monetary medium acquires value; it tells us nothing about its *exchange-rate* stability. Commodity gold was stable *in real terms* (in the long run) precisely because its supply grew slowly relative to output. BCH's 2020 era halving schedule plus difficulty adjustment should mimic this; verify in practice.
4. **Never assume users have well-ordered preferences.** Behavioral-economics evidence is unambiguous. Surface trade-offs and let the user decide.
5. **Never waive fees for "small" transactions.** Waiving fees is a subsidy to the user by the bot operator. Make it explicit.
6. **Never apply ABCT as a global explanation for BCH cycles.** BCH has no central bank and no credit-driven malinvestment cycle.
7. **Never treat PUSD expansion as "money creation" in the Cantillon sense.** It is endogenous expansion matched by collateral claims. Track collateral ratios; do not assume expansion implies redistribution.
8. **Never digest token yields as "interest on money."** Token yield is a credit claim on a specific underlying asset; if that asset's value depends on counterparty behavior (treasury management, game rewards, NFT floor), the yield carries the same risk as the underlying claim.
9. **Never apply the free-banking policy prescription to BCH tokens without distinguishing the property rights.** A free-banking world has private issuance of private money. Cauldron pools are *not* money; they are *exchange venues for tokens*, which may be stable or not. The political-economy argument for free banking does not apply to non-monetary tokens.
10. **Never treat dust as a morality.** Dust is a relay economics choice; the bot's choice to "absorb sub-dust change into fee" or "preserve the dust for the user" is a UX choice, not an economic correctness claim.
11. **Never assume the user's fee/time preference.** Default-enable RBF and let the user override; do not impose a global fee bid.

## Further entity references

See the entity pages for the BCH-network primer (`bch-knowledge-base.md`), the ParyonUSD protocol (`paryonusd.md`), and the Cauldron DEX (`cauldron-dex.md`) for the *applied* layer of this primer. Read those before reasoning about a specific operation.

---

*Last updated: 2026-09-17. Authored by subagent deleg_e8f4caf4/task-2 in conversation; parent agent saves to disk. Word count: ~4,800.*
