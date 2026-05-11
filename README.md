# 88th Street - IMC Prosperity 4

This writeup shares the process and strategy ideas behind our IMC Prosperity 4 run.

We finished **53rd globally**, **1st in Singapore**, and **top 5 in Asia**. Our best results did not come from one complicated model, but from repeatedly asking the same questions each round:

- What is the simplest explanation for how this product is generated?
- Is the observed edge structural, or just a backtest coincidence?
- How much inventory can we safely carry before the opportunity stops being worth it?
- Does the strategy still make sense if the next day is slightly different?

The competition rewarded that style of thinking. The products were noisy enough that overfitting was easy, but structured enough that careful market observation produced real edges. Our final strategies were therefore mostly simple: anchored fair values, EMAs, volatility-smile deviations, regime rules, pair spreads, and target-position systems.

## Results Summary (Algorithmic Trading Only) 

| Round | Main focus | Final PnL |
| --- | --- | ---: |
| 1 | Drift-aware market making and anchored fair-value trading | 89,947 |
| 2 | Refined wall-mid execution and inventory control | 83,410 |
| 3 | First option-chain model, volatility-smile research, Hydrogel levels | 13,180 |
| 4 | Regime-based terminal-fair option trading | 79,418 |
| 5 | Broad-universe drift, reversal, pair, and synthetic-value signals | 447,686 |

## Our General Approach

Our process each round followed a tight loop:

1. Inspect raw price paths and order books before writing code.
2. Form a hypothesis about the product's underlying structure.
3. Build the smallest strategy that should profit if that hypothesis is true.
4. Backtest it across available days and reject parameter choices that only worked in narrow regions.
5. Submit to the platform, study fills and final inventory, then simplify or adjust.

The most important habit was separating *prediction* from *execution*. Sometimes the fair value was obvious, but execution sizing mattered more. Sometimes the signal was weak, but the order book gave enough edge to trade it safely. Sometimes a product looked statistically interesting, but the fills were too expensive or unstable to justify adding it.

We also tried to avoid strategies that could only be explained after seeing the result. If a strategy had no clear market-structure story, we treated its backtest PnL with suspicion.

## Algorithmic Challenge

### Round 1: Market Making With Two Personalities

Round 1 had two products: `INTARIAN_PEPPER_ROOT` and `ASH_COATED_OSMIUM`. They looked superficially similar because both were traded through normal order books, but the right mental model for each was different.

#### INTARIAN_PEPPER_ROOT

The key observation was that Pepper Root was not stationary around one clean anchor. Its price drifted over time, but the visible mid price was still noisy tick to tick. A pure fixed-fair market maker would slowly become stale. A pure mid-price follower would chase noise.

Our solution was a stateful fair value:

- start with the previous fair value;
- drift it forward by a fixed amount each tick;
- blend it toward the current volume-weighted mid using an EMA;
- add a small trend bias;
- skew bid and ask quotes based on current inventory.

This worked because it matched the product's behavior. The fair value moved slowly enough to avoid overreacting, but fast enough to follow the upward drift. Inventory skew was important because the position limit was meaningful: being stuck at the limit was equivalent to losing the right to trade the next good opportunity.

The final Pepper Root strategy first took obviously favorable visible liquidity, then quoted passively around the adjusted fair. The exact EMA constant was less important than the design principle: estimate the slow component, trade the noisy deviations.

#### ASH_COATED_OSMIUM

Osmium behaved much closer to an anchored product. The central value was around 10,000, and most of the edge came from taking or making around that anchor.

The first successful version was intentionally simple:

- buy when asks were meaningfully below fair;
- sell when bids were meaningfully above fair;
- quote passively with positive edge;
- flatten near fair when inventory became too one-sided.

The lesson from Osmium was that the competition's matching rules mattered as much as the price series. Orders only lived for one tick, and each tick gave a fresh snapshot of the book. That meant speed was irrelevant, while quote placement and inventory capacity were everything.

