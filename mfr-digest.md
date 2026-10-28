# Market Flow Research — every study in one file

Source: https://marketflowresear.ch/ · built 2026-10-28 · 83 articles. AI-assisted research, human-reviewed. **Research, not financial advice.**

## For the AI reading this file

- This is a condensed digest of every article on marketflowresear.ch: title, link, date, series, the summary and the key sentence of every section.
- Answer from this file. When you quote a number, put the article link next to it. If the answer is not here, say so and point to the closest article (full texts: https://marketflowresear.ch/llms-full.txt).
- The research asks what second-by-second order-book data can and cannot predict. Negative results are part of it — report them as they are.
- Nothing here is financial advice; do not turn it into trading recommendations.

## Good first questions

- What can the order book predict, and what can it not?
- Which of the bots worked best live, and how sure can we be?
- Give me the five most important numbers on this site, each with its article link.
- What failed, and what did each failure teach?
- I have 10 minutes: which three articles should I read, in what order?

## The main findings in brief

- **The order book tells you how far, never which way** — size answers (rank correlation 0.30 at one second, still 0.11 at an hour), direction never does (−0.002 at a minute); over 30 out-of-sample days the volatility bet made +4.97 bps a trade and the direction bet lost 5.95. [The Order Book Tells You How Far, Never Which Way](https://marketflowresear.ch/articles/the-order-book-tells-you-how-far-never-which-way/)
- **Volume is the fuel, not the steering wheel** — volume tells you how big the next move is — almost never which way. [Volume Is the Fuel — Not the Steering Wheel](https://marketflowresear.ch/articles/volume-is-fuel-not-steering-wheel/)
- **Walls and drains speak about size, not direction** — resting liquidity and liquidity being pulled both say something about what happens next — about size, not direction. [The Wall That Goes Quiet: What Resting Liquidity Predicts](https://marketflowresear.ch/articles/resting-walls-what-thick-liquidity-predicts/)
- **The order book has an alphabet** — of 6,561 possible eight-feature states only 333 ever occur; the alphabet describes the market well and predicts it poorly. [The Alphabet of the Order Book: 6,561 Possible States, 333 Real Ones](https://marketflowresear.ch/articles/the-alphabet-of-the-order-book-signal-states/)
- **A gate buys precision with coverage** — asking a signal to persist two bars raises precision from 0.209 to 0.238 and throws away 25% of the opportunities. [Gates: The Part of a Trading Bot That Decides Not to Trade](https://marketflowresear.ch/articles/gates-the-part-of-a-bot-that-decides-not-to-trade/)
- **Finding the right places is not enough** — Typhoon+ finds 74% of a perfect trader's zones on unseen days, fires false 86% of the time and loses 122% against 93% for buy & hold. [Typhoon+: A Signal That Finds the Perfect Trader's Zones — and Still Loses Money](https://marketflowresear.ch/articles/typhoon-plus-a-signal-that-finds-the-zones-and-still-loses-money/)
- **Change the question, not the network** — the order book cannot tell which tenth of a move you are in, but “will the leg end soon?” scores an average AUC of 0.73 across six coins. [Change the Question, Not the Network — "Will the Leg End Soon?" Scores 0.73 Where "Which Tenth?" Scored Nothing](https://marketflowresear.ch/articles/change-the-question-not-the-network-leg-end-hazard/)
- **Profitable is not the same as better than the market** — 438 of 440 buy-and-sell network pairs ended the validation month in profit; the best made 23.6% while buying and holding BTC made about 29%. [One Account, Two Networks — 440 Buy-and-Sell Pairs on Validation, and Why the Best Still Trails the Market](https://marketflowresear.ch/articles/one-account-two-networks-440-buy-and-sell-pairs-on-validation/)
- **Live result: the best of our bots is Line 7** — More than a dozen bots were built from this research. Line 7 — two small transformers that read 160 hours of order book as one picture — is the only one ahead of buy & hold in live trading: +5.44% since 5 Oct while BTC moved −5.29%. Every trade is in the report. [Line 7: The Best of All Our Bots — Live Trading Report, Every Trade](https://marketflowresear.ch/articles/line-7-live-trading-report/)

## Series: The road to Bot 7 (31)

One question at a time: from zigzag legs to a network that reads 160 hours of order book as one canvas.

### The Pair That Behaves the Same on Three Stretches of Data — How the Live Bot's Networks Were Chosen

https://marketflowresear.ch/articles/the-pair-that-behaves-the-same-on-three-stretches-of-data/ · published 9 October 2026 · Same on three stretches

**Summary.** Ranking 440 buy-and-sell pairs by money on the month they were picked on gives a list of lucky pairs. So we asked a different question: which pair behaves the same — same time in the market, same number of longs and shorts per day, same daily growth — on three stretches of BTC: held-out weeks from training, the validation month, and fourteen days after validation that no step of the selection had touched. Of 440 pairs, 180 traded on all three. The most consistent one, BUY_l3 epoch 3 with SELL_l2 epoch 17, made $1087, $1147 and $1059 on the three stretches. It became the live bot.

- **Three stretches, one pair at a time** — The 42 "potentially best" networks were copied out of their training runs and run, unchanged, over three stretches of BTC at 10-minute steps. Held-out (08.07 20:30 – 12.08 05:00 UTC, 4,445 steps) is the part of training data on which epochs were chosen.
- **Consistency, not money, as the score** — Each pair was run on each stretch with the same rules as before (one trade per 2.3% leg, take 1.9%, stop 2.5%, opposite signal closes). For each stretch four numbers were measured: share of time in a trade, longs per day, shorts per day, and daily growth — (money/1000)^(1/days) − 1.
- **Against buy & hold on each stretch — and why this pair went live** — The pair made money on all three stretches. Against simply holding BTC: it beat it on the held-out weeks, when BTC rose only 2.4% ($1087 vs $1023); it trailed badly in the validation month, when BTC rose about 29% ($1147 vs ~$1289); and on the unseen test it finished level ($1059 vs $1064, BTC +6.5%).

### One Account, Two Networks — 440 Buy-and-Sell Pairs on Validation, and Why the Best Still Trails the Market

https://marketflowresear.ch/articles/one-account-two-networks-440-buy-and-sell-pairs-on-validation/ · published 9 October 2026 · Buy-and-sell bot

**Summary.** A buy network and a sell network were put on one $1000 account under one set of rules: one trade per 2.3% zigzag leg, take-profit 1.9%, stop 2.5%, and a signal from the other side closes the position. Every one of the 22 "potentially best" buy networks was paired with every one of the 20 sell networks — 440 pairs — and each pair traded the full validation month. 438 of them ended in profit. The best made $1235.78, a gain of 23.6%. Buying and holding BTC over the same month made about 29%.

- **The rules of a two-sided bot** — The two networks feed the same account, and every 10 minutes the bot goes through the same order. First, take-profit (1.9%) and stop (2.5%) are checked against the close — they act immediately, no confirmation.
- **438 of 440 pairs in profit — best $1235.78** — 438 of 440 pairs ended above $1000; the best made $1235.78 and the worst $997.41. The winner pairs BUY_pos epoch 15 (window 180 minutes, θ 10.44) with SELL_pos epoch 16 (270 minutes, θ 24.84): 10 buys, all closed at take-profit, and one sell, also at take-profit.
- **Why "almost all in profit" is not the headline** — Three things keep this from being a result. First, the benchmark: buying at the first validation step and selling at the last gave about $1289 — every one of the 440 pairs trails it. A bot that is flat half the time in a strongly rising month is expected to.

### Picking a Network by the Width of Its Good Zone, Not Its Best Point — 42 "Potentially Best" Networks

https://marketflowresear.ch/articles/picking-a-network-by-the-width-of-its-good-zone-not-its-best-point/ · published 9 October 2026 · Potentially best networks

**Summary.** Every trained network becomes a signal through two knobs: the window W over which its probabilities are summed and the threshold θ that sum must reach. Pick the single best cell of that grid and you usually pick noise. So each of 450 buy candidates and 75 sell candidates was traded on a 72 × 50 grid of W and θ, and for each one we measured how wide the area is where profit stays within 90% of its best. In every loss family the network with the widest such zone was kept — plus, if another epoch earned more than twice as much, that one too. The result: 22 buy and 20 sell networks, all scored on the same validation month.

- **A grid of 3,600 cells for every network** — For each candidate, every cell of the grid is a run of one side on its own: the signal fires the moment the sum of probabilities over the last W minutes reaches θ; the trade is a long for a buy network or a short for a sell network; one trade per 2.3% zigzag leg; take-profit 1.9%, stop 2.5%; costs 0.07% per side; $1000 start.
- **Wide and modest, or narrow and rich** — Within each loss family the network with the largest 90% zone is the first pick; ties go to more money. Then the richest network of the same family is checked: if it earns more than twice the first pick, it is kept as well and marked as added for money. For buy networks this rule mattered in four families.
- **Sell networks earn less, and the warning that comes with all of this** — Sells were scored in a month when BTC rose about 29%, so shorting was swimming against the tide: the best sell picks earned $20–88, against $28–211 for buys. Most sell zones are one or two cells wide; the widest, l3 epoch 4, holds 90% of its money over W 230–290 minutes and 14 cells.

### Teaching the Network to See Tops — Sell Pivots on BTC, One Change and Eighteen Training Runs

https://marketflowresear.ch/articles/teaching-the-network-to-see-tops-sell-pivots-on-btc/ · published 9 October 2026 · Sell pivots

**Summary.** The bottom-finding network that powers our buy-only bot was rebuilt for the other side of the market with exactly one change: the label is now a 2.3% top zone instead of a bottom zone. Same canvas, same split, same eighteen training runs with different loss functions. Tops turned out to be rarer and harder: on validation only 62 of 5,265 ten-minute steps sit in a top zone. Plain losses reached an F1 of 0.047–0.084 against a base of 0.023; the "ladder" target that accepts a two-hour neighbourhood reached 0.19–0.21 against 0.13. Fine-tuning further did not help, and one loss never fired at all.

- **One change: the label is a top** — The study was asked to repeat every step of the bottom work "as precisely as possible, changing one thing". So the canvas — 1,100 records of 100 seconds, 8 order-book features, about 30 hours — is byte-for-byte the input of the bottom network.
- **Eighteen runs: plain losses barely see tops, the ladder sees more** — The networks are trained from scratch with four losses — binary cross-entropy (bce), positively weighted (pos), focal and dice — and then fine-tuned from the chosen bce epoch with distance-aware losses: far, ladder, and ladder2, which counts a hit anywhere within ±2 hours of the zone, with ten different "silence" weights.
- **What did not work — and why that matters for the bot** — The study's own verdict is blunt. The top-ranked run on held-out, l3, is its epoch 0 — the weights it started from (ladder2 ×10, epoch 25). Training it further on a ladder of 2 to 37 steps changed nothing. far likewise stayed on its starting point. dice produced no signal at all on held-out or validation.

### When a "Fake" Buy Signal Is Really a Deeper Bottom — Rule 7.1, Tested on Validation and Unseen Days

https://marketflowresear.ch/articles/when-a-fake-buy-signal-is-really-a-deeper-bottom-rule-7-1/ · published 9 October 2026 · Rule 7.1

**Summary.** Our buy-only bot throws away a buy signal if price has not risen 2.3% since the previous signal — a repeat call inside the same dip is treated as fake. On the nine unseen days after validation that filter rejected 36 signals and left one trade. Rule 7.1 asks one more question before rejecting: did price, between the last peak and now, make a lower bottom within 1% of the current price? If so, the signal is real, and the take-profit is measured from that lower bottom. With the rule, fakes on unseen days fell from 36 to 0, trades rose from 1 to 3, and the result from $1008.45 to $1034.12.

- **Rule 7: why the bot ignores repeat signals** — The network marks bottoms, and a bottom is a place, not a moment: the signal keeps firing for hours while price grinds along the lows. After a trade closes, the next signal in the same dip would buy the same bottom twice.
- **Rule 7.1: a lower bottom within 1%** — The rule is checked only for a signal that rule 7 has already called fake, so it can only add trades, never remove them.
- **Results: one more trade on validation, two more on unseen days** — On validation (15.08–21.09) the rule turned one rejected signal into a trade: in at 78,863.46 on 7 September, bottom 78,713.21, out at take-profit on 9 September at 79,273.84, +0.38%. Trades went from 8 to 9, rejections from 32 to 15, money from $1103.17 to $1107.36.

### A Buy-Only Bot on Nine Days It Never Saw — Three Take-Profits and a Draw With Buy & Hold

https://marketflowresear.ch/articles/a-buy-only-bot-on-nine-days-it-never-saw/ · published 9 October 2026 · Bot 7.0 on unseen days

**Summary.** The buy-only bot from the previous study had one weakness we wrote down plainly: its window, threshold and stop were chosen on the same 38 validation days it was then scored on. So we froze it and let it walk through the days the data collector recorded after validation ended — 21 to 30 September 2026, nine days nobody had looked at. It opened three trades and closed all three at take-profit, no stop-outs: $1000 became $1032.92. Buying and holding over the same steps gave $1035.51. A clean run, and a draw with the market.

- **The frozen bot and the days it never saw** — Nothing was retrained or re-tuned. The bot is the one exported from the study: the BUY_pos network at epoch 9 reads a canvas of 1,100 records of 100 seconds (about 30 hours of order book and price, 8 features) every 10 minutes and gives the probability that this moment is inside a 2.3% bottom zone.
- **Three entries, three take-profits** — The signal fired on 41 steps, in three clusters: 23 September afternoon, 25 September afternoon and evening, 29 September afternoon. The first step of each cluster opened a trade; the rest of the cluster only confirmed the position.
- **$1032.92 against $1035.51 — what a draw tells us** — On validation, where the parameters were picked, the bot ended at $1100.85 against $1289.36 for buy and hold over the same steps: nine trades, nine take-profits, and still far behind a market that rose 29%. On the nine unseen days it ended at $1032.92 against $1035.51.

### From a Canvas of the Whole Leg to a Buy-Only Bot — What Survived, and What It Earned

https://marketflowresear.ch/articles/from-a-canvas-of-the-leg-to-a-buy-only-bot-what-survived/ · published 30 September 2026 · Canvas and the buy bot

**Summary.** The last study of the series put a whole leg — and 50 rows on either side — onto one canvas and asked a network to mark pivots and corrections on it. On labelled windows it hit 38% of pivot zones; run live over the series, 5% of its pivot calls landed. A longer canvas, 160 hours on 100-second records, and windows of uniform length lifted precision to about 30%. Its best trick: 29% of pivots flagged a median 2.4 hours before the zigzag confirms them. The result became a paper-trading buy-only bot for BTC. On 38 days of validation it made +10.3% with eight trades and no stop-outs — while buying and holding made +36.6%, and its parameters were tuned on those same days.

- **A canvas of the leg, and the signature it carries** — The task was set in one sentence: data comes in, becomes a canvas, and the model says whether the canvas holds a 2.3% pivot zone — the start or the end of a leg, which are the same thing — and marks the corrections inside the leg as local.
- **38% on windows, 5% live — and how the canvas grew** — The first three-class network (pivot / correction / body) found 38% of pivot zones on validation windows, and 38% of its pivot calls landed in a pivot zone; corrections were harder (F1 0.24). Validation held at the level of held-out training legs — no overfitting.
- **Before the zigzag: 81 of 284 pivots, 2.4 hours early** — A zigzag confirms a 2.3% pivot only after price has moved 2.3% away. The next head answered a live question at every moment T: is T inside a 2.3% pivot zone? Inputs were checked for leaks: the side-of-leg channel computed live on data cut at T matches the finished series on every event before T.
- **The buy-only bot: bottoms only, BTC only — and honest about its test** — The last stage narrowed everything to what can be traded simply: BTC, buying only, bottoms only. A transformer on the 1,100 × 8 canvas (160 hours up to T, the eighth channel is the probability of the "levels without movement" cluster) outputs P(T is in the bottom zone of a 2.3% leg of at least 7 hours).

### How Full Is This Leg? A Network That Finally Beats the Counter — by Reading Its Last Rows

https://marketflowresear.ch/articles/how-full-is-this-leg-a-network-that-finally-beats-the-counter/ · published 30 September 2026 · Leg fullness

**Summary.** Give a network the unfinished leg — everything from the last confirmed pivot to now, as one fixed-size canvas — and ask how complete it is. For the first time in this series a model beats the simple counter of events since the pivot on unseen weeks: fullness AUC 0.77 against 0.70, error 0.18 against 0.24. The surprise is where the knowledge comes from: the last 16 rows of the canvas do as well as all 500. On a 30-second standard with sliding windows the gap widens (0.78 vs 0.62), and the network locates a pivot inside its window with AUC 0.96. What it still cannot do cleanly is call "the leg is over" live: nine of ten such signals miss.

- **The edge of a correction looks like the edge of a leg** — Study 26 tried to estimate how much of its leg an event has travelled and could not beat the constant 0.5; a counter of events since the confirmed pivot was the bar nobody cleared. This study changes the input.
- **A canvas of the whole leg — and the last rows carry it** — Legs range from tens to thousands of events, so the prefix is laid onto a fixed canvas: the class is recomputed from second-by-second data at whatever step makes the prefix fit 500 rows × 9 blocks × 7 features — a long leg is compressed, a short one stretched.
- **No architecture beats the compressed canvas — and live calls still miss** — The next round removed the compression: the canvas fed whole, 500 rows, to attention with no stride; to LSTMs and transformers as a sequence; to an MLP on 4,000 inputs; to MLP-Mixers. Compression loses nothing.

### Where a Leg Really Ends — Four Events Earlier Than the Zigzag Says

https://marketflowresear.ch/articles/where-a-leg-really-ends-four-events-earlier-than-the-zigzag-says/ · published 30 September 2026 · Where a leg ends

**Summary.** We went looking for a unique, universal signature of a pivot in 531 legs of six coins. The book turned out to be a mirror — tops and bottoms read with one ruler once buyer and seller features swap. The zigzag's pivot turned out to sit in the wrong place: the true boundary of a leg is the extreme of the closing price within ±25 events, four events earlier, and there the signature becomes symmetric and one event sharp. What we did not find is uniqueness: a 0.5% correction carries the same signature, weaker. The signature measures the strength of a turn — not whether the leg is over.

- **A mirror, and a boundary in the wrong place** — The goal was a pivot signature that is both unique — present only at leg boundaries — and universal — the same for rises and falls. The class is 210×10-1×7d/d, whose features come in buyer/seller pairs, measured over 531 whole legs of six coins with 465 continuous joints. First, the mirror.
- **One event, and the last push is your own side** — The user's hypothesis was that at the end of a rise, sellers overpower buyers, and the leg's end should be set so that this final burst lies inside it. The data confirm it with the order reversed.
- **Not unique: a correction is a smaller boundary** — Third, uniqueness — and here the answer is no. A 0.5% correction carries the same signature, weaker: boundary 0.589, correction 0.353, leg body −0.033; boundary against correction AUC 0.664. The signature grows with the size of the turn — from 0.245 in the smallest quintile to 0.533 in the largest (Spearman +0.236).

### Can Our Networks' Outputs Become a Buy/Sell Signal? A Simple 1.2% Zigzag Says No

https://marketflowresear.ch/articles/can-the-networks-outputs-become-a-buy-sell-signal-a-simple-zigzag-says-no/ · published 30 September 2026 · Buy / sell from networks

**Summary.** Our networks each say something about a leg: it is ending, it is 80% done, this is a reversal zone. None says "buy" or "sell". So we combined their outputs into one decision and raced it against three rivals: each channel alone, a coin with the same number of signals, and a causal 1.2% zigzag on the same price. The combination found 40% of reversals within ±50 events — where chance at that frequency finds 30%. The plain zigzag found 72%. With $1,000 on BTC, the networks ended level with the coin; the zigzag ended next to buy-and-hold. What the networks do know is where a reversal was.

- **The race: combiner vs chance vs a simple rule** — The inputs are the outputs of the approved networks laid on one timeline per coin: legtooth and legline (where in the leg), hazard100 and hazard4 (a pivot soon), and the pivot and correction zones. Six coins, 412,811 events, 718 reversals of the 2.3% zigzag on EMA-10 (607 train, 111 later).
- **Why the combiner has nothing to combine** — The breakdown of the channels explains the failure. None of them carries direction (AUC 0.33–0.47 for up versus down). They duplicate each other (Spearman up to 1.00). And their level drifts between train and validation. A classifier of events has nothing to build a decision from that its input does not contain.
- **$1,000 on BTC: nothing beats holding** — Hits within ±50 events are not money. The last test asks directly: what would $1,000 have become on the unseen BTC window — 3,549 events, 13.3 days, price 63,043 → 77,404, an uptrend with six 2.3% reversals?
- **What the networks do know: where it happened** — The networks turned out useful for a different question than the one they were asked. They do not say when to reverse — but they say where the reversal was. A zigzag is late by construction: it waits for price to move away from the extreme.

### A Loss Ladder for Five Trained Networks — and the Biggest Win That Was a Dropout Bug

https://marketflowresear.ch/articles/a-loss-ladder-for-five-trained-networks-and-the-win-that-was-a-dropout-bug/ · published 30 September 2026 · Loss ladder

**Summary.** We took five approved networks from this series and tried to improve them without retraining: change the loss function between epochs, raise the penalty step by step, keep an epoch only if a fixed metric on validation improves. 124 attempts, 6.9 machine hours. One network truly improved — the one whose own training had stopped too early — and it improved from a smaller learning rate, not a new loss. The spectacular result, +0.47 on the zone model, vanished when the saved weights were measured again in a separate process: the GPU backend had left dropout switched on during evaluation.

- **The ladder and its fixed yardstick** — This series has several approved networks, each with its best version: weights, settings and the validation number it was chosen by. The question here: can an approved version be fine-tuned by changing the loss function between epochs so that its own fixed metric gets better?
- **Fine-tuning helps exactly where training stopped short** — The result sorts cleanly by one property — how each network's own training had ended. hazard100 had been stopped by patience while its metric was still rising (best epoch 8 of 11); the ladder added +0.0224 (0.7204 → 0.7428), with 14 of 29 attempts outside the noise.
- **The +0.47 that was not there** — The headline number of the run was zone: score 0.7055 → 1.1799, and the confusion between the pivot and correction channels cut from 0.417 to 0.263 — five rungs kept, the training curves spotless. Then all saved weights were measured again, event by event on validation, in a separate process.

### A Correction and a Reversal Are One Event of Different Strength — What the Book Shows Inside a Leg

https://marketflowresear.ch/articles/correction-and-reversal-are-one-event-of-different-strength/ · published 30 September 2026 · One axis: normal → correction → pivot

**Summary.** Every leg of a zigzag holds its own common level in the order book, and removing it would make the unusual events stand out — if only it could be known in time. It cannot: the previous leg predicts it so poorly that subtracting it adds noise. What the book does show is a reversal zone of about 15 events with a clear signature — and the same signature, weaker, inside every correction. Normal event, correction, pivot: one axis. Models find the pivot zone at AUC 0.72–0.75 and put 8.5× the base rate in their most confident answers, but no memory, loss or penalty separates a pivot from a correction without breaking the correction channel.

- **A common level — known only after the leg ends** — Inside one leg the seven features of 210×10-1×7d/d sit at a level different from the neighbouring leg's. That level is the same for every event of the leg, so it says nothing about any single event — it only hides the ones that stand out. The idea: subtract it and look at what is left.
- **Normal, correction, pivot — one axis** — Next, the zones themselves. A reversal zone is about 15 events (5.8 before, the pivot, 7.3 after) — some 53 minutes in which price moves 0.71%. A correction zone is shorter (11.4 events) and narrower (0.42%). The reversal has a signature in the class, and it is not an artefact of how zones are cut.
- **Models find the zone — not the difference** — A three-class model (normal / correction zone / pivot zone), trained on six coins because BTC alone has only eight pivot zones in validation, reaches AUC 0.724 for the pivot zone with a causal window — slightly better than a centred one (0.718): everything it knows about a reversal, it knows from the past.

### The Book Sees the Edge of a Leg, Not Its End — Event by Event on BTC

https://marketflowresear.ch/articles/the-book-sees-the-edge-of-a-leg-not-its-end/ · published 30 September 2026 · Edge, not end

**Summary.** Event by event on BTC, every feature of our order-book class carries its place in the leg: a U-shaped profile that holds inside individual legs, with the same sign in 80% of them. But what it reads is the distance to a pivot, not the progress through the leg. It tells a real end of a leg from a shallow 0.5% correction (AUC 0.669), and loses that ability as the correction deepens — at 1.5% there is nothing. A detector of "this is the tail of a leg" lifts precision up to ×19 over the base, yet cannot tell a tail from a head. The book knows it is at an edge; it does not know which one.

- **Place in the leg is written in every feature** — Study 29 measured the class along the leg with one summary per event. This one goes event by event on BTC (70,647 events, 70 legs of the 2.3% zigzag on EMA-10) and summarises each event six ways — mean, first and last block, peak, spread, slope — in ranks.
- **A real end vs a correction: only when the correction is small** — Knowing the pivot afterwards is easy; the causal question is harder. Inside every leg there are places where price turned against it and the leg went on. Is the true end different from such a correction — a pivot of a nested smaller zigzag, at least 40 events from the leg's ends?
- **The tail of a leg is visible — but so is its head** — Three more questions pressed on the same point. When can a new leg be called? The difference between the two sides of a pivot is present in the raw 63 numbers of an event from the very first event after it (AUC 0.26 on the strongest number, z −6.5, p = 0.000 against shuffled labels).

### The Order Book Along a Leg — Slow Legs, Thin Edges, and a Signal at the End That Beats Chance ×4.6

https://marketflowresear.ch/articles/the-order-book-along-a-leg-slow-legs-thin-edges-and-a-signal-at-the-end/ · published 30 September 2026 · The class along the leg

**Summary.** We laid the seven features of our best order-book class along 554 zigzag legs of six coins. Every feature marks the edges of a leg, with the book thinning per unit of flow right before a pivot. The same features tell a short leg from a long one early — not the exact length, but its distribution. A memory network trained on "the last 5% of the leg" puts 23% of its loudest calls there, against 5% by chance. And the honest cost is written next to each number: detecting a new leg takes 12–16 events, a price-path detector still does better, and the zigzag labels themselves arrive late.

- **Every feature marks the edges** — Study 28 measured the leg on price. Here the same pass is applied to 210×10-1×7d/d — six derivative ratios between book levels and flow, plus the price derivative — over 554 legs and 309,021 events on six coins.
- **A new leg is visible after 12–16 events** — Asked whether a 2.3% reversal happened within the last X events, gradient boosting on the class reaches AUC 0.737–0.762 for every X from 3 to 100 on the unseen weeks — longer history does not make the class see a reversal better.
- **Short or long: the leg's length as a distribution** — Stage 1 found that feature level and leg length are linked, but measured the level over the whole leg. Taken causally from the first K events after a pivot, the link holds: split legs into five equal groups by the level of support-per-flow in the first 10 events and the median lengths run 238 → 372 → 476 → 483 → 565.
- **"The last 5% of the leg": 23% hits where chance gives 5%** — The last stage labels the final 5% of each leg and trains a memory network on 131 channels from seven classes (451 legs train, 103 validation).

### A Leg Moves at Its Edges — and You See the New One Only After It Starts

https://marketflowresear.ch/articles/a-leg-moves-at-its-edges-and-you-see-the-new-one-only-after-it-starts/ · published 30 September 2026 · Edges of the leg

**Summary.** Split every 2.3% zigzag leg of six coins into tenths and measure where the price actually travels: 39% of the move happens in the first and last tenth, three times more per tenth than in the middle. The start of a leg reveals its direction and its volatility, never its length or its ending. A detector for "a new leg has begun" reaches 63% precision 21 events after the pivot and 98% after 31 — confirmation, not foresight. And twelve numbers about the price path beat every order-book class in the project at that task.

- **Where a leg makes its move** — Study 01 found two things and stopped there: the move of a 2.3% leg gathers at its edges, and the number of events in a leg varies from tens to thousands. This study measures both to the end, with a method written so the same pass can measure any class of the project.
- **What the start of a leg does and doesn't tell** — From the first events of a leg one can read its direction (0.978) and the typical size of counter-moves inside it (+0.60). One cannot read when it will end (+0.44, but a mere 4% better than the median), how it will end (−0.04 to +0.06), or its largest counter-move (−0.03).
- **Detecting a new leg: precision bought with delay** — The question "did the new leg start within the last X events?" gets AUC 0.80–0.83 for every window from 5 to 100 events on the unseen weeks. What changes is the label: at X = 5 it is true once per hundred events, at X = 100 once per five.
- **Twelve price numbers beat every class** — The last step asked the same question in the language of each event class. On the same series, labels and window, price beats the book: 0.830 against 0.759 for 210×10×7, 112 columns of levels and volumes.

### The ±0.15% Pivot Zone — Seventeen Ways to Describe an Event, One Honest Ceiling of 0.70

https://marketflowresear.ch/articles/the-pivot-zone-plus-minus-015-percent-seventeen-ways-to-describe-an-event/ · published 30 September 2026 · Pivot zone ±0.15%

**Summary.** Near every 2.3% pivot, price lingers within 0.15% of the turning level for a while — a median of 11 events, about 38 minutes, 2% of the whole series. Is that stretch recognisable from the order book alone? We asked gradient boosting on every event of six coins, with batches of 1 to 11 events and all 17 event classes this project has built. The answer is a consistent "a little": AUC 0.61–0.66 on the main class, 0.701 at best on raw book levels — and at a working threshold, 95 hits bought with 6,725 false signals.

- **A zone defined by price, measured by the book** — The zigzag is the same as in studies 21–26: 2.3% on an EMA-10 of event prices. Around each pivot, price holds the turning level for a while, and that stretch is the zone: the continuous run of events around the pivot where the EMA stays within ±0.15% of its value at the pivot.
- **Recognisable — statistically** — On 210×10-1×7d/d, the class of the last several studies, gradient boosting recognises the zone on the unseen weeks with AUC 0.61–0.66 at every batch length — stable, never falling to 0.5. One event gives 0.642; eleven events, 0.655. That is 693 inputs instead of 63 for +0.013. The working point shows what that means.
- **Seventeen classes: raw levels win, prices lose** — The zone, label, split, batch grid and model stayed fixed; only the description of the event changed — 17 classes with a dataset (three families of different density, so the base rates differ by family).

### The Filter That Would Lift Precision From 51% to 93% — and Why We Cannot Build It Yet

https://marketflowresear.ch/articles/the-filter-that-would-lift-precision-to-93-percent-and-why-we-cannot-build-it/ · published 30 September 2026 · The perfect filter we cannot build

**Summary.** Half of the reversal-confirmation signals are false, and they share one trait: they point into the middle of a leg. A filter that knows how far along its leg an event really is removes 45 of 49 false signals and not one true one — precision 51% → 92.7%. The catch is that "how far along" is only known in hindsight. Every causal estimator we built — trees, a memory network, ten loss functions, a 24-point grid — reaches an AUC of about 0.69 for "this is the last tenth of the leg", where the filter needs something near 0.95. With that estimate, the filter cuts true signals as fast as false ones.

- **The idea works — with the true answer** — The confirmation ladder of the last two articles answers "was there a pivot N events ago?" at about 50% precision on unseen weeks — half the signals are wrong. This study asks whether a second signal can cancel the wrong half.
- **The causal estimate cannot see the end of the leg** — The estimator gets only causal inputs: a counter of events since the last confirmed pivot, the five context numbers on four windows, and optionally the 62 class numbers.
- **A weak estimate cuts the wrong signals** — Used as a filter, the estimated share does the opposite of what was hoped: it cuts true signals faster than false ones. At a threshold of 0.5 the trees' estimate removes 8 false and 5 true signals (51.0% → 52.9%); at 0.9 it keeps 3 of 100 signals and precision falls to 37.5%.

### Can a Memory Network Rebuild Context From the Raw Sequence? Not From the Order Book

https://marketflowresear.ch/articles/can-a-memory-network-rebuild-context-from-the-raw-sequence/ · published 30 September 2026 · Memory vs aggregates

**Summary.** Trees won the reversal ladder with hand-made aggregates: five numbers about recent extremes over four look-back windows. A memory network should not need them — the price change of every event is already inside the class, and integrating it would rebuild the whole context. We gave an LSTM 60 events of raw class, of price context, and of both. The class alone reached an honest AUC of 0.77; price context alone, 0.965. Adding the class to price made it worse. And trees on five aggregates still found reversals two and a half times more precisely than the network.

- **The question: is the class enough?** — Studies 23 and 24 worked with aggregates: before any model saw the data, ten events were folded into medians, slopes and peak positions, and the context windows (200 to 1,600 events) were chosen by hand.
- **The class alone falls far behind price** — The gap is sharp. The class gives an honest AUC of 0.772 (0.735 on unseen weeks); price context, 0.965 (0.950). The network does not rebuild from d_price_avg what the ready-made position numbers provide.
- **Trees on five numbers still win** — Compared on the same rung and the same unseen weeks, gradient boosting from study 24 beats every network: 21.5% precision on the five context numbers, 21.4% on four windows, AUC up to 0.987 — against 8.5% and 0.950 for the best LSTM, even though that LSTM sees the same twenty numbers laid out over time.

### Pivot Confirmation on Three Unseen Weeks — Precision Holds, the Threshold Does Not

https://marketflowresear.ch/articles/pivot-confirmation-on-three-unseen-weeks/ · published 30 September 2026 · Held-out time

**Summary.** The reversal ladder of the previous article was scored on a held-out coin — but in the same weeks it trained on. Here the models trained on everything up to 14 August 2026 and were scored on the three weeks after, six coins, with the threshold fixed in advance from honest training scores. Precision did not fall: 51% two and a half hours after a pivot, 73% after six hours, on 88–93 reversals. What broke instead was the threshold itself — at two rungs of the ladder the models went completely silent while their AUC stayed above 0.9.

- **A stricter test: held-out time, fixed threshold** — Study 23 measured everything on train with a held-out coin: the model had not seen that coin, but it had seen those very weeks. Here the test is held-out time: train on all six coins up to 14 August 2026, then score 14 August – 4 September.
- **Precision survives the move to new weeks** — At rung 43 (150 minutes after the pivot) four windows plus class give 55.0% on train and 51.0% on the unseen weeks; four windows alone, 49.9% and 49.1%. At rung 103 (six hours) validation is even higher than train: 73.3% against 67.9%. Rung 43 is the working point: the highest recall (55–60%) at about 50% precision.
- **The model separates — the threshold does not travel** — The failure that did appear is of a different kind. At rung 23 three of the four input sets gave no signal at all on validation; at rung 3 the four-windows-plus-class set was silent.

### A Reversal Is a Place in the Move, Not a Shape in the Book — Five Causal Numbers Beat 62

https://marketflowresear.ch/articles/a-reversal-is-a-place-in-the-move-not-a-shape-in-the-book/ · published 30 September 2026 · Where, not what

**Summary.** The edge of a leg really is different from its middle — in 85% of 600 legs, by a factor of about 3.8, with a coin-flip control at 0.51. Yet no single 10-event batch can be picked out of the whole dataset by its book numbers. Five tests later the reason is clear: what identifies a reversal is where you are in the move, not what the book looks like. Five causal numbers about recent extremes lift the hit rate 121-fold over the base; the 62 numbers of the class manage 3.8-fold and add nothing on top. Asked again every ten events with four look-back windows, "was that a pivot?" climbs from 26% after ten minutes to 71% after six hours.

- **The difference is real — inside each leg** — Study 21 showed that on average over hundreds of legs the edge of a leg looks different from its middle. Study 22 showed that a single 10-event batch has no signature of its own. Both are true at once, and this study tries five ways to get the average difference out of the noise.
- **Longer batches help — four times slower than they should** — Averaging more events should shrink noise. It does, a little: the best AUC rises from 0.744 at 10 events to 0.767 at 60 (2.3–3.5 hours), and precision at 10% recall from ×4.3 to ×10.3 the base rate.
- **Where you are beats what you see: ×121 vs ×3.8** — The fourth and fifth tests stop describing the batch and describe the move instead, causally — only from what is known at that moment.
- **Ask again: 26% at ten minutes, 71% at six hours** — The last test turns the detector into a procedure. Instead of asking once, ask at every rung of a ladder: was there a pivot exactly N events ago?, for N from 3 to 203, with a held-out coin for scoring. With a single 400-event window the answer never improves: 22–35% at any delay.

### A Reversal Has No Signature of Its Own — One in Four Ordinary Stretches Looks Stronger

https://marketflowresear.ch/articles/a-reversal-has-no-signature-of-its-own-in-the-order-book/ · published 30 September 2026 · No reversal signature

**Summary.** Around a 2.3% pivot the order book changes in multiples: flow ratios two to five times bigger, book-per-flow ratios three to seventeen times smaller. That sounds like a fingerprint. It is not. One in four continuous 10-event stretches far from any pivot is stronger still, the strongest is 47 times stronger than the strongest reversal, and a neural network on all 630 numbers of a reversal batch reaches the same AUC as a straight line: about 0.70. The reversal is a shift of the same distribution, not a different kind of event.

- **The edge of a leg is five events** — This study works with one class, 210×10-1×7d/d from the previous article: seven derivative ratios over nine 21-second blocks. Along a leg the seven features spread widely near both pivots and bunch together in the middle. The first question was practical: how many events from each end of a leg count as its "edge"?
- **Strong is not special** — If size were the signature, the strongest stretches of the series would be reversals. So the series was scanned with a 10-event window, one step at a time, keeping only windows that never touch a reversal zone — 230,262 of them.
- **Nothing unique: every test lands at 0.70** — Four ways to find a signature, each one more demanding. Models on the raw batch. All 630 numbers of a reversal batch into a logistic regression and a small network. Validation AUC 0.695 and 0.687; the network scores 1.000 on train — it memorised 437 reversals with more parameters than examples.

### Derivatives of the Order Book — Reversals Leave a Trace, Not a Warning

https://marketflowresear.ch/articles/derivatives-of-the-order-book-reversals-leave-a-trace-not-a-warning/ · published 30 September 2026 · Derivatives of the book

**Summary.** We rebuilt the basic event of this series from scratch: 210 seconds cut into ten 21-second blocks, book levels divided by the volume of their own block, then first derivatives from block to block — on six coins, 402,731 events. The new description sees reversals clearly: around a pivot there is more flow and a thinner book per unit of that flow. But almost all of it sits in the first tenth of the next leg. The book reacts to a reversal far more than it foretells one. The last class of the family, derivative over derivative, lifts a detector of the reversal zone to AUC 0.700 on fresh legs — and becomes the workhorse of every study that follows.

- **A new family of events** — Every earlier study described an event through normalised differences between the two sides of the book. This one builds a new family of classes to look at the book in its own units: - 210×10×9: the same 210 seconds cut into 10 blocks of 21 seconds, the eight book features plus price.
- **Size, not sign — and it is after the pivot** — The first comparison was events within ±10 of a 2.3% zigzag pivot (EMA-10) against the rest of their legs. Signed derivatives do not separate them at all (AUC 0.475–0.514). Magnitudes do: near a pivot there is more flow (|d vol buy| 0.619) and a thinner book per unit of that flow (|d resistance| 0.344, i.e.
- **The broken side of the book** — Split by direction, the book is mirror-symmetric. After a bottom (an up leg starts), it is the resistance side that collapses in the first tenth: |d resistance| rank 3.49, its balance 3.63, while support is barely touched (4.75, 5.46). After a top, the opposite: support 3.51 and 3.70, resistance 4.14 and 4.91.
- **Derivative over derivative: 0.638 → 0.700** — The family then got two compact classes. 210×10-1×2d folds each side into one ratio: d_up = d vol buy / d resistance and d_down = d vol sell / d support.

### Change the Question, Not the Network — "Will the Leg End Soon?" Scores 0.73 Where "Which Tenth?" Scored Nothing

https://marketflowresear.ch/articles/change-the-question-not-the-network-leg-end-hazard/ · published 30 September 2026 · Ask what the data answers

**Summary.** The memory network of the previous study learned to count instead of reading the market. We opened its loss function and found why: counting was the exact optimum. Then a plain perceptron settled it — the 60 order-book numbers of an event carry zero information about which tenth of the leg it sits in. They do carry information about something else: whether a pivot is close. Rebuilt around that question — four horizons, one monotone curve — the same data scores an average AUC of 0.73 across six coins. The order book shows the exhaustion of a move. It still does not show its direction.

- **The loss never asked for class 10** — Study 19 ended with a network that never once called the end of a leg on 42,192 events. Rather than tune it, this study looked at what it was minimising.
- **Ask "is a pivot close?" — and the features wake up** — The tenth of a leg is defined by its full length. "A pivot is close" is a local property of price, visible from the event itself. Same perceptron, same features, same legs: the question "does the leg end within the next N events" gives AUC 0.644–0.658 for N = 5–20 on the features alone, while the counter drops to 0.54.
- **One small validation set almost hid the result** — For several sections the reference number was an AUC measured on BTC after the split date: 3,800 events, six pivots. The one-horizon model scored 0.604 there. Run on all six coins, the same model averaged 0.710 — BTC was the worst of the six, not the typical one.
- **What the event shows before a pivot: uniformity** — Which part of the event carries the signal? Averages carry nothing: 0.49–0.52 for every feature. The range across the 15 blocks of the event does — volume balance alone scores 0.649 on train (0.702 on validation), against 0.733 for the whole memory network.

### A Memory Network Asked "Is the Leg Over?" Learned to Count, Not to Read

https://marketflowresear.ch/articles/a-memory-network-that-learned-to-count-not-to-read/ · published 30 September 2026 · End of the leg

**Summary.** An LSTM with attention walked each 2.3% zigzag leg event by event and, at every step, said how far along the leg was. Across five label designs, one coin and then six, it never once placed its end-of-leg signal inside the last tenth of a fresh leg. What it learned was a clock: a steady count from the start of the leg. When the zigzag stopped handing it that start — memory reset only when the network itself called a pivot — it did not call a single one in 42,192 events.

- **A new question: the whole leg, event by event** — Studies 16–18 judged each event through a window of fixed length and asked "where in the leg are we". This one changes the unit. The network gets a whole leg, from the event after one pivot to the next pivot, event by event, remembers everything it has seen, and at each step outputs one number: is the leg ending here?
- **It shouts "end" all the time** — Under a positive-class weight of 7.6, the cheapest answer is to keep the end-of-leg probability above threshold almost everywhere. On one 877-event leg it crossed the threshold 550 times.
- **Six coins, 602 legs — the step became a ruler** — To get statistical power the training set grew to all six coins: 602 legs, 332,891 events. Two penalties shaped the output into a staircase — no jumps over a class, no steps backwards. They worked perfectly: zero jumps, 99.0% clean +1 steps on train, and the same on fresh data. And that is exactly the problem.
- **Take away the zigzag's gift** — One advantage had been given for free all along. A 2.3% pivot is confirmed only after price moves 2.3% away from the extreme, yet every leg entered the network with a clean memory exactly at its first event. Live, nobody knows where that first event is. So the last test removed it.

### Four Numbers per Event and a 28-Cell Answer — Where the Signal Really Came From

https://marketflowresear.ch/articles/four-numbers-per-event-and-a-28-cell-answer/ · published 30 September 2026 · Four numbers, 28 cells

**Summary.** The previous networks read each book feature through 13 exponential averages and topped out at a correlation of 0.29 with the leg target. Strip the averages away and the same features, raw per event, correlate at 0.09. So the link was made by the smoothing, not by the snapshot. We then gave a transformer six coins, 345,000 training windows and a new kind of output — 28 cells instead of one number — to stop it hiding in the middle. It stopped hiding. It did not start hitting: 5.4% of real pivot-zone signals, with false calls a day away.

- **The snapshot barely knows the leg** — Studies 16 and 17 fed their networks a 13 × 4 matrix: four book features, each smoothed by 13 EMAs up to 130 events. The strongest single link with the target there was |r| = 0.294 — volume balance under a 130-event EMA. This study asks how much of that is in the features themselves.
- **Smooth each leg by its own length** — The target twitches between pivots, and earlier studies calmed it with one fixed window for all legs. But 2.3% legs vary enormously: median 802 events, shortest 227, longest 2,431. One window undersmooths long legs and flattens short ones.
- **28 cells instead of one number** — The minimum of MSE on a noisy target is the conditional mean, which is why every model so far clustered near zero: saying −0.9 and missing costs more than saying nothing. So the output changed.

### Same Data, Different Readout — How a Transformer Fails Differently From an Attention Query

https://marketflowresear.ch/articles/same-data-different-readout-transformer-vs-attention-query/ · published 30 September 2026 · Readout

**Summary.** We repeated the previous study exactly — same 2.3% zigzag target, same pivot zone, same loss, split and seed — and changed one thing: how the network reads its own sequence. Averaging the 13 steps instead of letting a learned query pick them cut the error from 0.554 to 0.476 and made "vertex" calls 44% precise instead of 3%. Tuning then turned the same model into a noisy caller again. And past the end of the dataset, both readouts lose the direction. The readout decides how a network is wrong, not whether it is right.

- **Averaging vs a learned query** — The setup is identical to the previous article: 13 EMA periods × 4 book features per event, a 2.3% zigzag on a 90-event EMA of price as the target, the pivot zone |y| ≥ 0.90 within ±40 events, a spread-penalised weighted MSE, a time split and seed 0. Every number here compares directly.
- **A grid that found one lucky run** — Stopping at epoch 3 was a hint that the settings deserved a search, so the transformer — unlike wattn — got the full procedure: a coarse grid, a fine grid around the best, then training length. The coarse grid had 24 configurations; 21 hit the zone zero times on validation, 14 of them stopping after 2–8 epochs.
- **Longer windows and ten unseen days** — With windows of K = 5 … 50 events, context again adds neither accuracy nor direction: MAE stays at 0.51–0.58, sign at 52–57%, with no trend in K. Hits jump — 9.1% at K = 5, 19.9% at 10, zero at 15–25, 25.7% at 45, zero at 50.

### A Network That Found the Pivot Zone — Only Where It Was Trained

https://marketflowresear.ch/articles/a-network-that-found-the-pivot-zone-only-where-it-was-trained/ · published 30 September 2026 · Pivot zone net

**Summary.** We asked an attention network to do one specific thing: tell when price is near the end of a 2.3% zigzag leg — within 10% of the pivot and no more than 40 events from it. On the weeks it trained on it caught 65% of those zones. On validation, 4%. On test, 3.8%. On ten days after the dataset ended, its correlation with the target turned negative. This is a measured negative result, and the measuring tool it produced — the pivot zone — is what we carry forward.

- **Measure the distance to the pivot, not the average error** — The target is the zigzag of this series at its larger scale: pivots of a 2.3% zigzag drawn on a 90-event EMA of price, and each event's position in its leg rescaled to −1 at the low pivot and +1 at the high one (min-max, so the distance to a vertex means a distance in price).
- **One event in, 26,049 parameters — memorisation** — The network is wattn — attention over the 13 EMA periods with a learned query for the readout, 26,049 parameters, trained with a spread penalty so its output is not squashed to the middle (output spread 0.969 of the target's). The spread came back; the vertices did not.
- **Smoother target, more context — no curve** — Two fixes were tried on the target and the input. Smoothing between zones. The leg target twitches between pivots. Of eight ways to calm it, a moving median removed reversals (×563) but not roughness (×1.15), and a straight line through the leg killed the link to the features (max |r| from 0.294 to 0.241).
- **Ten unseen days: the sign flips** — The two finalists after a grid search: K = 10 (width 64, one layer) catches 12.3% of the zone on validation with sign 61.8% and Pearson +0.226; K = 45 (two layers) catches 18.8% with Pearson +0.291. Dropping the one or two weakest features even helps a little (+2.6 and +0.9 points of hits).

### Six Networks Tried, the Winner Has 53 Parameters — Teaching a Model Where Price Sits in Its Leg

https://marketflowresear.ch/articles/six-networks-one-winner-with-53-parameters/ · published 30 September 2026 · Zigzag net

**Summary.** Correlations said the slow order book knows roughly where price sits inside its zigzag leg. So we trained networks to say it out loud: a number from −1 (at the low pivot) to +1 (at the high pivot), from four book features under thirteen averages. Six families on the same data, a time split with five walk-forward windows — and the linear model with 53 parameters won. On the final test it beats the constant for the first time (MAE 0.438 vs 0.446, sign 55.1%); on eleven fresh days outside the dataset, sign 60.7% on 44 legs. Small, consistent — and the reason is a loss that pays attention to the edges of the leg.

- **A target that moves slowly, and a split that respects it** — The target is the position of each event inside its zigzag leg: an independent 0.8% zigzag on event prices, the leg rescaled by rank to [−1, +1] and smoothed with a 10-event EMA on the leg (so the pivots stay put).
- **Six families, same budget — the simplest wins** — Six model families were compared on identical data and budget, each with default settings — families, not tunings. The metric is the mean validation MAE over the five windows.
- **Weight the edges of the leg — and the model generalises** — The weighted loss is w = 1 + α·|y|^β: an error at a pivot costs more than an error mid-leg. Under plain MSE almost all of the gradient came from the middle of the leg, where there is least signal; the weight moves attention to the edges, where there is some.
- **Eleven fresh days, and how much to believe it** — The saved model and scaler were then run on raw data outside the dataset — 14 August 14:00 to 25 August 2026, rebuilt through the same chain of classes. On 2,512 events and 44 legs: MAE 0.387, R² 0.100, Pearson 0.344, sign 60.71%.

### Twelve Hours of Order-Book State and the Direction of a Move — From 0.16 to 0.42

https://marketflowresear.ch/articles/twelve-hours-of-order-book-state-and-the-direction-of-a-move/ · published 30 September 2026 · Slow book state

**Summary.** Does the order book, averaged over about half a day, know which way the current move is going? We smoothed four book features with exponential averages on 52 horizons from 35 minutes to 58 hours and correlated them with the direction of 2.3% and 0.8% zigzag moves on 40,242 BTC events. The raw answer is weak (−0.16 at best). Three changes of question — a price feature instead of price, the position inside the leg instead of the direction, and a zigzag drawn on the same smoothing — lift the ceiling to 0.42. That is 17% of the variance, from a description that ignores the last hour entirely.

- **The raw answer: weak, and pointing backwards** — The events are the same 210-second windows as everywhere in this series — 40,242 BTC events from 14 February to 16 August 2026, laid out as one continuous timeline.
- **A price feature beats the price** — The first change: instead of averaging the price, turn it into a feature built by the same rule as the others — the normalised difference between upward and downward movement inside each 14-second block, i.e. how straight the price travelled. All four series then live on the same [−1, 1] scale.
- **Ask where, not which way — on a smoothed zigzag** — Two more changes of question moved the ceiling far more than any feature did. Position instead of direction. Every event gets its place inside its zigzag leg: −1 at the low pivot, +1 at the high pivot, proportional in between — and the feature is rescaled the same way over the same leg, so like is compared with like.
- **What 0.42 means — and what it does not** — The picture is consistent and it is useful: slow book state carries real information about the phase of a move, and it carries it best when the question is posed on the same time scale as the features.

### Predicting the Next Phase of a Move From the Order Book — 832 Window Sizes, One Honest Ceiling

https://marketflowresear.ch/articles/predicting-the-next-phase-of-a-move-from-the-order-book/ · published 30 September 2026 · Next cohort

**Summary.** Every event of BTC's order book sits in one of four phases of a zigzag: rising, a pullback inside a rise, falling, or a pullback inside a fall. We asked a closed-form linear model to name the phase of the next event from the N events before it — and tried every N from 2 to 833. The signal is real from N = 2, peaks at 0.418 accuracy on about one zigzag leg of history against a 0.377 baseline, and is essentially binary: up or down, the pullbacks go unseen. The labels of the past alone would score 0.976.

- **Four phases of a zigzag** — The labels come from two zigzags laid on the same price series: a big one with a 2.3% threshold and a nested one at 0.8%. Their pivots merge into a single timeline, and the stretch between two neighbouring pivots is a segment.
- **A closed-form model, 832 window sizes** — The forecaster is deliberately the simplest honest thing: each cohort is a Gaussian in the space of the window, with a shared covariance, and the decision is Bayes' rule — a linear discriminant computed from sums in closed form. No iterations, no seed, no tuning.
- **Binary in practice — and the ceiling that labels would give** — Look at what the best model actually says. At N = 203 it names 55.2% of rising events and 55.1% of falling ones — and 0.1% and 2.2% of the two pullback cohorts. Pullbacks are a third the size of their trend cohorts, and the order-book window does not separate them, so the model simply stops naming them.

### A Leg Is the Sum of Its Events — and Size Beats Count

https://marketflowresear.ch/articles/a-leg-is-the-sum-of-its-events-size-beats-count/ · published 30 September 2026 · Legs and events

**Summary.** We cut the price of six coins into 210-second events and into 2.3% zigzag legs, then asked the simplest question a direction model has to answer: how does a leg actually make its move? Over 712 legs and 409,998 events the answer is exact — the leg is the sum of its events (0.999 on clean legs) — and surprising: up legs are only 51.9% up-events. The move is carried by the size of events, not by how many point the right way.

- **The map before the labels** — Every study in this series works on the same raw material: the Binance order book recorded every second — eight book features plus price — for six coins since February 2026.
- **A leg is exactly the sum of its events** — The precise question was about legs: moves of at least 2.3% on a smoothed price (EMA-10 of event prices), between zigzag pivots. We used the full continuous series of all six coins — 712 legs (355 up, 357 down), 409,998 events, median 416 events per leg, median move 4.41%.
- **Why you cannot read direction from one event** — Here is the number that explains why direction is hard. Inside an up leg, 51.9% of events are rising; inside a down leg, 46.7%. That is the entire count advantage — about five percentage points, accumulated over hundreds of events. It is the same on every coin: 0.514–0.525 in up legs, 0.458–0.479 in down legs.
- **Legs that move against their own majority** — If the edge is that thin, some legs must lack it. We took them apart. 87 legs (12.2%) made their move while having fewer events pointing their way. They are not glitches: they win on size, and they win in jumps.

## Series: Live bots — every result (7)

Bots built from this research and run live: winners and losers, every trade.

### Line 7: The Best of All Our Bots — Live Trading Report, Every Trade

https://marketflowresear.ch/articles/line-7-live-trading-report/ · published 28 October 2026 · Live report · Line 7

**Summary.** We have built and tested more than a dozen trading bots on the research published here. Line 7 is the only one ahead of buy & hold in live trading: from 5 October to 28 October 2026 its paper account made +5.44% after costs while BTC fell 5.29% — 6 trades, 5 take-profits. Real positions on the exchange: +1.39 USDT. Six trades are not a track record; this report lists every one of them and is updated as trades close. Research, not financial advice.

- **All our bots, one scoreboard** — Every bot in this table was built from the research on this site. Of those that traded live, every one except Line 7 ended behind simply buying and holding over the same days — including the two that made money. Line 7 is the only bot ahead of buy & hold in live trading: +5.44% against −5.29%.
- **How Line 7 decides** — Every 10 minutes Line 7 looks back over the last 160 hours of second-by-second Binance order book for BTC — resting liquidity on both sides, liquidity added and pulled, buy and sell flow — folds it into a canvas of 1,100 records of 100 seconds with 8 features each, and hands the canvas to two small transformers.
- **Every trade** — Costs are inside every number: 0.10% exchange fee and 0.02–0.03% slippage per side. Trades are booked at the close of the 10-minute decision step, the way the bot itself books them. No position is open at the time of this update.
- **The paper account against buy & hold** — The paper account of the current version started with $10,000 on 5 Oct 22:00 UTC. On 28 October 2026, 11:50 UTC it holds $10,543.75 (+5.44%). Buying BTC at the first step and holding it to the last would have given −5.29%.
- **Real money on the exchange** — Every signal is also sent to a real CEX.IO margin account: a $100 position at ×4 leverage. Closed so far: Net on the exchange: +1.39 USDT. The exchange side differs from the paper account for reasons we publish as they happened.
- **What six trades can and cannot tell** — A winning trade has paid about +1.63% after costs and the stop −2.76%, so the bot needs to win about 63% of its trades just to stand still. A bot with exactly that break-even win rate and no edge at all would still win 5 or more of 6 trades in about 28% of cases — roughly one time in 4. Six trades cannot rule luck out.

### Bot 7 — From an Order-Book Canvas to Live Trading: Every Stretch, Every Result, Every Real Trade

https://marketflowresear.ch/articles/bot-7-from-an-order-book-canvas-to-live-trading-every-result-every-trade/ · published 9 October 2026 · Bot 7 overview

**Summary.** Bot 7 is the end of a long research line: a transformer reads 30 hours of BTC order book as one canvas and says how likely it is that this moment sits in a 2.3% bottom — or, in its second version, a 2.3% top. This article puts the whole path in one place: how the networks were trained, what each version made on training-period weeks it was not fitted on, on validation, on days nobody had looked at, and since it went live. All live trades so far are listed at the end, model and exchange side by side. As of 9 October 2026 08:20 UTC the record is three closed trades: two take-profits and one stop.

- **How bot 7 was trained** — Input. Every 10 minutes the bot takes the last 160 hours of second-by-second Binance order-book data (resting liquidity on both sides, liquidity added and pulled, buy and sell flow), folds it into 1,100 records of 100 seconds and 8 features each, and gives the canvas to a small transformer.
- **Results on every stretch — against buy & hold** — All historical runs start from $1000, put the whole account into each trade, use no leverage and pay 0.05% fee plus 0.02% slippage per side. Held-out training weeks, 8 Jul – 12 Aug (bot 7): $1086.70 against $1022.67 for buying and holding — the only stretch where the bot clearly wins, in a month BTC rose 2.4%.
- **Live since 30 September — every trade so far** — The bot runs on the data collector's live feed, a decision every 10 minutes. The model's trades are booked on paper from $10,000 at the close of the decision step, with exchange-level costs (0.10% fee + 0.02–0.03% slippage per side).

### Bot 6, Line 61: Nine Trades, Eight Winners, and Why We Are Not Celebrating

https://marketflowresear.ch/articles/bot-6-line-61-nine-trades-eight-winners/ · published 17 August 2026 · Line 61 live

**Summary.** Line 61 of Bot 6 holds one position at a time and has been live since 17 July 2026: 31 days, 9 closed trades, +11.81 USD on a 100 USD slot, win rate 89%. Its entry gate is agreement between two independent models plus a context check; its exits are a 2% target, a 4% stop and a hard 72-hour clock. This is the complete record, including the comparison with its twin line running the identical recipe — which is the part that keeps us honest.

- **One trade at a time: how line 61 works** — Line 61 is one instance of Bot 6 — a bot that holds one position at a time, across all six coins, with a single slot of 100 USD. It has been running on live data since 17 July 2026, on paper, with a fee of 0.15% per action. Its entry rule is a gate made of agreement rather than of thresholds.
- **31 days live: the numbers** — As of 2026-08-17, after 31 days live: Open at the time of writing: BTC long +1.98% after 31 h — +1.83 USD unrealised in total. One trade at a time over 31 days means roughly one closed position every 3.5 days — and that number is the most informative one in the table.
- **Nine trades, eight winners, and why that is not proof** — 7 of line 61's 9 closed trades hit their take profit, the win rate is 89%, and the line is +11.81 USD ahead on a 100 USD slot. This is the best live result in the project so far, and this section exists to argue against reading too much into it. Start with the sample. 9 trades.

### Bot 6, Line 60: A Bot That Mostly Waits — and What the Clock Costs It

https://marketflowresear.ch/articles/bot-6-line-60-a-bot-that-mostly-waits/ · published 17 August 2026 · Line 60 live

**Summary.** Line 60 of Bot 6 holds one position at a time and has been live since 15 July 2026: 33 days, 9 closed trades, +0.68 USD on a 100 USD slot, win rate 44%. Its entry gate is agreement between two independent models plus a context check; its exits are a 2% target, a 4% stop and a hard 72-hour clock. This is the complete record, including the comparison with its twin line running the identical recipe — which is the part that keeps us honest.

- **One trade at a time: how line 60 works** — Line 60 is one instance of Bot 6 — a bot that holds one position at a time, across all six coins, with a single slot of 100 USD. It has been running on live data since 15 July 2026, on paper, with a fee of 0.15% per action. Its entry rule is a gate made of agreement rather than of thresholds.
- **33 days live: the numbers** — As of 2026-08-17, after 33 days live: Open at the time of writing: ETH long +1.50% after 40 h — +1.35 USD unrealised in total. One trade at a time over 33 days means roughly one closed position every 3.7 days — and that number is the most informative one in the table.
- **What the clock costs it** — Line 60's defining number is not its profit — it is that 7 of its 9 closed trades ended on the 72-hour clock, not at a target or a stop. The bot was right that something was happening; it was wrong, or too early, about when.

### Bot 4: What Forty Days of Live Paper Trading Actually Look Like

https://marketflowresear.ch/articles/bot-4-forty-days-of-live-paper-trading/ · published 17 August 2026 · Bot 4 live

**Summary.** Bot 4 has been trading live on paper since 09 July 2026 — 40 days, 49 closed trades, +29.53 USD on 600 USD of capital, a 80% win rate, against +1.49% on average for buying and holding the same coins. This is the full record: the rules it never deviates from, every closed trade, the open positions right now, and the hourly referee that can hand the account to a freshly trained challenger — which has happened 4 times so far.

- **The rules it has been playing by, unchanged** — Bot 4 has been running on live market data since 09 July 2026 — no breaks, no manual entries, no discretion. It trades on paper: every fill is simulated at the price its own data feed sees, with a fee of 0.15% charged on both the entry and the exit.
- **40 days live: the numbers** — As of 2026-08-17, after 40 days live: Open at the time of writing: DOGE long +1.38% after 20 h; BNB long -0.22% after 98 h; SOL long +0.41% after 113 h; XRP long -1.38% after 144 h; ETH long -0.20% after 247 h; BTC long -0.16% after 311 h — -1.07 USD unrealised in total.
- **Champion versus challenger, refereed hourly** — The interesting part of Bot 4 is not the model — it is that the model is replaceable while running. Two copies trade side by side on the same live data: the champion, whose numbers are above, and a challenger trained on fresher data with a different take-profit/stop-loss geometry.

### Gates: The Part of a Trading Bot That Decides Not to Trade

https://marketflowresear.ch/articles/gates-the-part-of-a-bot-that-decides-not-to-trade/ · published 17 August 2026 · Gates

**Summary.** A trading bot's model produces a number; its gate decides whether that number is worth a fee. This article measures what gates actually cost. Demanding a signal persist two bars instead of one raises precision from 0.209 to 0.238 and throws away 25% of the opportunities. Bot 4 stacks four gates and still lands inside the noise across random seeds. Bot 6 gates on agreement between two models — and two live lines running that identical gate for a month finished +11.81 and +0.68 USD apart. Gates buy precision with coverage; nothing here is free.

- **A gate buys precision with coverage** — Every trading bot has two halves, and only one of them gets written about. The first half is the model: it looks at the market and produces a number. The second half is the gate — the rule that turns that number into "act" or, far more often, "do nothing".
- **Bot 4: four gates in a stack** — Bot 4 is where this gets concrete. Its entry rule is a stack of four independent gates, and a trade happens only if all of them open: 1. probability — the classifier's chance of the move must clear π = 0.55; 2.
- **Bot 6: the gate is agreement** — Bot 6 uses a different kind of gate, and it is our favourite one because it cannot be tuned into a corner: agreement. Two independent models look at the same market. The trade happens only when both say the same thing, the ensemble's confidence clears τA = 0.618, and a context check confirms the wider state.

### Typhoon+: A Signal That Finds the Perfect Trader's Zones — and Still Loses Money

https://marketflowresear.ch/articles/typhoon-plus-a-signal-that-finds-the-zones-and-still-loses-money/ · published 17 August 2026 · Typhoon+

**Summary.** We drew the zones a perfect trader would use — 0.9% zigzag turns, a 0.15% band around each one — and found that the price sits inside them barely 3% of the time. A single gradient-boosting model, pooled over six coins, then finds 74% of those zones on days it never saw. It also fires false 86% of the time, leans mostly on price geometry rather than the order book, and loses 122% when traded plainly against 93% for buy & hold. This is the article about a signal that finds the right places and still cannot pay the fee.

- **The zones a perfect trader would use** — Start from the answer instead of the model. If a trader knew the future, where exactly would they act? Not everywhere — a perfect trader touches the market only at turns, and spends the rest of the time doing nothing. So we drew those places first and only then asked whether the order book can see them.
- **One model, six coins, four wrong calls in five** — One model for all six coins, not six models. The training pool mixes every coin's bars, the split is chronological — the first 70% of days train, the last 30% validate and are never seen during training — and the engine is gradient boosting.
- **Trading it: −122% against −93% for buy & hold** — A zone finder is not a trading system, and this is where the line stops being flattering.

## Series: Order-book research (45)

What second-by-second order-book data can and cannot tell you: walls, drains, volume, market states, reversals.

### A Detector for Corrections Inside a Move — and Why It Took a New Way of Measuring

https://marketflowresear.ch/articles/a-detector-for-corrections-inside-a-move/ · published 31 August 2026 · Corrections, not exits

**Summary.** We stopped measuring how loud the order book is and started measuring how much wall stands per unit of aggression — eight features collapsed into four ratios. On that new scale a class appears that we could not see before: corrections inside a move, 5,396 of them, events that already look like a reversal without being one. Three detectors find them at 0.94 precision on a held-out half, and an external 0.8% zigzag confirms the class survives after loudness is matched away (×1.69). For a bot, that is the difference between holding a trend and being shaken out of it.

- **Eight features become four ratios** — Every measurement system we built before this one asked the same question — how loud is the market right now — and every one of them ran into the same ceiling. A single number, the average level of six flow features, matched a convolutional network reading all 1,680 raw numbers of a window.
- **The correction: a reversal that isn't one** — A leg is a move of at least 2.3% in one direction. Inside it the price does not travel in a straight line — it stalls, pulls back, and continues. Those pauses are where a trading bot loses money: a stop gets hit, a position is closed, and the move resumes without you.
- **Matched for loudness, checked from outside** — Every claim in this project has to survive the same two questions: is it just loudness, and does anything outside the model agree. Corrections are louder than reversals — median volume 0.514 against 0.395 — so the first question is not optional. Loudness matched.
- **Three detectors, and what they are for** — Three detectors: one for up legs, one for down legs, one shared. The split is 50/50 by time — 20 contiguous blocks, even blocks train (15,606 windows), odd blocks validate (15,606). Contiguous on purpose: a random split lets the neighbouring window leak the answer. Two grids.

### The Order Book Tells You How Far, Never Which Way

https://marketflowresear.ch/articles/the-order-book-tells-you-how-far-never-which-way/ · published 17 August 2026 · How far, not which way

**Summary.** We asked eight order-book features the same two questions at 16 horizons from one second to a day, on six coins: how far will the price move, and which way. Size answers — ρ 0.30 at a second, still 0.11 at an hour. Direction never does: -0.002 at a minute, -0.003 at a day. Then we bet real rules on both answers over 30 out-of-sample days: the volatility bet makes +4.97 bps a trade, the direction bet loses 5.95. This is the measurement that decides what our bots are allowed to bet on.

- **Two questions, sixteen horizons** — Two questions look alike and are not. How far will the price move in the next H seconds asks for a size. Which way asks for a sign.
- **A spike is activity, not a side** — Why should a burst of order-book activity say nothing about direction? Look at what an event actually is. Take every moment a feature spikes above its own normal level and measure the next five minutes: Every flow feature multiplies the next five minutes' volatility by 1.46–1.66.
- **Betting on size, and the graveyard of direction bets** — If size is predictable and direction is not, the honest test is a bet on size.

### We Taught a Network to See a Move. It Only Heard How Loud It Was.

https://marketflowresear.ch/articles/we-taught-a-network-to-see-a-move-and-it-only-heard-how-loud-it-was/ · published 15 August 2026 · Moves, not noise

**Summary.** We trained three networks on the same 56,263 order-book windows and learned two things we did not want to learn. A single number — how loud the window is — matches a convolutional network on all 1,680 of them (0.7575 against 0.7571). And once you match windows by that number, the inflow features that carried the whole model separate nothing at all. What does survive is small, directional, and points at a different kind of bot than the one we set out to build.

- **Three targets and an honest split** — We had a cleaned dataset and no idea whether the cleaning meant anything. The chain that produced it — density clustering, then k-means, then k-means again — threw away 25,051 of 56,263 windows and kept 823 of 863 reversal windows. So the discarded part earned a name: «no reversal possible».
- **The ceiling: one number, and what is under it** — The two mirror targets are learned almost perfectly. The reversal is not. Two things to say plainly. First, the network lost in all three targets — to boosting on 40 descriptors in the mirror pair, and to a logistic regression on eight numbers in the reversal task. Fourth line of this project to end that way.
- **Direction: invisible in the move, visible at the turn** — If the book cannot tell a reversal from an ordinary loud moment, can it at least tell which way the market is going? We split the question in two, because the answers turn out to be opposite. Inside the body of a move — nothing.
- **A week the models had never seen** — Everything above is measured on data the models were trained on, out of fold. The last check is different: 2,506 windows recorded after the dataset was frozen — 6.54 days the networks could not have seen, scored with frozen weights and frozen thresholds, nothing retrained, nothing recalibrated.

### A Recipe, Not a Result: How We Clean an Order-Book Dataset

https://marketflowresear.ch/articles/a-recipe-not-a-result-how-we-clean-an-order-book-dataset/ · published 14 August 2026 · Dataset cleaning recipe

**Summary.** This is the least glamorous and most useful study of the series: not a finding, but a procedure. How to take a raw dataset of order-book windows and produce a clean one — 56,263 windows in, 31,212 out, 95% of the reversals still there — with every step checked, every trap written down, and the whole thing runnable on another coin by one command.

- **The chain: four narrowings** — The chain has four narrowings, each justified by the step before it. The important discipline: every removal is decided by a label found on the same data, so the reversal rate in what remains rises mechanically — from 2.26% to 2.64%.
- **Eight features, six, four** — Two of the steps ask a question worth asking on its own: which of the eight features are actually doing the work?
- **Where to stop, and how you know** — Where does this stop? Not at a number of steps, but at a diagnostic. The method's own ceiling is 0.65 for the largest share: above it, the split is "almost everything against a crumb", which reproduces itself out of nothing.
- **The recipe itself** — The whole procedure is published as an executable recipe — CLEANUP_DATASET.md, downloadable below.

### Throwing Away a Third of the Data and Keeping 98% of the Reversals

https://marketflowresear.ch/articles/throwing-away-a-third-of-the-data-and-keeping-the-reversals/ · published 14 August 2026 · The cleaned dataset

**Summary.** If a third of your data provably contains almost nothing you are looking for, it should not be in the training set. We removed 18,744 of 56,263 windows — everything two independent definitions of 'quiet' agree on — and kept 98.1% of all the reversal windows.

- **What was removed, and what it cost** — Two sets are subtracted, and they overlap, so what is removed is their union, not their sum: The second set is worth a note. The quiet-PCA label needs an external 28-day norm, so it does not exist for the first 28 days of the dataset — but the model does not need the norm, it only needs the window's own 1,680 numbers.
- **The cleaned set splits the same way** — Cluster the cleaned set exactly as before, and nothing structural changes. Two of the HDBSCAN classes contain exactly zero reversals, which is itself useful: those are shapes of the book that never sit on a turn.
- **What a reversal window actually looks like** — What does a reversal window actually look like? Averaged cell by cell over all 847 of them against the 36,672 others: All six flow features roughly double. Both const walls go slightly down.
- **Cutting finer: purity for coverage** — Finally, cut the cleaned set finer with k-means and watch the trade-off. Purity roughly doubles from K=3 to K=10 — and the number of reversals actually caught falls by three quarters. That is an exchange rate, not a discovery: no new class appears, the same "inflow" region is simply being sliced thinner.

### Twenty-Five Weeks, Twenty-Five Rebuilds: Which Clustering Survives

https://marketflowresear.ch/articles/twenty-five-weeks-twenty-five-rebuilds-which-clustering-survives/ · published 14 August 2026 · Week by week

**Summary.** A clustering that only exists once is a story, not a result. So we rebuilt it 25 times — every week of the dataset from scratch, its own PCA, its own UMAP, its own clusters — and compared each week with the answer found on the whole dataset. One of the two methods reproduces. The other does not, and it is the fashionable one.

- **Rebuilt, not scored** — The protocol is deliberately harsh. A week is not scored by the global model — it is rebuilt completely: its own feature scale, its own PCA, its own UMAP, its own clusters, with min_cluster_size scaled to the week's size.
- **k-means: the same answer every week** — k-means reproduces itself, and it does so twice over. Every single week, independently, decided that its own data supports the same number of classes, and in a typical week the labels agree with the global ones on the great majority of windows.
- **HDBSCAN: two classes or seven, depending on the week** — HDBSCAN does not reproduce. Same method, same settings, same market — and depending on the week it returns two classes with almost no noise, or seven classes with two thirds of the data unplaced.
- **What survives, and what that means in practice** — What we keep from this. The axis is stable; the density-based subdivision of it is not. In practice that means: use k-means (or, better, the explicit threshold rules it implies) for anything that has to run continuously, and use HDBSCAN only to explore — with a full rebuild and a hard null every single time.

### Tops Hide, Bottoms Don't: an Asymmetry We Did Not Expect

https://marketflowresear.ch/articles/tops-hide-bottoms-do-not-the-asymmetry-we-did-not-expect/ · published 14 August 2026 · The falling half

**Summary.** This is the mirror of the previous study: the fall from a top to a bottom, 27,180 windows, the same pipeline to the letter. The structure repeats — and one number does not. Tops hide from the order book four times more often than bottoms do.

- **The mirror image of the previous study** — The zigzag with a 2.3% threshold cuts the price into legs: every leg runs from one turning point to the next.
- **The structure repeats exactly** — Same result as on the whole dataset, and that is the point of running it: the split does not change when we restrict it to one half of the move. The quiet classes hold almost no reversals at all, the noise holds 156 of them at ×0.50. Stability is 0.764 against 0.207 for the hard null.
- **Counted as turning points** — Counting reversal windows is misleading on its own. Windows step every 210 seconds, so up to nine of them fit into the ±15-minute band around a single pivot: one cluster of windows easily looks like nine separate events.
- **The asymmetry: a top is a quieter event** — Everything about the two halves matches: the same K, nearly the same class sizes, nearly the same enrichment of reversals. The asymmetry is in what the clustering fails to place. At a bottom the book always says something: not one bottom disappears completely.

### The Rising Half of a Move: What the Order Book Looks Like Between a Bottom and a Top

https://marketflowresear.ch/articles/the-rising-half-of-a-move-what-the-order-book-looks-like/ · published 14 August 2026 · The rising half

**Summary.** Half of every price move is a climb from a bottom to a top. We took only that half — 26,921 windows of the order book — and clustered it blind. The structure that comes out is the same one the whole dataset gives, which is good news: it means the classes are not an artefact of mixing rises and falls together.

- **Cutting the data by the geometry of the move** — The zigzag with a 2.3% threshold cuts the price into legs: every leg runs from one turning point to the next.
- **The same axis, restricted to one half** — Same result as on the whole dataset, and that is the point of running it: the split does not change when we restrict it to one half of the move. The quiet classes hold almost no reversals at all, the noise holds 42 of them at ×0.17. Stability is 0.795 against 0.146 for the hard null.
- **Counted as turning points, not as windows** — Counting reversal windows is misleading on its own. Windows step every 210 seconds, so up to nine of them fit into the ±15-minute band around a single pivot: one cluster of windows easily looks like nine separate events.
- **The reversals that fall into the noise** — Some reversal windows fall into the noise — 42 of them from 12 different turns. Are those special reversals the clustering failed to see? No. They are the quiet edge of the same event: A turn lasts long enough to produce both loud and quiet windows.

### One Axis, Again: Clustering 56,000 Order-Book Windows Without a Teacher

https://marketflowresear.ch/articles/one-axis-again-clustering-order-book-windows-without-a-teacher/ · published 14 August 2026 · Classes without a teacher

**Summary.** Take 56,263 windows of the order book, 210 seconds each, eight features per second. Tell the algorithm nothing — not the price, not the time, not our own signals. Ask only one thing: does this pile of windows fall apart into groups by itself? It does, along exactly one axis, and that axis is not the one anybody hopes for.

- **The method, and the three guards that make it mean something** — The recipe is deliberately boring, because the guards matter more than the method: PCA to 32 dimensions, then k-means with K from 2 to 10, and in parallel UMAP to 10 dimensions and HDBSCAN. The 2-D map you see in the charts is only a map — clusters are never searched for on it.
- **One axis: is there inflow, or is there not** — K=2, chosen by how far the silhouette beats the null rather than by the silhouette itself. The whole difference is the flow features — inflow and drain on both walls — while the const walls themselves stand at the same level in both classes and take no part in the division.
- **'Noise' is not a mystery, it is a level** — HDBSCAN leaves 72% of the windows unassigned, and "noise" sounds like "the algorithm gave up". So we asked whether the noise can be told apart from everything else — and it can, almost trivially.
- **Two independent definitions of a quiet market** — Cut the noise once more and its quiet half — we call it 1.1 — is 18,320 windows with 16 reversal windows in them: ×0.28 the base rate, which is to say practically dead. We already had a separate, independently trained detector of quiet market — "quiet PCA", built with an external 28-day norm. Do the two agree?

### One Network for Three Market States: What a Pooled Reversal Model Learns

https://marketflowresear.ch/articles/one-network-for-three-market-states/ · published 14 August 2026 · Pooled reversal model

**Summary.** We trained one network on the events of three different market states at once — 55,929 events, 3,758 of which sit on a real turning point — and asked it a single question: is this event a reversal or an ordinary event of its own class? The answer is interesting in a way we did not plan: the model works, the validation set lies about which model is best, and an unsupervised split of the same pool finds classes that have nothing to do with reversals at all.

- **One pool, three states, one subtraction** — Three detectors mark events in our data: storm (loud order book), calm (quiet), class 2 (the middle). Until now each had its own model.
- **The model works. The validation set lies.** — The split is blocked in time: the timeline is cut into 50 chunks, five go to validation and five to test, every other one, with a 30-minute embargo around each. Then 18 architectures are screened, the best three are trained to 48 epochs, and the answers are read at 11 checkpoints inside the run.
- **What the pool splits into on its own** — Now the same pool, but without any labels at all: PCA → UMAP → HDBSCAN, and our reversal marks applied only afterwards, as a check. The split is real — stability 0.813 on a full rebuild with a different seed, against 0.378 for a hard null where the cells are shuffled between events.
- **Every compression lost to one average** — Last question of this study: does the event need to be compressed before the model sees it? We ran PCA and an autoencoder at five bottleneck widths, with the same logistic head on top of both, so that what is compared is the compression and not the head.

### Three Classes, One Question: What It Takes to Separate a Reversal

https://marketflowresear.ch/articles/three-classes-one-question-what-it-takes-to-separate-a-reversal/ · published 13 August 2026 · Separating the signal

**Summary.** Three market states, one question: can an order-book event that sits on a turning point be told apart from the ordinary events of its own class? This is the overview of that work — the three per-class studies side by side, every engine we tried in between, and the parts that hurt: a grid search that bought 0.02 AUC, a network on 1,680 numbers that lost to a single average, and a threshold that looked like 63% precision until it was asked to survive two months of new data, where it became 39%.

- **Three classes and the ruler that makes them comparable** — The point of all of this is not a leaderboard. A trading bot needs one thing from research: a moment when the book says something is about to change with enough purity that a position taken on it survives the 0.30% round-trip fee. Everything below is an attempt to isolate that moment inside one market state at a time.
- **Everything we tried, and what it cost** — Before the per-class work, the same question was attacked head-on inside the storm class, with everything we had. This is the part worth reading if you are deciding whether to spend a week on architecture.
- **The threshold that did not survive time** — Here is the part that decides whether any of this can be automated. Take the best single-feature model and stop using the calibrated threshold.
- **The pain, and what is left standing** — What holds up. A reversal event is real and it is visible in the book.

### Class 2: the Middle State Where No Single Feature Wins

https://marketflowresear.ch/articles/class-2-the-middle-state-where-no-single-feature-wins/ · published 13 August 2026 · Class 2 reversals

**Summary.** The class 2 state fires on 21,704 windows of this dataset, and 1,961 of them (9.0%) sit on a turning point of price. This page asks one question and answers it with one frozen network: can a reversal be told apart from the ordinary events of its own class, and which of the eight order-book features carries that difference? Best single feature: vol_buy at 15.8% precision. Best set: 6 features at 23.5% precision and 23.6% recall, against a base rate of 9.0% — ×2.6.

- **What a reversal looks like inside the class 2 class** — The point of all of this is not a leaderboard. A trading bot needs one thing from research: a moment when the book says something is about to change with enough purity that a position taken on it survives the 0.30% round-trip fee. Everything below is an attempt to isolate that moment inside one market state at a time.
- **One network per feature** — If the signal is a level, the cleanest way to find out whose level it is to give the network one feature and nothing else: an input of exactly 210 numbers, the same frozen architecture, the same split, eight separate models.
- **Adding one feature at a time** — One feature at a time answers who, not how much together. So the sets are grown greedily: start from the best single feature, then at every round try each remaining feature as an extra input channel and keep the one that raises precision on the validation set.
- **What this class hands to a trading bot** — Put the two views together. On a test of 2,170 events the model fires about 196 times, and 23.5% of those firings are real reversals. In counts: roughly 46 hits, 150 false alarms, and about 150 reversals missed in silence. That is the shape of what a bot would actually receive from this class — not a verdict, a rate.

### Reversals in a Quiet Market: Where 2.5% of Events Hide

https://marketflowresear.ch/articles/reversals-in-a-quiet-market-where-two-percent-of-events-hide/ · published 13 August 2026 · Calm reversals

**Summary.** The calm state fires on 29,303 windows of this dataset, and 739 of them (2.5%) sit on a turning point of price. This page asks one question and answers it with one frozen network: can a reversal be told apart from the ordinary events of its own class, and which of the eight order-book features carries that difference? Best single feature: vol_buy at 10.8% precision. Best set: 8 features at 14.9% precision and 15.3% recall, against a base rate of 2.5% — ×6.0.

- **What a reversal looks like inside the calm class** — The point of all of this is not a leaderboard. A trading bot needs one thing from research: a moment when the book says something is about to change with enough purity that a position taken on it survives the 0.30% round-trip fee. Everything below is an attempt to isolate that moment inside one market state at a time.
- **One network per feature** — If the signal is a level, the cleanest way to find out whose level it is to give the network one feature and nothing else: an input of exactly 210 numbers, the same frozen architecture, the same split, eight separate models.
- **Adding one feature at a time** — One feature at a time answers who, not how much together. So the sets are grown greedily: start from the best single feature, then at every round try each remaining feature as an extra input channel and keep the one that raises precision on the validation set.
- **What this class hands to a trading bot** — Put the two views together. On a test of 2,930 events the model fires about 73 times, and 14.9% of those firings are real reversals. In counts: roughly 11 hits, 62 false alarms, and about 63 reversals missed in silence. That is the shape of what a bot would actually receive from this class — not a verdict, a rate.

### The Storm Class: Which Order-Book Feature Marks a Reversal

https://marketflowresear.ch/articles/the-storm-class-which-order-book-feature-marks-a-reversal/ · published 13 August 2026 · Storm reversals

**Summary.** The storm state fires on 5,249 windows of this dataset, and 1,085 of them (20.7%) sit on a turning point of price. This page asks one question and answers it with one frozen network: can a reversal be told apart from the ordinary events of its own class, and which of the eight order-book features carries that difference? Best single feature: vol_buy at 49.5% precision. Best set: 5 features at 54.1% precision and 50.0% recall, against a base rate of 20.7% — ×2.6.

- **What a reversal looks like inside the storm class** — The point of all of this is not a leaderboard. A trading bot needs one thing from research: a moment when the book says something is about to change with enough purity that a position taken on it survives the 0.30% round-trip fee. Everything below is an attempt to isolate that moment inside one market state at a time.
- **One network per feature** — If the signal is a level, the cleanest way to find out whose level it is to give the network one feature and nothing else: an input of exactly 210 numbers, the same frozen architecture, the same split, eight separate models.
- **Adding one feature at a time** — One feature at a time answers who, not how much together. So the sets are grown greedily: start from the best single feature, then at every round try each remaining feature as an extra input channel and keep the one that raises precision on the validation set.
- **What this class hands to a trading bot** — Put the two views together. On a test of 525 events the model fires about 109 times, and 54.1% of those firings are real reversals. In counts: roughly 59 hits, 50 false alarms, and about 54 reversals missed in silence. That is the shape of what a bot would actually receive from this class — not a verdict, a rate.

### We Went Looking for the DNA of a Reversal. We Found a Ceiling Instead.

https://marketflowresear.ch/articles/we-went-looking-for-the-dna-of-a-reversal-we-found-a-ceiling-instead/ · published 12 August 2026 · Reversal DNA

**Summary.** An earlier study on this dataset (175 days of BTC) found 271 order-book events that sit exactly on the turning points of price, and showed that a model cannot separate them from the thousands of ordinary events of their own class. This study asks a narrower question: forget separating a reversal from its class — is there anything, on its own, that a reversal event's 271 examples share? Nine different methods attack the question from nine angles: a two-condition threshold rule, a raw-vector kinship check, a full neural detector, a scan for the shortest useful slice of the window, four models on four quarters of it, one model on all of its 30-second slices at once, and three independent models — one per parent market state. Every method finds something real, reproducible, and well above every null control we built. None of them clears the bar a trading signal needs. The best operating point we found, buried inside the loudest of four market states, is right roughly one time in three.

- **What we are looking for, and why it is a different question** — The predecessor study, We Found the Reversal Signal. Machine Learning Could Not Tell It Apart, took four detectors trained on this dataset — storm, class 2, calm and PCA-quiet, from an earlier study — and found 271 groups of their firings that sit on a turning point of price (ZigZag 2.3%, ±15 minutes).
- **The cheapest hypothesis first: a two-condition rule** — Before training anything, we asked the simplest possible question in the style this project uses for its live rules elsewhere on the dashboard: is there a 1–2-condition AND rule on 11 group-level numbers (8 average feature levels, group size, duration, sharpness) that separates a class's reversal groups from the rest of its own firings?
- **Are the 271 events even related to each other?** — A cheaper check than training anything: are the 3734 windows that make up the 271 events more similar to each other than to a random window?
- **The full network, the whole window, against the whole dataset** — Group-level rules were the cheapest hypothesis.
- **Where in the window does the signal live?** — If a 1680-number window carries this little signal, maybe most of it is dead weight.
- **One model, all seven slices, at once** — Instead of four separate models, we cut every window into 7 non-overlapping 30-second slices and trained one model to recognise a "reversal slice" on all 7 positions together — 393,841 examples instead of 56,263.
- **Splitting by parent class: where the ceiling actually sits** — Section ③ already hinted that "reversal" means something different in each parent class — storm has some family resemblance, calm has none.
- **What we tried and put aside** — Three more approaches were tried on the group-level representation of section ② before we moved to full-window networks, and all three are left in the code but not on the dashboard tab, because none improved on what is reported above.
- **What this is worth** — Nine methods, one direction of travel. A group-level threshold rule barely clears random. A full network on the whole window reaches ×2.9 lift at 20% precision. Splitting the window by position changes nothing — the same signal is there whether you look at the first 30 seconds or the last.

### We Found the Reversal Signal. Machine Learning Could Not Tell It Apart.

https://marketflowresear.ch/articles/we-found-the-reversal-signal-machine-learning-could-not-tell-it-apart/ · published 9 August 2026 · Signals

**Summary.** A market moves in legs: a few percent one way, a turning point, then a few percent the other. This study asks whether the order book carries an event that marks the turn — and whether that event can be separated from the thousands of ordinary events of the same class. On 175 days of BTC the first half works: groups of signals pile up at turning points 3.7× more often than chance, every class shows the same reversal signature, and inside a storm the shape of the flows tells a top from a bottom where price itself says nothing. The second half does not. Every way we tried to learn 'is this a reversal' landed on top of its own null control, and a class retrained with all reversal windows removed still fires on them — harder than on its ordinary events. The signal is real; the discriminator is not.

- **Legs, groups, and one rule that shapes everything** — Two markings of the same seconds are laid over each other. The first is price phase. A centred 5-minute average smooths the tick noise, a 2.3% ZigZag marks the turning points, and the stretch between two turning points is a leg.
- **The signal: storm groups pile up at the turns** — The null control matters more than the observation here. There are many groups and few turning points, so overlaps happen on their own. So the entire sequence of a class's groups is circularly shifted against price 999 times, and the same count is repeated.
- **The signature of a reversal — the same in every class** — Every group is reduced to one average matrix of 8 features × 210 seconds, and the buckets of groups (up leg · down leg · bottom→top · top→bottom · outside legs) are compared.
- **Shape: the one place the book beats price** — If level is all there is, then the arrangement inside the 210 seconds carries nothing. So we measured it directly. Each feature of a window gets five descriptors: level (mean), slope (OLS), third third minus first, curvature, and sharpness (max minus mean). Then: top versus bottom. Two safeguards.
- **Where it stops: three grids and a null control** — Everything above is description. Turning it into a decision means a classifier, so we built three of them, each with its own full grid (feature sets × model families × hyperparameters: logistic, boosting, forest, kNN, RBF-SVM, dilated Conv1D on the raw 8×210). Three rules keep this honest.
- **The decisive test: classes retrained without reversals** — If a reversal event were a distinct subtype of its class, a detector that had never seen one would not fire on it.
- **Why the negatives are contaminated: pullbacks inside a leg** — There is a structural reason a classifier is being set an unfair question here, and it is worth measuring rather than asserting. A leg is a move from one turning point to the next, but it does not go straight: price keeps falling back against it.
- **What this is worth** — Found. Groups of storm signals concentrate at turning points 3.7× above chance and survive a 999-shift null. The reversal signature is identical across classes — all flow features up, walls down — and strongest in the quiet ones.

### Sixteen Networks, One Working Answer — and the Price Benchmark That Nearly Killed It

https://marketflowresear.ch/articles/sixteen-networks-one-working-answer/ · published 8 August 2026 · Nets

**Summary.** We trained sixteen one-class autoencoders on Bitcoin order-book windows: four market states, and inside each state one network per phase of the price move. Each network saw only its own kind of window and nothing else — no negative examples at all. One thing out of sixteen works: inside a storm you can tell a bottom reversal from a top one. On half the data that looked like AUC 0.6264; measured properly on all 800 events it is 0.5928. Then a one-line price benchmark scored 0.8548 on the same question. This article is about what survives that, and where it survives.

- **Sixteen networks that never saw a counter-example** — Two markings of the same seconds go into this study, and neither is produced here. The market state comes from four detectors trained in an earlier study — storm, PCA-quiet, class 2 and calm — applied to windows of 8 order-book features × 210 seconds, out-of-fold, so every window is judged by a model that never saw it.
- **One thing works, and only in one class** — On the validation half, three questions are asked of the same four errors: whose phase is this (four-way, chance 0.25), bottom reversal versus top, and reversal zone versus the body of a leg.
- **Making the number usable — and watching it shrink** — Four changes, in order. Double cross-fitting. In the first pass, networks learned on the first half of days and were measured on the second, so half the events had no score at all. Now a mirror pair of networks is trained on the validation half and used to score the training half.
- **The benchmark that nearly kills it — and the one place it does not** — Price inside the same 210-second window scores 0.4722 on "top or bottom" — nothing, exactly as expected: at an extremum price does not move for three and a half minutes. But there is a second price benchmark, and it is not polite.
- **A hypothesis that failed: networks on subclasses** — An earlier study showed that in the quiet classes, events inside reversal zones are stronger — the top tercile of intensity turns up there more often.
- **What this adds up to** — Four things survive. One. Out of sixteen one-class networks, one cell reads: direction of a reversal inside a storm.

### Does the Phase of a Move Change What the Order Book Looks Like?

https://marketflowresear.ch/articles/does-the-phase-of-a-move-change-what-the-order-book-looks-like/ · published 7 August 2026 · States

**Summary.** We have four detectors of market state and a map of where price is in its move — reversal zones and the legs between them. Laying one over the other answers a question that sounds simple and is not: does the phase of a move change what a market state looks like? On 171 days of BTC the answer splits. The count of events barely moves — each class fires roughly in proportion to how long the phase lasts. What moves is their strength, and only for the quiet classes. And in one narrow place the order book finally beats price: telling a top from a bottom inside a storm.

- **Two markings of the same time** — Everything here compares two labellings of the very same seconds. The first is the market state: a window of 8 order-book features × 210 seconds, classified by four detectors trained in another study — storm, PCA-quiet, class 2 and calm. Answers are taken out-of-fold: every window is judged by a model that never saw it.
- **How often each state fires in each phase** — The simplest question first: does a class fire more often in a reversal than in the body of a move? Each model is compared only against itself — their bases differ by an order of magnitude (9.3% of the time for storm, 61.4% for calm), so the table shows lift, not raw rates.
- **The features themselves, phase by phase** — So far we compared what the models said. Now we compare what they saw: the 8 × 210 matrix in exactly the form it is fed in.
- **Inside each class: the 1680 numbers, and the honest benchmark** — Per-class, per-second now.
- **The strongest events are the least characteristic** — Next we cut every (class × phase) group into three parts inside itself: the 34% most distinct events, the 33% typical ones, and the 33% weakest. Two orderings, because "most distinct" reads two ways — by intensity (mean z of the eight levels) and by atypicality (Mahalanobis distance from the group's own centre).
- **The count barely moves — the strength does** — Finally, the plain bookkeeping: how many events of a class fall into each phase, and how strong they are. The terciles here are cut across the whole class, not inside the phase — otherwise every column would read 34/33/33 by construction and there would be nothing to compare.
- **What this adds up to** — Three things survive from all of the above. One. The phase of a move does not decide whether a state appears — the counts follow the clock. It decides how strong the state is when it appears, and only for the quiet classes. Storm is indifferent to the phase. Two.

### Eight Ways to Compress 210 Seconds: the Autoencoder That Lost to PCA

https://marketflowresear.ch/articles/the-autoencoder-that-lost-to-pca/ · published 6 August 2026 · Autoencoder

**Summary.** We cut 170 days of per-second Bitcoin order-book data into 50,143 back-to-back windows and pushed them through eight different compressions — PCA, a convolutional autoencoder, and six variants of both — to see whether a network finds market structure a linear method misses. It does not. Its raw stability is the best in the table and most of that stability survives when the data is shuffled into noise, which means it was never structure at all.

- **A dirty tape, cut end to end** — This study asks one question: when the whole tape is cut into windows — not just the interesting moments — does a neural network find structure that a linear method cannot? Before any model, the data had to be made honest.
- **Nine configurations, and how little they mattered** — The autoencoder is a 1-D convolutional network: eight features as eight channels, convolution along time only, trained to reconstruct the window it was given. Nine configurations were trained — three bottleneck widths (16, 32, 64), three depths, three noise levels, three learning rates.
- **Where the training actually stopped** — The production model was then trained properly: 308,808 weights, 72 epochs, and the weights kept are those of epoch 64 — the point where validation loss stopped improving while training loss kept falling. Final validation loss 0.10476.
- **What the network reproduces — and what it gives up on** — Reconstruction loss in a standardised scale is unreadable, so it was converted into something you can argue with: by what factor does the reconstruction miss the true value, feature by feature, on validation windows. The two resting walls come back at ×1.35 — nearly exact. The four flows miss by ×5.08 to ×6.17.
- **Eight compressions of the same windows, one rule** — The same 50,143 windows were then pushed through eight different compressions — PCA, the autoencoder on all eight features, the autoencoder on each half of the book, two coarse-grained variants, and the recipe borrowed from the neighbouring market-states study — and clustered identically downstream.
- **The null control eats the network's advantage** — Read the red bars in the chart above and the case against the autoencoder is finished. On all eight features the network reaches stability 0.709 — the kind of number that gets published. Its null control reaches 0.599.
- **Where classes do survive** — Two spaces come through with both a real gap and a legal partition.
- **What we take from this** — The network was given every advantage — its own architecture search, a blocked split with an embargo, 308,808 weights, and the same downstream clustering as everything else — and it did not earn its place.

### Two Poles of Market Activity, Found Without a Teacher on Six Coins

https://marketflowresear.ch/articles/two-poles-of-market-activity-found-on-six-coins/ · published 6 August 2026 · Market states

**Summary.** We cut 122–141 days of per-second order-book data on 6 coins into 164,342 windows of 210 seconds, showed a clustering pipeline nothing but the eight order-book features, and asked whether the market falls into natural states. It does — two poles of flow activity with a wide grey zone between them — and the same pipeline run on shuffled data produces nothing, which is what makes the answer worth reporting. The walls turn out to be irrelevant to the split, and the autoencoder we brought in to beat PCA lost to it.

- **One question, and a deliberately blind input** — Does the market fall apart into natural states? Not into the states we invented — into whatever the data itself insists on, if we take away every crutch. So the input was made deliberately blind.
- **Clustering always returns something — so we ran it on nothing** — Hand a clustering algorithm pure noise and it will hand you back tidy clusters. That is not a risk to be mentioned in a footnote; it is the default outcome, and it is why most "market regimes" you read about are decoration. So every number here is measured twice. Once on the data.
- **The autoencoder was not needed** — The obvious objection is that PCA is a linear rotation of axes and the market is not linear. So the identical pipeline was run with a convolutional autoencoder in place of PCA — eight features as eight channels, convolution along time only, trained on each coin, everything downstream unchanged.
- **What actually separates the two poles** — Now the interesting part. The two poles are not "high price movement" and "low price movement" — price was never shown to the model. So what is different about them? Feature by feature, with the band covering the 5th to 95th percentile of each class, the answer on Bitcoin splits the eight features cleanly in two.
- **Six coins, one answer — and one coin that argues** — A finding on one coin is an anecdote. The same pipeline was therefore run independently on all 6, with no shared parameters beyond the recipe. Two things repeat everywhere.
- **The strongest single piece of evidence** — The clearest argument that the poles are real is not a metric at all. It is that two methods with nothing in common find the same ones. Take only the windows where both engines — the linear rotation and the convolutional network — confidently assigned an extreme state.
- **What this is not** — Four limits, because a result without its boundaries is advertising. It is not a full partition of the market. The named classes cover roughly 40–50% of the time. The rest — 61% on Bitcoin — is a continuum with no dense clumps in it. The honest statement is "two edges and a long middle", not "the market has N states".
- **What we take from this** — Three things survive from this study. The market does have two ends of an activity scale, they are found without a teacher, they are found on 6 coins independently, and the pipeline that finds them produces nothing when the structure is removed.

### Local tops and bottoms in the order book: what four trained models actually see

https://marketflowresear.ch/articles/local-tops-and-bottoms-what-four-trained-models-actually-see/ · published 6 August 2026 · Pivots

**Summary.** We marked every local top and bottom on 171 days of per-second Bitcoin order-book data — 734 of them, defined by a rule rather than by hand — and asked four trained market-state detectors whether they can see them. The zone itself is invisible. The moment of the extremum is not, and neither is 'an extremum within the next hour'. But a plain price range from the same window beats the order book at both, and the honest edge turns out to live somewhere else entirely: in telling a top from a bottom when price itself has nothing to say.

- **What we marked, and why exactly this way** — A local top is not a point on a chart — it is a place where price stops going up and starts going down. To find those places without hand-picking them, we use a ZigZag with a 1.0% threshold: a new extremum is registered only once price has moved away from it by at least that much.
- **What it looks like up close** — Zoomed in to a few days, the structure is obvious to the eye: green boxes sit on the tops, red boxes on the bottoms, and between them are the legs — the moves of at least 1% that connect one extremum to the next. This picture is not decoration. It explains half of the results that follow.
- **Can four trained models see a zone?** — We have four market-state detectors from a separate line of work: storm, class 2, calm and PCA-quiet. Each takes the raw 8x210 matrix of one 210-second window and returns a probability.
- **Where the real hooks are** — Instead of one question we asked five, and the useful answers are all in the details. The order book marks the edges of a zone, not the zone. Inside a zone the storm probability starts high, collapses below baseline in the middle, and rises again at the exit.
- **Two chains: everything after which price fell, against everything after which it rose** — We stitched all the top pieces into one chain and all the bottom pieces into another, then compared them feature by feature.
- **The same signal in four different market states** — Every 210-second window belongs, by its midpoint, to one of four states: a top zone, a bottom zone, the body of an up leg, or the body of a down leg.
- **Inside the classes: the 1680 numbers the network actually sees** — So far we compared what the network said. Here we compare what it saw: the 8x210 matrix in exactly the form it is fed in.
- **What this adds up to, and what comes next** — We have found classes, and they are real: found without supervision, reproduced by a second independently written pipeline, and detectable by trained models with out-of-fold AUC in the high 0.9s.

### Four Market States, Found Twice by Two Independent Pipelines

https://marketflowresear.ch/articles/four-market-states-found-twice-by-two-independent-pipelines/ · published 6 August 2026 · Replication

**Summary.** Four market states, found twice by two pipelines that share no code. We wrote our research recipe down as an executable diary, then wrote a second implementation from that text alone — different seed, different block split, different network initialisation — and ran it over the same 54,764 windows of BTC order-book data. The states came back: 93.0% of windows landed in the same one (ARI 0.7726), and two independently trained detectors fire on the same 98.5% of windows. Where both fire, the next hour's price range is ×1.432 the median of the same day.

- **The data both pipelines saw** — Everything below runs on one input: per-second order-book features for a single coin — wall thickness, inflow and drain on each side, and executed buy/sell volume. Eight numbers a second, 171 days of BTC, cut edge to edge into windows of 210 seconds.
- **Four states: what the algorithm says, what the model says** — The line trains four production targets, and this is all four of them on one shared time axis. Green is the state as the algorithm assigned it; red is the state as a model trained on that assignment reads it back out of the raw 8 × 210 matrix.
- **Two independent runs across the whole history** — The top panel is the storm state as each pipeline assigned it, day by day, over the full 171 days; the bottom panel is the two detectors firing.
- **Twelve hours, window by window** — Close up, on a slice of history at the natural resolution of the method: four rows — my state, their state, my model's probability, their model's probability — over the same 210-second windows.
- **Where the two runs disagree, and what price did next** — The left half is the state matrix — my run's rows against the other run's columns, as a share of all windows. The diagonal holds 93.0% of them. The right half asks the question a single model cannot answer: what happens in the off-diagonal cells?
- **Conclusion: the states are real, and we are going to use them** — As we can see, the states are there.

### There Are No Classes in the Order Book — Only One State That Survives

https://marketflowresear.ch/articles/no-classes-in-the-order-book-only-one-state/ · published 5 August 2026 · Window classes

**Summary.** We locked a 210-second window to its own 1,680 numbers — eight order-book features, no norm, no price, no clock — and asked whether the market falls into classes. It does not: the gap statistic sits at 0.0017, silhouette matches noise, and at some K the shuffled null is more stable than the data. One state survives every check: 9.8% of windows where flows run hot and both walls thin out, followed by ×1.78 the usual hourly range.

- **One window, 1,680 numbers, nothing else** — The constraint came first and everything follows from it: a 210-second window may be classified only by its own 1,680 numbers — eight order-book features across 210 seconds. No multi-day normalisation, no price, no time of day, no neighbouring windows.
- **Before naming classes, ask whether classes exist** — Clustering always returns something, so the first job is to test the premise. Four independent criteria, each with its own null control run through the entire pipeline. The gap statistic at K=2 comes out at 0.0017 — a partition compresses the data no better than noise does.
- **Two respected criteria that lie here** — The bootstrap likelihood-ratio test rejects K=1 with LR = 23,673 against a null distribution of 123. Overwhelming — until you run the same test on shuffled cells, where it rejects K=1 just as confidently with LR = 14,117. It measures non-Gaussian marginals, not clusteredness.
- **The structure is real — it just isn't classes** — None of this says the data is empty. Sixteen axes capture 83% of the descriptor variance against 62% on the null; on the raw 1,680-number vector it is 30.6% against 1.2%. The structure is simply continuous rather than discrete. A ladder of nulls locates it.
- **Without an external norm, you classify the calendar** — Here is the trap that costs an entire study if missed. With no multi-day normalisation, the loudest signal in the data is the general activity level, and that drifts from month to month. Cramér's V between class label and month is 0.35; one class holds 63% of the first half of history and a fraction of the second.
- **One state survives everything** — What is left is a single genuine state, and it is not subtle: 9.8% of windows in which all six flow features run 1.4–1.9× their own scale while both resting walls sit thinner than normal. The book is coming apart while trade keeps running.
- **Three comparisons you can run on any window** — The state distils into three thresholds on geometric feature levels — no model, no matrices, eight frozen constants and three numbers: √(resist_minus · support_minus) > 1.417, √(resist_plus · support_plus) > 0.909, √(const_resist · const_support) < 1.234.
- **Teaching a machine to see the same thing** — Four targets, input the full 8 × 210 raw matrix, transformed only inside the window. The dataset is cut into ten random parts; each window is scored exactly once by a model that never saw it. The convolutional network wins on every target and every fold — for the sharp state AUC 0.9968 (PR-AUC 0.9746).
- **Reproduce it on your own data** — Everything above is a procedure, not a table of results. A class here is never a list of windows — it is defined by its rank in feature-flow level, so the same recipe recomputes it on another coin, another period, a longer history.

### Event Classes Found Without a Teacher: 3 Survive, 60% Are Noise

https://marketflowresear.ch/articles/event-classes-found-without-a-teacher/ · published 3 August 2026 · Event classes

**Summary.** We asked the order book to sort its own events into classes — 2,786 events, eight features, no price and no human labels. Out of 512 configurations only 5 produced anything stable, 60% of events belong to no class at all, and the three that do exist raise volatility while leaving direction at a coin flip.

- **Let the data name its own events** — Every earlier study started from a human definition: a spike is this, a wall is that. Here we did the opposite.
- **Most configurations find nothing — and that is the point** — Clustering will always return something, so the real work is deciding what to believe. We swept 512 configurations and demanded stability, not beauty: only 5 survived the suitability filter. The winner explains 73% of input variance and lands on k=3 classes with bootstrap ARI 0.85 (sd 0.019).
- **Beating the shuffled null** — A stability number means nothing on its own, so we ran the same pipeline on time-shuffled and fully shuffled data. Real data: ARI 0.61 on the full bootstrap and 0.67 across folds. Shuffled null: 0.33. The gap is the whole claim of this study — the structure is in the book, not in the method.
- **Sixty percent of events belong to nothing** — The honest headline: 60% of events are noise — HDBSCAN refuses to assign them to any class, and we do not force it. Only 40% form stable groups. A classifier that labelled everything would look far more impressive and be far less true.
- **The three classes that do exist** — What survives: class 0 — 546 events (19.6%), led by drain out of the sell wall; class 1 — 363 events (13.0%), led by the resting buy wall; class 2 — 89 events (3.2%), led by the resting buy wall.
- **How the classes differ, feature by feature** — The same difference stated as levels rather than curves: for each class, how far each feature sits from its own normal. This is the most compact description of what the algorithm found.
- **How long an event lasts** — Median event duration is 7.7 minutes, and the distribution is wide and skewed — the classes differ in duration as clearly as they differ in shape.
- **What happens after — the part price is allowed to answer** — Only now does price enter. Forward volatility after an event averages 0.0396 and the forward range 0.442, both meaningfully above the quiet baseline. Direction: 49.6% of events are followed by an up move — a coin flip, again, on classes that were found without ever looking at price.
- **Where the classes sit on the price line** — The discovered classes drawn over the whole dataset, plus a zoom. Useful as a sanity check: the classes are not concentrated in one regime or one month.
- **Transitions between states** — Which state hands over to which, and how the sides balance inside each one. Note what is absent: no forecast is claimed here — transitions are a description of the sequence, not a prediction of it. _Research, not financial advice._

### The Alphabet of the Order Book: 6,561 Possible States, 333 Real Ones

https://marketflowresear.ch/articles/the-alphabet-of-the-order-book-signal-states/ · published 3 August 2026 · Signals

**Summary.** We described the order book as a single word: eight features, three states each. Of the 6,561 words that could exist, only 333 ever occur — and when we tested that alphabet out of sample, it described the market well and predicted it poorly.

- **A signal is a state of the whole book, not one number** — We stopped looking at features one at a time. At every moment each of the 8 order-book features is in one of three states — spike, normal or lull — so the market's condition is an eight-character word. That gives 6,561 possible words, of which only 333 ever actually occur across 12,477 observed windows.
- **Most of the time, nothing is happening** — One word dominates everything: all-normal, every feature quiet at once. It covers 6,351 of 12,477 windows — about 51% of the time — and its forward move is the smallest in the set (median 0.097%, up-share 49%).
- **What an active state looks like** — The most common non-quiet word is 01101111 — 864 windows. Its median absolute move is 0.142% against 0.097% for the quiet state, i.e. roughly ×1.5 more movement, while its up-share sits at 48% — near a coin flip. The pattern repeats at every level of this project: state tells you the amplitude, not the sign.
- **One feature alone versus several at once** — An isolated signal is one feature leaving its baseline while the other seven stay normal. They are informative in a narrow way: the strongest directional tilt belongs to executed sell volume (median -0.204%, up-share 18%, n=103), and the largest forward movement to build-up into the buy wall (median 0.250%).
- **How fast the information dies** — Every signal has a shelf life. Measured across horizons 5s, 60s, 5m, 1h, the advantage is concentrated in the first minutes and is gone well before the hour mark — consistent with the volatility studies, where flow-based edge decayed on the same clock.
- **The states drawn over price** — Rather than trusting the labels, look at them: the state sequence laid directly over BTC price. Long stretches of quiet, bursts where several features light up at once, and the price behaviour around those bursts.
- **The honest test: does it survive out of sample?** — This is the section that matters. We split the history in half, learned the behaviour of each state on the first part (6,238 windows) and applied it blind to the second (6,239). Direction AUC falls from 0.623 in-sample to 0.554 out-of-sample; the 'will it move at all' model falls from 0.576 to 0.492 — i.e.
- **What we take from this** — Three things. The market's state space is far smaller than it could be — 333 words out of 6,561. Quiet dominates, so any live rule spends most of its time doing nothing.

### From Calm to Calm: a Standard for What Counts as a Signal (BTC)

https://marketflowresear.ch/articles/signal-standard-from-calm-to-calm-btc/ · published 3 August 2026 · Signal standard · BTC

**Summary.** We stopped defining market signals with a stopwatch. A signal is a continuous departure from calm, bounded by calm on both sides — the market decides where it starts and ends. Here is the standard, applied second by second to 167 days of BTC, with every event classified and located on the price path.

- **A signal is what sits between two calms** — Most signal definitions start with a stopwatch: look at the last fifteen minutes, the last hour, the last day. The market does not run on our clock. So we let it draw its own boundaries. Each of the eight order-book features gets a rolling 100-hour norm — its own average over the recent past, nothing forward-looking.
- **What "normal" actually means** — Our first definition of normal was "every feature within ±5% of its 100-hour average". That band turns out to be empty: it covers 0.00% of all time. Order-book features are heavy-tailed — a five-minute level routinely swings tens of percent while nothing notable happens — so the band has to be measured, not assumed.
- **The anatomy of one event** — Here is a single signal, drawn as multiples of each feature's own norm. Everything starts flat near 1×, one or more features lift off, the burst runs, and the event closes only when the whole book has settled back.
- **Duration is the first axis** — Event length spans three orders of magnitude, so we cut it into terciles measured on this coin: short (up to 13 min), medium (to 30 min) and long. The distribution is strongly skewed — most signals are minutes, a few run for hours, and the longest class (INFL·L·3) has a median of 186 minutes.
- **The class system** — Each event is labelled LEADER-GROUP · DURATION · INTENSITY. The leader group is whichever family deviated most from its own norm: the resting walls (WALL), inflow into them (INFL), drain out of them (DRAIN), or executed volume (VOL). Intensity is the peak over norm, again in terciles.
- **Who leads the burst** — Executed volume leads most events — VOL heads 539 bursts against 141 for the walls, 130 for drain and 87 for inflow. That is the expected order: trades move faster than resting liquidity. The more interesting split is inside the volume family.
- **Pair dominance, measured against each feature's own norm** — Inside each pair we ask who deviated more — but relative to its own norm, not in raw size. Raw comparison would be useless: inflow is nearly always bigger than drain in absolute terms, so the "winner" would be fixed in advance.
- **Where each class shows up** — Now the location test. A ZigZag with a 1.2% reversal threshold splits the history into 213 rising stretches (1986 h) and 214 falling ones (2013 h) — almost exactly balanced, so counts can be compared directly.
- **What actually separates rising from falling** — The location test is blunt about which parts of the passport carry information: bursts where buy volume deviated more sit in rising stretches 64% of the time (n=373, q=0.000), while sell-dominated bursts sit there only 39% (n=353, q=0.000); the same split by leader: vol_buy-led 62% rising vs vol_sell-led 39%; long…
- **The other half: the calm states** — Between any two events sits a calm state, and it deserves the same treatment. There are 4255 of them here, median 11 minutes, together 47% of the clock.
- **Is there a lull before the storm?** — The folklore says a long silence charges the market up.
- **What this is not** — ZigZag pivots are known only after the reversal has happened, so "this class lives in rising stretches" describes the past — it is not a forecast and cannot be traded as one. Class-level skews are thin at the tails and mostly do not survive multiple-comparison correction; the slice-level tests are the honest layer.

### Side Against Side: Resistance and Support Compared

https://marketflowresear.ch/articles/resistance-against-support/ · published 31 July 2026 · Side vs side

**Summary.** We put all six order-book features grouped into the resistance side and the support side into one study and asked how the pieces relate, which one leads, and what the combination predicts. Same boundary as always: it speaks about size, not side.

- **The pieces, measured separately** — This study looks at all six order-book features grouped into the resistance side and the support side. Before combining anything we show each component on its own scale, per coin — otherwise a composite hides which part is actually doing the work.
- **How the pieces relate** — Cross-correlation between the components, averaged over coins. The strongest pairs are buy-wall build-up vs buy-wall drain +0.92; sell-wall build-up vs sell-wall drain +0.92; sell-wall drain vs buy-wall drain +0.80.
- **Which one moves first** — Lagged correlation answers the ordering question: when one component shifts, does the other follow, and after how long? Profiles that peak away from zero lag indicate a lead-follow relationship rather than a coincidence.
- **What the combination predicts** — The same volatility sweep as in every study, now for the combined view: link strength against the size of the next move across horizons, per coin. As everywhere else, the usable signal sits at the short end and concerns magnitude.
- **Where it peaks** — The best window per coin for the combined reading.
- **Impact per unit** — Price impact per +1σ of each component, side by side — useful to see which piece carries the mechanical push and which is merely correlated with it.
- **States and transitions** — Labelled states for the combined reading, and how they hand over to one another.
- **Memory of the combination** — Autocorrelation of the combined reading by timeframe — the same stickiness we see in every single feature, preserved after combining.
- **Extremes** — How far above normal the extremes of this combination run. _Research, not financial advice._

### Flow Against the Wall: When Volume Meets Resting Liquidity

https://marketflowresear.ch/articles/flow-against-the-wall/ · published 31 July 2026 · Flow vs wall

**Summary.** We put executed volume hitting the resting wall on the opposite side into one study and asked how the pieces relate, which one leads, and what the combination predicts. Same boundary as always: it speaks about size, not side.

- **The pieces, measured separately** — This study looks at executed volume hitting the resting wall on the opposite side. Before combining anything we show each component on its own scale, per coin — otherwise a composite hides which part is actually doing the work.
- **How the pieces relate** — Cross-correlation between the components, averaged over coins. The strongest pairs are buy volume vs sell volume +0.52; the sell wall vs sell volume +0.18; the sell wall vs the buy wall +0.17.
- **Which one moves first** — Lagged correlation answers the ordering question: when one component shifts, does the other follow, and after how long? Profiles that peak away from zero lag indicate a lead-follow relationship rather than a coincidence.
- **What the combination predicts** — The same volatility sweep as in every study, now for the combined view: link strength against the size of the next move across horizons, per coin. As everywhere else, the usable signal sits at the short end and concerns magnitude.
- **The 2×2: thin vs thick, quiet vs busy** — Splitting the market into four cells — flow low/high against the wall thin/thick — gives the cleanest statement of the interaction. Mean absolute move per cell: low/thin: 0.0099, low/thick: 0.0093, high/thin: 0.0139, high/thick: 0.0139.
- **Where it peaks** — The best window per coin for the combined reading.
- **Impact per unit** — Price impact per +1σ of each component, side by side — useful to see which piece carries the mechanical push and which is merely correlated with it.
- **States and transitions** — Labelled states for the combined reading, and how they hand over to one another.
- **Memory of the combination** — Autocorrelation of the combined reading by timeframe — the same stickiness we see in every single feature, preserved after combining.
- **Extremes** — How far above normal the extremes of this combination run. _Research, not financial advice._

### The Life of a Sell Wall: Standing, Growing, Thinning

https://marketflowresear.ch/articles/lifecycle-of-a-sell-wall/ · published 31 July 2026 · Wall lifecycle

**Summary.** We put the three phases of the same sell-side wall — how much stands there, what arrives, what leaves into one study and asked how the pieces relate, which one leads, and what the combination predicts. Same boundary as always: it speaks about size, not side.

- **The pieces, measured separately** — This study looks at the three phases of the same sell-side wall — how much stands there, what arrives, what leaves. Before combining anything we show each component on its own scale, per coin — otherwise a composite hides which part is actually doing the work.
- **How the pieces relate** — Cross-correlation between the components, averaged over coins. The strongest pairs are sell-wall build-up vs sell-wall drain +0.92; the sell wall vs sell-wall build-up +0.32; the sell wall vs sell-wall drain +0.26.
- **Which one moves first** — Lagged correlation answers the ordering question: when one component shifts, does the other follow, and after how long? Profiles that peak away from zero lag indicate a lead-follow relationship rather than a coincidence.
- **What the combination predicts** — The same volatility sweep as in every study, now for the combined view: link strength against the size of the next move across horizons, per coin. As everywhere else, the usable signal sits at the short end and concerns magnitude.
- **Where it peaks** — The best window per coin for the combined reading.
- **Impact per unit** — Price impact per +1σ of each component, side by side — useful to see which piece carries the mechanical push and which is merely correlated with it.
- **States and transitions** — Labelled states for the combined reading, and how they hand over to one another.
- **Memory of the combination** — Autocorrelation of the combined reading by timeframe — the same stickiness we see in every single feature, preserved after combining.
- **Extremes** — How far above normal the extremes of this combination run. _Research, not financial advice._

### Executed Sell Volume, Measured Alone

https://marketflowresear.ch/articles/executed-sell-volume-alone/ · published 31 July 2026 · Sell volume

**Summary.** We isolated executed sell volume — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of executed sell volume** — Before asking what executed sell volume predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of executed sell volume with the size of the next move peaks at the fastest horizon (+0.23 on ETH at 5s) and decays monotonically toward the daily scale (+0.02). Direction is a different story: the same sweep against signed returns tops out around 0.12. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 5s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of executed sell volume, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of executed sell volume grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of executed sell volume starts at 0.14–0.25 at the finest scale and rises to 0.57–0.67 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no executed sell volume; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of executed sell volume runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where executed sell volume changes state, and what price does around it. _Research, not financial advice._

### Executed Buy Volume, Measured Alone

https://marketflowresear.ch/articles/executed-buy-volume-alone/ · published 31 July 2026 · Buy volume

**Summary.** We isolated executed buy volume — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of executed buy volume** — Before asking what executed buy volume predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of executed buy volume with the size of the next move peaks at the fastest horizon (+0.25 on ETH at 5s) and decays monotonically toward the daily scale (+0.04). Direction is a different story: the same sweep against signed returns tops out around 0.12. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 5s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of executed buy volume, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of executed buy volume grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of executed buy volume starts at 0.14–0.24 at the finest scale and rises to 0.57–0.67 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no executed buy volume; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of executed buy volume runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where executed buy volume changes state, and what price does around it. _Research, not financial advice._

### Drain Out of the Buy Wall: Liquidity Leaving Before the Move

https://marketflowresear.ch/articles/drain-out-of-the-buy-wall/ · published 31 July 2026 · Buy drain

**Summary.** We isolated drain out of the buy wall — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of drain out of the buy wall** — Before asking what drain out of the buy wall predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of drain out of the buy wall with the size of the next move peaks at the fastest horizon (+0.36 on BTC at 5s) and decays monotonically toward the daily scale (+0.07). Direction is a different story: the same sweep against signed returns tops out around 0.06. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 5s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of drain out of the buy wall, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of drain out of the buy wall grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of drain out of the buy wall starts at 0.29–0.44 at the finest scale and rises to 0.76–0.82 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no drain out of the buy wall; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of drain out of the buy wall runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where drain out of the buy wall changes state, and what price does around it. _Research, not financial advice._

### Build-Up Into the Buy Wall: What Arriving Liquidity Predicts

https://marketflowresear.ch/articles/build-up-into-the-buy-wall/ · published 31 July 2026 · Buy build-up

**Summary.** We isolated build-up into the buy wall — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of build-up into the buy wall** — Before asking what build-up into the buy wall predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of build-up into the buy wall with the size of the next move peaks at the fastest horizon (+0.38 on BTC at 5s) and decays monotonically toward the daily scale (+0.06). Direction is a different story: the same sweep against signed returns tops out around 0.10. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 5s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of build-up into the buy wall, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of build-up into the buy wall grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of build-up into the buy wall starts at 0.29–0.46 at the finest scale and rises to 0.77–0.82 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no build-up into the buy wall; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of build-up into the buy wall runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where build-up into the buy wall changes state, and what price does around it. _Research, not financial advice._

### Drain Out of the Sell Wall: Liquidity Leaving Before the Move

https://marketflowresear.ch/articles/drain-out-of-the-sell-wall/ · published 31 July 2026 · Sell drain

**Summary.** We isolated drain out of the sell wall — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of drain out of the sell wall** — Before asking what drain out of the sell wall predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of drain out of the sell wall with the size of the next move peaks at the fastest horizon (+0.37 on BTC at 5s) and decays monotonically toward the daily scale (+0.07). Direction is a different story: the same sweep against signed returns tops out around 0.05. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 5s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of drain out of the sell wall, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of drain out of the sell wall grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of drain out of the sell wall starts at 0.29–0.45 at the finest scale and rises to 0.75–0.82 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no drain out of the sell wall; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of drain out of the sell wall runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where drain out of the sell wall changes state, and what price does around it. _Research, not financial advice._

### Build-Up Into the Sell Wall: What Arriving Liquidity Predicts

https://marketflowresear.ch/articles/build-up-into-the-sell-wall/ · published 31 July 2026 · Sell build-up

**Summary.** We isolated build-up into the sell wall — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of build-up into the sell wall** — Before asking what build-up into the sell wall predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of build-up into the sell wall with the size of the next move peaks at the fastest horizon (+0.38 on BTC at 5s) and decays monotonically toward the daily scale (+0.07). Direction is a different story: the same sweep against signed returns tops out around 0.10. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 5s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of build-up into the sell wall, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of build-up into the sell wall grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of build-up into the sell wall starts at 0.31–0.46 at the finest scale and rises to 0.77–0.83 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no build-up into the sell wall; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of build-up into the sell wall runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where build-up into the sell wall changes state, and what price does around it. _Research, not financial advice._

### The Resting Buy Wall, Measured Alone

https://marketflowresear.ch/articles/the-resting-buy-wall-alone/ · published 31 July 2026 · Buy wall

**Summary.** We isolated the resting buy wall — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of the resting buy wall** — Before asking what the resting buy wall predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of the resting buy wall with the size of the next move peaks at the fastest horizon (-0.08 on BTC at 5s) and decays monotonically toward the daily scale (-0.03). Direction is a different story: the same sweep against signed returns tops out around 0.29. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 86400s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of the resting buy wall, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of the resting buy wall grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of the resting buy wall starts at 0.67–0.80 at the finest scale and rises to 0.73–0.91 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no the resting buy wall; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of the resting buy wall runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where the resting buy wall changes state, and what price does around it. _Research, not financial advice._

### The Resting Sell Wall, Measured Alone

https://marketflowresear.ch/articles/the-resting-sell-wall-alone/ · published 31 July 2026 · Sell wall

**Summary.** We isolated the resting sell wall — one feature, no partner, no composite — and measured what it predicts second by second across six coins. The answer is the size of the next move, never its side.

- **The shape of the resting sell wall** — Before asking what the resting sell wall predicts, we look at what it is: how the quantity is distributed across its own range, per coin. The profile is heavily skewed — long stretches of very little, punctuated by rare, very large readings.
- **What it predicts, across horizons** — Rank correlation of the resting sell wall with the size of the next move peaks at the fastest horizon (-0.08 on BTC at 5s) and decays monotonically toward the daily scale (-0.04). Direction is a different story: the same sweep against signed returns tops out around 0.29. Size, yes; side, no.
- **Where each coin peaks** — The best window differs by coin but always sits at the short end — 5s to 86400s. Anything slower than that is reading yesterday's news.
- **Price impact per unit** — Per +1σ of the resting sell wall, price moves by a fraction of a basis point, and the sign is not stable across bands. This is the quantitative version of 'it does not push price in a usable direction'.
- **What counts as normal** — Normal levels of the resting sell wall grow by orders of magnitude from a second to a day, so 'high' is always relative to a window. This chart is the baseline every later threshold is measured against.
- **Memory** — Autocorrelation of the resting sell wall starts at 0.68–0.80 at the finest scale and rises to 0.75–0.86 at intermediate windows. The quantity clusters in time: a busy minute follows a busy minute far more often than chance allows.
- **Where the seconds live** — Most seconds carry almost no the resting sell wall; the interesting behaviour is confined to a thin tail. Working with means here would describe a market that does not exist.
- **How large the extremes get** — A peak second of the resting sell wall runs many times above its own normal. That ratio is what makes threshold rules feasible at all — the extremes are unmistakable once the baseline is right.
- **Spike, normal, lull** — Every moment gets labelled relative to its own baseline. The three states separate cleanly in volatility and poorly in direction — the same result the whole project keeps producing.
- **The states over price** — The labelling drawn directly over a BTC price window: where the resting sell wall changes state, and what price does around it. _Research, not financial advice._

### The Support Cycle: Liquidity Arriving vs Leaving

https://marketflowresear.ch/articles/support-cycle-build-up-versus-drain/ · published 31 July 2026 · Support cycle

**Summary.** We measured the same buy-side wall growing and thinning second by second across six coins, and asked the only question that matters: does it tell you anything about what happens next? It does — about size, not direction.

- **Which side carries more of the support cycle** — Across the whole archive the two sides sit close together: the ratio between build-up into the buy wall and drain out of the buy wall runs 0.37–0.71× depending on the coin.
- **Does it predict the size of the next move** — This is the core question. Rank correlation with the size of the next move is positive on 6 of 6 coins, and strongest at the shortest horizon — +0.39 on BTC at 5s. By the daily scale it fades to +0.07.
- **How much price actually moves per unit** — We measure the price shift per +1σ of the support cycle, split into low, middle and high bands. The effect is small in absolute terms — fractions of a basis point per σ — and asymmetric between the two sides. That asymmetry is worth knowing, but it is nowhere near a tradable directional push on its own.
- **What counts as normal, by timeframe** — 'Normal' has no absolute value here: the support cycle measured per second is a different quantity from the same thing aggregated over an hour or a day, and the scale grows by orders of magnitude (log axis).
- **Where the link peaks** — Each coin has a window where the support cycle lines up best with what follows, marked with a star. The peaks cluster at the short end — the information decays quickly, so any use of this signal has to act on the scale of seconds to minutes rather than hours.
- **Does the imbalance predict direction** — Barely. Taking the signed difference between build-up into the buy wall and drain out of the buy wall and correlating it with the direction of the next move gives at most ρ≈0.22 anywhere in the grid, and it collapses toward zero on most coins and most horizons.
- **How long one-sidedness survives** — A lopsided book does not stay lopsided. The share of imbalance that persists falls from about 51–63% at one second to roughly 16–45% by the twenty-minute mark. Whatever pushes the support cycle to one side is undone quickly — which is exactly why it cannot steer longer moves.
- **Memory: does a busy second predict a busy second** — Yes, and this is the most robust property in the whole study. Lag-1 autocorrelation of the support cycle is 0.36–0.49 at the fastest scale and 0.29–0.40 at the slowest, on every coin. The quantity is sticky: its own past is a better predictor of its near future than anything about price.
- **How extreme the extremes are** — Extremes are not mild. A peak second can carry many times the normal level of the support cycle, which is why averages describe this data badly and why we work with quantiles and episodes instead of means.
- **Where the seconds actually live** — Most seconds are unremarkable: the distribution of the support cycle is concentrated in the low bands with a long thin tail. The market spends the overwhelming majority of its time doing nothing interesting, and the tail is where every episode we care about comes from.
- **Spike, normal, lull — what each looks like** — We label every moment as spike, normal or lull relative to its own baseline, then look at how price behaves inside each state. The states differ from each other in volatility far more clearly than in direction — the same asymmetry that runs through the whole project.
- **The same states laid over price** — Here the labelling is placed directly on a price window so you can see it rather than trust it: where the support cycle flips state, and what price does around those flips.
- **Extremes arrive in clusters** — Spikes in the support cycle are not evenly spaced — they bunch into storms, and a spike inside a cluster is a different animal from an isolated one. Any threshold rule that ignores clustering will fire in bursts and then sit idle for hours.
- **What it forecasts, honestly** — Pulling it together: the two sides of the support cycle move together (coupling reaches 0.90–0.98 at the long end), the quantity forecasts its own future well, and it carries a positive signal about the size of the next move. What it does not carry — on any horizon we measured — is the side.

### The Resistance Cycle: Liquidity Arriving vs Leaving

https://marketflowresear.ch/articles/resistance-cycle-build-up-versus-drain/ · published 31 July 2026 · Resistance cycle

**Summary.** We measured the same sell-side wall growing and thinning second by second across six coins, and asked the only question that matters: does it tell you anything about what happens next? It does — about size, not direction.

- **Which side carries more of the resistance cycle** — Across the whole archive the two sides sit close together: the ratio between build-up into the sell wall and drain out of the sell wall runs 0.36–0.72× depending on the coin.
- **Does it predict the size of the next move** — This is the core question. Rank correlation with the size of the next move is positive on 6 of 6 coins, and strongest at the shortest horizon — +0.39 on BTC at 5s. By the daily scale it fades to +0.07.
- **How much price actually moves per unit** — We measure the price shift per +1σ of the resistance cycle, split into low, middle and high bands. The effect is small in absolute terms — fractions of a basis point per σ — and asymmetric between the two sides. That asymmetry is worth knowing, but it is nowhere near a tradable directional push on its own.
- **What counts as normal, by timeframe** — 'Normal' has no absolute value here: the resistance cycle measured per second is a different quantity from the same thing aggregated over an hour or a day, and the scale grows by orders of magnitude (log axis).
- **Where the link peaks** — Each coin has a window where the resistance cycle lines up best with what follows, marked with a star. The peaks cluster at the short end — the information decays quickly, so any use of this signal has to act on the scale of seconds to minutes rather than hours.
- **Does the imbalance predict direction** — Barely. Taking the signed difference between build-up into the sell wall and drain out of the sell wall and correlating it with the direction of the next move gives at most ρ≈0.22 anywhere in the grid, and it collapses toward zero on most coins and most horizons.
- **How long one-sidedness survives** — A lopsided book does not stay lopsided. The share of imbalance that persists falls from about 51–63% at one second to roughly 16–46% by the twenty-minute mark. Whatever pushes the resistance cycle to one side is undone quickly — which is exactly why it cannot steer longer moves.
- **Memory: does a busy second predict a busy second** — Yes, and this is the most robust property in the whole study. Lag-1 autocorrelation of the resistance cycle is 0.36–0.49 at the fastest scale and 0.29–0.42 at the slowest, on every coin. The quantity is sticky: its own past is a better predictor of its near future than anything about price.
- **How extreme the extremes are** — Extremes are not mild. A peak second can carry many times the normal level of the resistance cycle, which is why averages describe this data badly and why we work with quantiles and episodes instead of means.
- **Where the seconds actually live** — Most seconds are unremarkable: the distribution of the resistance cycle is concentrated in the low bands with a long thin tail. The market spends the overwhelming majority of its time doing nothing interesting, and the tail is where every episode we care about comes from.
- **Spike, normal, lull — what each looks like** — We label every moment as spike, normal or lull relative to its own baseline, then look at how price behaves inside each state. The states differ from each other in volatility far more clearly than in direction — the same asymmetry that runs through the whole project.
- **The same states laid over price** — Here the labelling is placed directly on a price window so you can see it rather than trust it: where the resistance cycle flips state, and what price does around those flips.
- **Extremes arrive in clusters** — Spikes in the resistance cycle are not evenly spaced — they bunch into storms, and a spike inside a cluster is a different animal from an isolated one. Any threshold rule that ignores clustering will fire in bursts and then sit idle for hours.
- **What it forecasts, honestly** — Pulling it together: the two sides of the resistance cycle move together (coupling reaches 0.90–0.99 at the long end), the quantity forecasts its own future well, and it carries a positive signal about the size of the next move. What it does not carry — on any horizon we measured — is the side.

### Liquidity Arriving: What a Growing Wall Says Next

https://marketflowresear.ch/articles/wall-build-up-liquidity-arriving/ · published 31 July 2026 · Inflow

**Summary.** We measured fresh resting liquidity arriving into the book second by second across six coins, and asked the only question that matters: does it tell you anything about what happens next? It does — about size, not direction.

- **Which side carries more of wall build-up** — Across the whole archive the two sides sit close together: the ratio between the sell-wall build-up and the buy-wall build-up runs 0.94–1.03× depending on the coin.
- **Does it predict the size of the next move** — This is the core question. Rank correlation with the size of the next move is positive on 6 of 6 coins, and strongest at the shortest horizon — +0.43 on BTC at 5s. By the daily scale it fades to +0.07.
- **How much price actually moves per unit** — We measure the price shift per +1σ of wall build-up, split into low, middle and high bands. The effect is small in absolute terms — fractions of a basis point per σ — and asymmetric between the two sides. That asymmetry is worth knowing, but it is nowhere near a tradable directional push on its own.
- **What counts as normal, by timeframe** — 'Normal' has no absolute value here: wall build-up measured per second is a different quantity from the same thing aggregated over an hour or a day, and the scale grows by orders of magnitude (log axis).
- **Where the link peaks** — Each coin has a window where wall build-up lines up best with what follows, marked with a star. The peaks cluster at the short end — the information decays quickly, so any use of this signal has to act on the scale of seconds to minutes rather than hours.
- **Does the imbalance predict direction** — Barely. Taking the signed difference between the sell-wall build-up and the buy-wall build-up and correlating it with the direction of the next move gives at most ρ≈0.28 anywhere in the grid, and it collapses toward zero on most coins and most horizons.
- **How long one-sidedness survives** — A lopsided book does not stay lopsided. The share of imbalance that persists falls from about 73–79% at one second to roughly 7–10% by the twenty-minute mark. Whatever pushes wall build-up to one side is undone quickly — which is exactly why it cannot steer longer moves.
- **Memory: does a busy second predict a busy second** — Yes, and this is the most robust property in the whole study. Lag-1 autocorrelation of wall build-up is 0.33–0.50 at the fastest scale and 0.30–0.40 at the slowest, on every coin. The quantity is sticky: its own past is a better predictor of its near future than anything about price.
- **How extreme the extremes are** — Extremes are not mild. A peak second can carry many times the normal level of wall build-up, which is why averages describe this data badly and why we work with quantiles and episodes instead of means.
- **Where the seconds actually live** — Most seconds are unremarkable: the distribution of wall build-up is concentrated in the low bands with a long thin tail. The market spends the overwhelming majority of its time doing nothing interesting, and the tail is where every episode we care about comes from.
- **Spike, normal, lull — what each looks like** — We label every moment as spike, normal or lull relative to its own baseline, then look at how price behaves inside each state. The states differ from each other in volatility far more clearly than in direction — the same asymmetry that runs through the whole project.
- **The same states laid over price** — Here the labelling is placed directly on a price window so you can see it rather than trust it: where wall build-up flips state, and what price does around those flips.
- **Extremes arrive in clusters** — Spikes in wall build-up are not evenly spaced — they bunch into storms, and a spike inside a cluster is a different animal from an isolated one. Any threshold rule that ignores clustering will fire in bursts and then sit idle for hours.
- **What it forecasts, honestly** — Pulling it together: the two sides of wall build-up move together (coupling reaches 0.96–0.99 at the long end), the quantity forecasts its own future well, and it carries a positive signal about the size of the next move. What it does not carry — on any horizon we measured — is the side.

### Liquidity Pulled: What Happens When a Wall Drains

https://marketflowresear.ch/articles/wall-drain-liquidity-pulled-before-the-move/ · published 31 July 2026 · Drain

**Summary.** We measured resting liquidity being pulled away from the book second by second across six coins, and asked the only question that matters: does it tell you anything about what happens next? It does — about size, not direction.

- **Which side carries more of wall drain** — Across the whole archive the two sides sit close together: the ratio between the sell-wall drain and the buy-wall drain runs 0.94–1.05× depending on the coin.
- **Does it predict the size of the next move** — This is the core question. Rank correlation with the size of the next move is positive on 6 of 6 coins, and strongest at the shortest horizon — +0.40 on BTC at 5s. By the daily scale it fades to +0.07.
- **How much price actually moves per unit** — We measure the price shift per +1σ of wall drain, split into low, middle and high bands. The effect is small in absolute terms — fractions of a basis point per σ — and asymmetric between the two sides. That asymmetry is worth knowing, but it is nowhere near a tradable directional push on its own.
- **What counts as normal, by timeframe** — 'Normal' has no absolute value here: wall drain measured per second is a different quantity from the same thing aggregated over an hour or a day, and the scale grows by orders of magnitude (log axis).
- **Where the link peaks** — Each coin has a window where wall drain lines up best with what follows, marked with a star. The peaks cluster at the short end — the information decays quickly, so any use of this signal has to act on the scale of seconds to minutes rather than hours.
- **Does the imbalance predict direction** — Barely. Taking the signed difference between the sell-wall drain and the buy-wall drain and correlating it with the direction of the next move gives at most ρ≈0.17 anywhere in the grid, and it collapses toward zero on most coins and most horizons.
- **How long one-sidedness survives** — A lopsided book does not stay lopsided. The share of imbalance that persists falls from about 65–77% at one second to roughly 9–12% by the twenty-minute mark. Whatever pushes wall drain to one side is undone quickly — which is exactly why it cannot steer longer moves.
- **Memory: does a busy second predict a busy second** — Yes, and this is the most robust property in the whole study. Lag-1 autocorrelation of wall drain is 0.35–0.51 at the fastest scale and 0.28–0.39 at the slowest, on every coin. The quantity is sticky: its own past is a better predictor of its near future than anything about price.
- **How extreme the extremes are** — Extremes are not mild. A peak second can carry many times the normal level of wall drain, which is why averages describe this data badly and why we work with quantiles and episodes instead of means.
- **Where the seconds actually live** — Most seconds are unremarkable: the distribution of wall drain is concentrated in the low bands with a long thin tail. The market spends the overwhelming majority of its time doing nothing interesting, and the tail is where every episode we care about comes from.
- **Spike, normal, lull — what each looks like** — We label every moment as spike, normal or lull relative to its own baseline, then look at how price behaves inside each state. The states differ from each other in volatility far more clearly than in direction — the same asymmetry that runs through the whole project.
- **The same states laid over price** — Here the labelling is placed directly on a price window so you can see it rather than trust it: where wall drain flips state, and what price does around those flips.
- **Extremes arrive in clusters** — Spikes in wall drain are not evenly spaced — they bunch into storms, and a spike inside a cluster is a different animal from an isolated one. Any threshold rule that ignores clustering will fire in bursts and then sit idle for hours.
- **What it forecasts, honestly** — Pulling it together: the two sides of wall drain move together (coupling reaches 0.94–0.98 at the long end), the quantity forecasts its own future well, and it carries a positive signal about the size of the next move. What it does not carry — on any horizon we measured — is the side.

### The Wall That Goes Quiet: What Resting Liquidity Predicts

https://marketflowresear.ch/articles/resting-walls-what-thick-liquidity-predicts/ · published 31 July 2026 · Walls

**Summary.** We measured resting limit liquidity sitting on both sides of the book second by second across six coins, and asked the only question that matters: does it tell you anything about what happens next? It does — about size, not direction.

- **Which side carries more of resting walls** — Across the whole archive the two sides sit close together: the ratio between the resting sell wall and the resting buy wall runs 0.93–1.06× depending on the coin.
- **Does it predict the size of the next move** — This is the core question. Rank correlation with the size of the next move is negative on 4 of 6 coins, and strongest at the shortest horizon — -0.11 on BTC at 5s, with the remaining coins sitting near zero. By the daily scale it fades to -0.08.
- **How much price actually moves per unit** — We measure the price shift per +1σ of resting walls, split into low, middle and high bands. The effect is small in absolute terms — fractions of a basis point per σ — and asymmetric between the two sides. That asymmetry is worth knowing, but it is nowhere near a tradable directional push on its own.
- **What counts as normal, by timeframe** — 'Normal' has no absolute value here: resting walls measured per second is a different quantity from the same thing aggregated over an hour or a day, and the scale grows by orders of magnitude (log axis).
- **Where the link peaks** — Each coin has a window where resting walls lines up best with what follows, marked with a star. The peaks cluster at the short end — the information decays quickly, so any use of this signal has to act on the scale of seconds to minutes rather than hours.
- **Does the imbalance predict direction** — Barely. Taking the signed difference between the resting sell wall and the resting buy wall and correlating it with the direction of the next move gives at most ρ≈0.31 anywhere in the grid, and it collapses toward zero on most coins and most horizons.
- **How long one-sidedness survives** — A lopsided book does not stay lopsided. The share of imbalance that persists falls from about 42–59% at one second to roughly 6–10% by the twenty-minute mark. Whatever pushes resting walls to one side is undone quickly — which is exactly why it cannot steer longer moves.
- **Memory: does a busy second predict a busy second** — Yes, and this is the most robust property in the whole study. Lag-1 autocorrelation of resting walls is 0.74–0.83 at the fastest scale and 0.79–0.90 at the slowest, on every coin. The quantity is sticky: its own past is a better predictor of its near future than anything about price.
- **How extreme the extremes are** — Extremes are not mild. A peak second can carry many times the normal level of resting walls, which is why averages describe this data badly and why we work with quantiles and episodes instead of means.
- **Where the seconds actually live** — Most seconds are unremarkable: the distribution of resting walls is concentrated in the low bands with a long thin tail. The market spends the overwhelming majority of its time doing nothing interesting, and the tail is where every episode we care about comes from.
- **Spike, normal, lull — what each looks like** — We label every moment as spike, normal or lull relative to its own baseline, then look at how price behaves inside each state. The states differ from each other in volatility far more clearly than in direction — the same asymmetry that runs through the whole project.
- **The same states laid over price** — Here the labelling is placed directly on a price window so you can see it rather than trust it: where resting walls flips state, and what price does around those flips.
- **Extremes arrive in clusters** — Spikes in resting walls are not evenly spaced — they bunch into storms, and a spike inside a cluster is a different animal from an isolated one. Any threshold rule that ignores clustering will fire in bursts and then sit idle for hours.
- **What it forecasts, honestly** — Pulling it together: the two sides of resting walls move together (coupling reaches 0.85–0.98 at the long end), the quantity forecasts its own future well, and it carries a negative signal about the size of the next move. What it does not carry — on any horizon we measured — is the side.

### About Market Research Lab — What We Collect and Why

https://marketflowresear.ch/articles/about-market-research-lab/ · published 29 July 2026 · About

**Summary.** Most market commentary is storytelling. We built something testable instead: 166 days of second-by-second order-book data across six coins, and a rule that nothing gets published unless it survives out-of-sample.

- **What we collect** — Every second, around the clock, we record the Binance order book for six pairs — BTC, ETH, SOL, BNB, XRP and DOGE. For each pair we store eight microstructure features plus price: the resting buy and sell walls, how those walls thicken and thin out, and the raw buy/sell volume hitting them.
- **Why we do it** — Most crypto commentary is storytelling: a chart, a narrative, no way to check it. We wanted the opposite — claims small enough to be measured and specific enough to be wrong. Every statement we publish has a number behind it, a sample size, and a test on data the model never saw.
- **How we dig** — First we build baselines: what counts as 'normal' for this coin at this timeframe. Turnover per second is tiny; aggregate to minutes, hours and days and it grows by orders of magnitude, so a 'spike' only means anything relative to its own window — never as an absolute number.
- **What we test** — Three lines of work. Volatility forecasts built from order-book flow. A market-state oscillator that reads the tape's temperature. And trading bots — always walk-forward, always benchmarked against simple buy & hold, priced with a realistic 0.15% fee per action, with take-profit deliberately smaller than stop-loss.
- **What we've learned: size is forecastable** — Order-book flow predicts how big the next move will be. Rank correlation runs around 0.55 in the moment and roughly 0.25–0.30 five minutes ahead — consistently, on all six coins.
- **What we've learned: direction is not** — The same data says almost nothing about which way. Forward directional correlation is about zero by every method we tried — imbalance sweeps top out near ρ≈0.2 on BTC and ETH and collapse to zero on SOL, XRP and DOGE.
- **Our promise** — The moment a strategy shows a positive, validated, out-of-sample result, it gets published here — the method, the numbers and the caveats, including the ones that hurt. Until then we keep digging in the open: each study becomes an article with its charts, and every chart is reproducible from the archive.

### Volume Is the Fuel — Not the Steering Wheel

https://marketflowresear.ch/articles/volume-is-fuel-not-steering-wheel/ · published 25 July 2026 · Volume

**Summary.** We recorded the Binance order book every second for six coins over five months and ran eighteen tests on what volume really does. The through-line: volume tells you how big the next move is — almost never which way.

- **Buy vs sell — which side is bigger** — Over the full history, buying and selling are almost perfectly matched. Sell-volume is marginally larger on five of six coins — ratios from 1.02× (BTC) to 1.06× (XRP) — while SOL is fractionally buy-led (0.98×). Buy-share sits between 48.5% and 50.5%. No coin is structurally dominated by one side.
- **Does volume predict the size of the move** — Yes — clearly. Flow intensity and the size of the next move rise together, strongest on the shortest horizons: rank correlation peaks around 0.22–0.33 within seconds and decays as the window widens.
- **Who moves price more — buying or selling** — We measure the price shift per +1σ of volume. Aggressive buying nudges price up (about +0.10 bp/σ on BTC at the low band) while aggressive selling barely moves it (about −0.01 bp/σ) — a small, asymmetric impact ratio.
- **What counts as normal volume, by timeframe** — 'Normal' depends entirely on the clock. Per-second turnover is tiny; aggregate to minutes, hours and days and it grows by orders of magnitude (log scale). Establishing this baseline per coin per timeframe is what later lets us call a second 'a spike' — always relative to its own window, never an absolute number.
- **At which horizon volume predicts best** — The link is strongest almost immediately. Total flow's correlation with the move peaks at the ~5-second window (ρ≈0.33 on BTC) and fades monotonically toward the daily scale. Imbalance peaks much later (~10 min) and even there is tiny (ρ≈0.008). The usable edge is short-horizon, and it is about magnitude — not side.
- **Does imbalance predict direction** — Only weakly, and unevenly. Imbalance→direction correlation reaches ρ≈0.18 (BTC) and 0.20 (ETH) but collapses to about zero on SOL, XRP and DOGE. The directional bias is under ~1 bp per σ everywhere. So even the side of the flow — not just its size — is a poor compass for where price goes next.
- **How long imbalance lasts before it evens out** — Imbalance is fleeting. The share that persists falls fast as you widen the window — on BTC from ~84% at one second to ~40% at one minute and ~18% by twenty minutes. Buyers and sellers rebalance quickly; a one-sided burst rarely stays one-sided for long. That short lifetime is exactly why it can't steer longer moves.
- **Volume has memory (autocorrelation)** — Volume has real memory, and it grows with scale. Lag-1 autocorrelation of buy-flow on BTC rises from 0.23 (1s) to 0.36 (5s), 0.51 (1 min) and 0.57 (5 min). Busy seconds cluster into busy minutes — activity is self-reinforcing. This persistence is precisely what makes the size signal, volatility, forecastable at all.
- **Each coin's volume fingerprint** — Every coin has its own volume signature across timeframes — BTC and ETH trade in the thousands of coin-equivalents per bucket, XRP and DOGE in the hundreds-to-thousands — with buy and sell curves tracking each other tightly.
- **How big a spike really is** — A 'spike' is the top-1% second (above q99) for its coin and window. These peaks tower over the ordinary second — many multiples of the normal level — and they are exactly the moments the size signal fires. Defining them relative to each coin's own baseline keeps the comparison fair across very different markets.
- **Most seconds are quiet** — Sorting every second into 20 bands (ten below normal, ten above) shows a heavy concentration around and below the normal line, with a thin, long tail of high-volume seconds — the spikes. Buy and sell distributions mirror each other. The market spends most of its time idling, punctuated by rare bursts.
- **Episodes: spike / normal / lull** — Grouping seconds into episodes (spike >q80, normal, lull <q20), the average BTC spike lasts ~60s, moves 0.09% and runs at 0.74 bps/s of volatility — but its net direction is ≈0. Only the extreme-imbalance tails tilt: strongly buy-heavy spikes close +0.065%, strongly sell-heavy −0.069%.
- **Long waves (hours…48h): the macro scale** — Zoom out and something changes. Macro flow-waves last ~1.5h (median) and carry ~1% moves. Here sustained one-sidedness finally aligns with direction: BTC waves dominated by buying close about +0.89%, those dominated by selling about −0.90%.
- **State sequences: which follows which** — States are sticky and they chain. Overlaid on a 96-hour ribbon, spike / normal / lull persist and transition in patterns, and the next episode's volatility depends on the current state.
- **Can we read direction as it forms?** — We split each episode into a forming half and a later half. Early buy-share correlates strongly with the move happening now (ρ≈0.63 on BTC) — you can read the present. But its correlation with the later move is ≈0 (ρ≈−0.08). You can see what is happening; you cannot use it to predict what comes next.
- **A spike at the range edge — reversal or continuation?** — Does a spike near the top or bottom of the day's range signal a reversal or a continuation? Barely either. Position-in-range vs direction correlates ρ≈0.06 on BTC spikes, with buy/sell tilts of a few percent at most. Context helps a little — but a range-edge spike is not a clean directional trade.
- **Spikes come in storms** — Spikes are not evenly spaced — they cluster. Intervals between them have CV≈1.5 (well above 1), a median gap of ~5 minutes, and the vast majority arrive within six hours of the previous one. Interestingly, clustered spikes carry slightly smaller moves (0.089%) than isolated ones (0.103%).
- **The forecast: memory, coupling, volatility** — Pulling it together: buy and sell flow are nearly independent second-to-second (ρ≈0.13) but become tightly coupled at longer windows (ρ≈0.80 by twenty minutes) — a single 'activity' factor. That shared, persistent flow is what our models turn into a forward-volatility forecast.

### Order flow predicts how much, not which way

https://marketflowresear.ch/articles/order-flow-predicts-volatility/ · published 15 July 2026 · Volatility

**Summary.** Across six coins, order-flow surges line up with bigger price moves — but not their direction. A walk-forward look at what the tape can and can't tell you.

---
How to read the site: https://marketflowresear.ch/articles/how-to-read-this-site-in-five-minutes/
