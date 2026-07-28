---
layout: post
title: "The most dangerous person in my backtest is me"
date: 2026-07-28
---

*Methodology only: no strategy identity, no signals, no parameters, no performance numbers. Those stay private, which is rather the point of the post.*

---

My first backtest engine had a bug that let it trade for free, and it paid nothing on idle cash. It found some very exciting strategies. Everything I've built since exists because of what that felt like.

Market Heuristics is a research pipeline over Indian equities. It ends in a small app that tells me what to do once a month, and I've spent far more of it on not fooling myself than on anything resembling alpha. Three parts are worth writing up: measuring survivorship bias instead of gesturing at it, treating corporate-action data as hostile, and sealing the evaluation so that future-me can't quietly move the goalposts.

## Measure the bias, don't assert it

Most retail backtests run on today's index members, which means every historical portfolio quietly excludes the companies that died. Everyone knows this is bad. Fewer people check how bad.

The fix first: reconstruct membership point-in-time by starting from the current constituent set and walking the dated replacement history backwards, undoing each change. Resolve renames and mergers before any set arithmetic. The build enforces its invariants and refuses to emit output if any fail: exactly the right member count at every month-end across 17 years, no duplicates, every name mappable to a symbol, plus twenty hard-coded spot checks against known index events.

Then the measurement: run the same frozen slate of strategies, same engine, twice. Once on as-of-date membership, once on recent membership applied retroactively. The per-strategy difference is the survivorship bias, measured. Sanity check that the reconstruction is honest: buy-and-hold on the rebuilt universe lands within dividend noise of the investable index, closing a gap of several percentage points a year.

The finding I can share abstractly: the bias is not a constant offset you can subtract. It varies several-fold across strategy families, and it's largest for exactly the families that concentrate in names heading for trouble. If your backtest universe is survivor-only, some strategy types are lying to you much harder than others.

There's also a second universe, built with no index committee at all: the exchange's own daily settlement files, which list delisted names because they traded that day, with membership defined by trailing liquidity rank. Survivorship-free by construction. Six companies a free data vendor had simply erased were recovered from that archive, verified at 0.997+ daily-return correlation on overlaps. One gap remains (dividends missing on those recovered names) and it's declared in the docs rather than buried. Where a fundamentals source is survivor-biased and restated, the docs state the consequence plainly: those biases flatter results, so a null is a robust kill and a positive is only ever a candidate.

## Corporate actions: assume every source lies sometimes

The integrity layer cross-examines sources instead of trusting any of them. Vendor daily returns get correlated against the exchange archive's own within-row return, which is corporate-action-free ground truth. When they diverge, the direction decides the severity. A raw price cliff the vendor smoothed means the vendor correctly adjusted a real split: benign. A cliff in the vendor data that the exchange never saw means the vendor fabricated a gap: hard failure, refresh gates shut.

Correlation has a blind spot, though: when both sources agree on a real as-traded gap, like a demerger, nothing diverges. So a second scan looks for large single-day moves on calm index days. That one is review-only, never auto-quarantine, because a large single-day move is also what a genuine crash looks like. There's a pharma company in my data whose real regulatory disasters are indistinguishable, numerically, from an unadjusted split. A heuristic that can't tell those apart doesn't get to make decisions alone.

The adjustment heuristic itself is tested against 93 names with known-correct adjusted series. It has to recover their real events without inventing events on crash days, and the build refuses to proceed if it can't. And one QC window got widened after a real miss: a mis-dated split sat inside the signal's lookback but outside the check's window. The failure and the fix are both documented in the code, next to each other, which is where that kind of thing belongs.

## Seal the evaluation before you see the results

Here's the uncomfortable part. After the data is clean, the biggest remaining threat is the researcher. I will, given the chance, re-run things until they look good and remember it as diligence.

So the chance gets removed structurally. Strategy slates, windows, and decision rules are declared before results are seen; that's been the practice across four research generations. The production configuration shipped with a dated pre-registration written before the first live trade, and its review criteria are sealed: they may not be modified, reinterpreted, or postponed afterward. Two dated checkpoints, one standing tripwire, all written as mechanical conditions with nothing left to judgment. The measurement spec is written in advance too, including what the strategy has to beat: a 100-seed Monte Carlo of random selections sharing the same universe, calendar, modeled costs, and regime state, so the comparison isolates selection skill and nothing else. A mediocre first year is pre-committed to be ignored, in writing, because expected noise shouldn't get to cause tinkering. Demotion is one-way. A demoted configuration doesn't come back on the strength of the same backtest.

The app enforces this where willpower would fail. There is no strategy selector and no settings panel: a live switch between alternatives is the forking-paths machine, so it doesn't exist. There is no monthly scoreboard for the experimental sleeve; performance surfaces at the sealed checkpoint dates and not before, because watching a one-regime discovery wiggle is how tinkering starts. Deposits mid-cycle are sizing events against frozen picks, never re-ranking events, enforced as separate code paths.

The pre-registration also keeps an honest ledger of its own evidence: each component labeled by validation status, including one adopted at face value as an in-sample discovery, with the winner's curse named as such. It records where the pipeline's recommendation and my decision differed. Writing down your own disagreement with your own tooling is uncomfortable. It's also the cheapest audit trail I know.

## The app at the end

All of it feeds a local FastAPI-and-SQLite app that produces a monthly briefing. It never trades; I execute at the broker myself. Its source of truth is an append-only journal, with positions, cash, and cost basis derived by aggregation and never stored. Corrections flip a void flag and everything downstream recomputes. The app imports the same selection library the backtests use, and a parity test holds them together: identical eligibility, scores, and picks at historical decision dates, exit non-zero on any mismatch.

For calibration: ~4,300 daily exchange files parsed into a 6.7-million-row table, 2009 to now, about 2,400 symbols in the latest session, eight research generations each of which exists because the previous one failed at something specific. No CI, no packaging; it's a single-operator system and the rigor budget went to the data and the protocol. The strategy it runs is live, and no, this post doesn't say what it is.

---

*Stack: Python, pandas, numpy, scipy, scikit-learn, FastAPI, Jinja2, SQLite with no ORM, plain files for the research path, exit-code test gates.*