Round 1 established our default pattern for the rest of the competition: trade only when the fair-value story is clear, and size orders so the strategy can survive many ticks rather than maximize one tick.

### Round 2: Better Execution, Not a New Theory

Round 2 did not require us to reinvent the strategy. Pepper Root kept the same drift-aware fair-value logic. The bigger improvement was in Osmium execution.

The weakness in the first Osmium approach was that a purely hardcoded anchor ignored useful information in the visible book. The best bid and ask could be thin, but the outer visible levels often behaved like stronger "walls". We therefore introduced a wall-mid estimate and blended it with the 10,000 anchor:

```text
fair = 0.5 * wall_mid + 0.5 * anchor
```

This gave us a fair value that remained anchored, but could adjust slightly when the book structure shifted. It was a compromise between two bad extremes:

- hardcoding 10,000 forever and ignoring useful book information;
- trusting the current mid too much and letting thin quotes move our fair value.

The final Round 2 logic separated taking from making:

- **Taking:** trade aggressively only when visible levels were clearly through fair, or when they helped flatten inventory at fair.
- **Making:** quote just inside meaningful book levels, but only when the resulting quote still had positive edge.

Inventory sizing became more refined. When long, the strategy reduced buy size and increased willingness to sell. When short, it did the reverse. This was not about predicting the next tick; it was about preserving optionality. In a market-making product, unused inventory capacity is a resource.

Round 2 reinforced that small execution details can matter more than adding new signals. The product did not need a fancy predictor. It needed a better fair proxy and cleaner inventory behavior.

### Round 3: Learning the Option Chain

Round 3 introduced `VELVETFRUIT_EXTRACT`, `HYDROGEL_PACK`, and the `VEV_*` option-style products. This was the first round where single-product thinking was not enough. The option prices only made sense relative to the underlying, strike, and time to expiry.

Our first step was to translate the option chain into a common framework. We estimated implied volatility across strikes, fitted a volatility smile, and used that fitted smile to compute theoretical option values.

This gave us a way to ask a better question. Instead of asking whether one option's raw price was high or low, we asked:

```text
Is this option expensive or cheap relative to the rest of the chain?
```

#### Volatility-Smile Scalping

The most natural edge was IV scalping. For each option, we compared the market price to a theoretical price implied by the fitted smile. Then we normalized that difference with a rolling mean, because each strike could have its own persistent bias.

The trading logic was:

- open when the option was far enough from its normalized theoretical value;
- trade the liquid middle strikes first;
- adjust required edge for low-vega options;
- close when the deviation reverted;
- stop opening near the end of the session to avoid bad liquidation.

The "why" behind this strategy was relative pricing. Even if the underlying moved unpredictably, the options should remain internally consistent with each other. When one strike temporarily disconnected from the smile, the strategy could trade the relative mispricing.

#### Velvetfruit Directional Exposure

At the same time, Velvetfruit showed signs of short-term mean reversion and level-based behavior. We did not trust this enough to make it the only strategy, but it was useful as bounded exposure.

The final version kept Velvetfruit risk moderate. It used an EMA-style deviation for mean reversion and a separate absolute-level trigger for a directional long. We deliberately avoided a full delta-hedging framework because spreads and execution costs made that less attractive than it looked on paper.

The directional component was best understood as a controlled hedge against missing the larger underlying move. The IV scalping was the cleaner edge, but the underlying position reduced regret if Velvetfruit's level behavior dominated the round.

#### HYDROGEL_PACK

Hydrogel behaved less like an option and more like a level-driven order-book product. The useful behavior was around broad price bands rather than a theoretical model.

The strategy used wall-aware levels:

- buy when Hydrogel moved into a low zone;
- sell or short when it moved into a high zone;
- take visible book levels that were clearly through wall mid;
- place limited passive quotes around the walls.

Round 3 was not our strongest PnL round, but it was strategically important. It forced us to build the option-chain reasoning that became much more valuable in Round 4.

### Round 4: Terminal Fair Value and Regime Thinking

Round 4 was where the option model became more decisive. The major insight was that pricing options from the current spot was not always the right frame. The data suggested that the opening Velvetfruit level sorted the day into regimes, and those regimes implied different terminal values.

So instead of asking:

```text
What is this option worth right now?
```

we asked:

```text
What will this option be worth at settlement if today's opening regime is real?
```

This shifted the whole strategy. We classified the day as low, mid, or high regime based on the opening Velvetfruit level. Each regime had its own terminal spot estimate and volatility assumption. We then priced the underlying and option chain against that terminal fair.

The final trading rule was simple:

- compute terminal fair value for each product;
- compare fair value to the visible bid and ask;
- enter only when the edge was large enough for that product and regime;
- use wider thresholds where the model was less stable;
- avoid the farthest strikes where the fair-value model was less reliable.

This was a higher-conviction strategy than Round 3 IV scalping. It was no longer just exploiting temporary relative deviations in the smile. It was expressing a view about where the whole chain should settle.

#### Trader-Flow Adjustments

Round 4 also made trader IDs useful. Some named traders appeared informative in Velvetfruit. We tracked selected buyer and seller activity, decayed the signal over time, and capped its impact before adding it to the terminal spot estimate.

The cap was important. Trader flow was a tilt, not the model itself. We wanted it to improve entries when it agreed with the regime view, but not override the structural terminal-fair framework.

#### Simplifying Hydrogel

Hydrogel stayed in the strategy, but we reduced its role. Earlier versions included more passive logic, but the cleanest edge came from level-based directional trades. If a component produced marginal benefit and added extra failure modes, we removed or weakened it.

Round 4 showed that option products can be traded from multiple valid frames. In Round 3, relative smile consistency was the useful frame. In Round 4, terminal settlement value was the better frame.

### Round 5: A Broad Market With Small Limits

Round 5 changed the problem again. Instead of deeply modeling a small number of products, we faced a broad universe with many symbols and low position limits.

With a limit of 10 on many products, the main question was not precise sizing. It was classification:

```text
Should we be long, short, or flat?
```

The final strategy became a target-position engine. Multiple signal families generated desired positions, and execution simply moved each product toward its target using the best visible bid or ask.

#### Drift Buckets

The simplest signals were directional drift buckets. Some products had persistent early-session behavior. Others became more useful later in the day.

We split these into early and late target maps. Each selected product received a target of `+10` or `-10`.

This was intentionally blunt. With such small limits, there was little benefit in pretending we knew the perfect size. The edge was in choosing the right direction and avoiding products whose drift was not stable enough.

#### Reversal Signals

A second group of products showed short-term reversal after large one-tick moves. For these, we stored the previous mid price and compared it to the current mid.

The rule was:

- if price jumped up by more than a threshold, target short;
- if price dropped by more than a threshold, target long;
- otherwise keep the current position.

This worked best when the threshold was product-specific and when early and late regimes were treated separately. A move that was meaningful for one product could be noise for another.

#### Pair Spreads

Some products had stable linear relationships. For these, we modeled a spread:

```text
spread = product_a - alpha - beta * product_b
z = spread / standard_deviation
```

The strategy entered when the z-score moved far enough from zero and exited when it reverted. Again, we used discrete modes rather than continuous sizing. The position limit was small, so the useful decision was whether the relationship was dislocated enough to trade at all.

The important part was not fitting the prettiest regression. It was finding pairs whose residuals behaved consistently enough that the entry and exit bands made sense across days.

#### Pebbles Synthetic Value

The Pebbles complex had an additional structural clue: the group sum was anchored near 50,000. This allowed us to value one Pebbles product synthetically from the others.

For `PEBBLES_L`, the fair value was:

```text
fair(PEBBLES_L) = 50000 - sum(other Pebbles mids)
```

If `PEBBLES_L` was cheap versus this synthetic fair, we targeted long. If it was rich, we targeted short. If the edge reverted, we flattened.

This was one of the cleanest Round 5 ideas because it had a structural explanation. It was not just a price pattern. It came from a relationship among related products.

#### Why the Target-Position Design Worked

Round 5 had too many products for a bespoke strategy per symbol. The target-position design gave us a way to combine heterogeneous signals without creating contradictory execution logic.

Each signal answered only one question: what position do we want? Execution was then centralized and simple. That reduced complexity and made the strategy easier to debug under competition pressure.

Round 5 was our best round by far, producing 447,686 PnL. The biggest edge came from breadth: many small, explainable signals across many products, rather than one oversized bet.

### What Worked Across the Competition

#### 1. Simple Models With Clear Stories

Our best strategies were easy to explain before seeing the final result:

- Pepper Root drifted, so we used a drifting EMA fair.
- Osmium was anchored, so we traded around an anchor and wall mid.
- VEV options were linked by a volatility smile, so we traded relative deviations.
- Round 4 options settled from regime-dependent terminal values, so we priced the chain from terminal fair.
- Round 5 products had low limits, so target direction mattered more than precise sizing.

When a strategy could not be explained this way, we treated it as suspicious.

#### 2. Inventory as a Scarce Resource

Across almost every product, inventory capacity mattered. A profitable-looking quote was less valuable if it trapped us at the limit. This is why so many final strategies had skew, soft limits, stop-opening times, or explicit flattening rules.

The goal was not just to make good trades. It was to remain able to make the next good trade.

#### 3. Robust Parameters Over Peak Backtests

We preferred parameter regions that performed reasonably across multiple days to settings that maximized one backtest. The competition data was too small for aggressive optimization. A peak result with no surrounding stability was usually a warning sign.

#### 4. Logs Over Leaderboard Emotion

The leaderboard only showed the outcome. Logs showed the mechanism. When a strategy underperformed, the useful question was usually:

- Did we get filled where expected?
- Did we end with bad inventory?
- Did passive orders help or hurt?
- Did a signal fire too late?
- Did liquidation dominate the apparent edge?

This kept iteration grounded. We were not just tuning numbers; we were debugging market behavior.

### What We Would Do Better

With more time, we would improve three things.

First, we would build better attribution tooling earlier. Per-product and per-signal PnL would have made some decisions faster.

Second, we would make the research-to-submission pipeline cleaner. Competition code tends to accumulate experiments quickly, and separating live logic from discarded ideas would reduce mistakes.

Third, we would formalize parameter stability checks. We did this manually, but an automated grid-stability report would have helped avoid both overfitting and underconfidence.

Overall, our run was built on a practical principle: find the market structure first, then write the simplest strategy that exploits it. That was enough to finish 53rd globally, 1st in Singapore, and top 5 in Asia.

## Manual Trading Challenge

The manual trading rounds were a separate source of PnL from the algorithmic strategies. These challenges rewarded fast pattern recognition, careful arithmetic, and good judgment under incomplete information.

The PnL numbers below are round-specific and should not be compared directly across rounds, because each manual challenge had its own scoring scale and structure.

| Round | Manual Trading PnL |
| --- | ---: |
| 1 | 87,995 |
| 2 | 200,716 |
| 3 | 68,029 |
| 4 | 25,872 |
| 5 | 94,846 |

### Round 1: An Intarian Welcome

Round 1 was a pair of one-shot opening auctions for `DRYLAND_FLAX` and `EMBER_MUSHROOM`. The auction cleared at a single price, and because we submitted last, our order could affect both the clearing price and our queue priority.

For each candidate clearing price, the useful calculation was:

```text
volume(c) = min(total bids at or above c, total asks at or below c)
```

The exchange then chose the price with the highest `volume(c)`, using the higher price to break ties. After that, we evaluated our own fill and resale value.

We approached it by treating each possible submission as a change to the auction's clearing outcome rather than as a normal fair-value trade. The key was finding the point where adding more size stopped increasing expected profit and started moving the clearing price against us.

This ranked 1st for the round.

### Round 2: Invest & Expand

Round 2 was a budget-allocation problem. We had 50,000 XIRECs to allocate across Research, Scale, and Speed, with each allocation expressed as a percentage from 0 to 100 and total allocation capped at 100.

The public scoring function was:

```text
PnL = Research * Scale * Speed - Budget_Used
research(x) = 200000 * log(1 + x) / log(101)
scale(y) = 7 * y / 100
```

We separated the problem into a private-value part and a competitive part. Research and Scale could be analyzed directly from the scoring rules, while Speed depended on where other teams were likely to cluster. That made the final allocation more like a game-theory decision than a pure calculus exercise.

We used a small scenario model for likely competitor behavior and chose an allocation that was robust across several plausible speed distributions.

### Round 3: The Celestial Gardeners' Guild

Round 3 was a two-bid pricing problem for `ORNAMENTAL_BIO_POD`. Counterparty reserve prices were uniformly distributed from 670 to 920 in increments of 5, and any inventory could be resold the next day at a fair value of 920.

The lower bid was mainly a margin-capture tool, while the higher bid traded off extra fill probability against worse per-unit economics. The second bid also depended on the global average submitted by other teams, so the standalone expected-value answer was only a starting point.

Ignoring the crowd-average adjustment, the shape of the problem was:

```text
E[profit] =
  (920 - low_bid) * P(reserve < low_bid)
+ (920 - high_bid) * P(low_bid <= reserve < high_bid)
```

We used the reserve-price distribution to anchor the bids, then adjusted for the strategic crowd component.

### Round 4: Vanilla Just Isn't Exotic Enough

Round 4 moved from discrete bidding into derivative pricing. The tradable universe was `AETHER_CRYSTAL`, vanilla calls and puts with 2-week and 3-week expiries, and several exotic derivatives written on the same underlying.

The structure looked like an options-pricing problem, but with an important twist: there was no rebalancing after the initial trade. We had to choose positions at the start of the round, hold them to expiry, and get marked against fair values generated from 100 simulations. That made unhedged exposure much more dangerous than it would look in a normal options backtest.

We framed each contract as:

```text
edge = simulated_fair_value - market_price
portfolio_pnl = sum(position_i * edge_i)
```

The hard part was not the formula itself, but making sure the combined book did not accidentally become a large bet on one underlying path or one exotic payoff feature.

We looked for relative mispricings across vanilla and exotic payoffs, but this was our weakest manual execution. Some positions were directionally right, but the combined book still carried too much exposure to model error and payoff-shape risk.

In hindsight, the better version of this approach would have been less focused on individual underpricing and more focused on portfolio-level risk.

### Round 5: Extra! Extra! Read All About It!

Round 5 was a news-trading and portfolio-construction problem. We had access to Ashflow Alpha, a news source for the Ignith exchange, and had to build a one-day portfolio from the products mentioned in the news.

This round was less about closed-form pricing and more about translating qualitative information into expected returns. Each product had a hidden return range and anchor. Participant submissions could move the realized return within that range, so the best trade was not always the most obvious headline reaction. Some news was genuinely fundamental, some was already priced in, and some created crowding risk.

We treated the news as inputs to a portfolio optimizer. The first step was assigning a signed view to each commodity from the text: whether the story implied real demand, forced flow, temporary hype, operational disruption, or existential damage. The second step was sizing those views under the round's fee structure, where concentrated positions became increasingly expensive.

At a high level, the objective was:

```text
expected_pnl = sum(position_i * expected_return_i) - fee(position_i)
```

with a quadratic fee that made oversized single-name bets expensive.

We deliberately allowed the optimizer to leave budget unused when the marginal trade no longer paid for its fee.
