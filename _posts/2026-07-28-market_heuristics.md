---
layout: post
title: "The most dangerous person in my backtest is me"
date: 2026-07-28
description: "A survivorship-free equity research pipeline, and the discipline of not fooling yourself."
---

*Methodology only: no strategy identity, no signals, no parameters, no performance numbers. Those stay private, which is rather the point of the post.*

My first backtest engine had a bug that let it trade for free, and it paid nothing on idle cash. It found some very exciting strategies. Everything I've built since exists because of what that felt like.

Market Heuristics is a research pipeline over Indian equities. It ends in a small app that tells me what to do once a month. Most of the work went into three things: measuring survivorship bias instead of gesturing at it, treating price data as hostile, and sealing the evaluation so future-me can't move the goalposts.

## Measure the bias, don't assert it

Most retail backtests run on today's index members. Every historical portfolio in them quietly excludes the companies that died. Everyone knows this is bad. Fewer people check how bad.

The fix: reconstruct membership point-in-time. Start from the current constituent set, walk the dated replacement history backwards, undo each change. Resolve renames and mergers before any set arithmetic.

The build refuses to emit output unless its invariants hold: exactly the right member count at every month-end across 17 years, no duplicates, every name mapped to a traded symbol. Twenty hard-coded spot checks against known index events on top.

Then the measurement. Run the same frozen slate of strategies, same engine, twice: once on as-of-date membership, once on recent membership applied retroactively. The difference, per strategy, is the survivorship bias. Sanity check: buy-and-hold on the rebuilt universe lands within dividend noise of the investable index.

The finding I can share abstractly: the bias is not a constant you can subtract. It varies several-fold across strategy families, and it's worst for exactly the families that concentrate in names heading for trouble. If your universe is survivor-only, some strategies are lying to you much harder than others.

There's a second universe with no index committee at all: the exchange's own daily settlement files, which list delisted names because they traded that day. Membership is trailing liquidity rank. Survivorship-free by construction. Six companies a free data vendor had simply erased were recovered from that archive, verified at 0.997+ return correlation on overlaps. One gap remains (dividends on those six) and the docs declare it instead of burying it.

## Assume every source lies sometimes

Vendor daily returns get cross-examined against the exchange archive's own within-row return, which is corporate-action-free ground truth. When they diverge, the direction decides severity.

A raw price cliff the vendor smoothed? The vendor correctly adjusted a real split. Benign. A cliff in the vendor data the exchange never saw? Fabricated gap. Hard failure, refresh gates shut.

Correlation has a blind spot: when both sources agree on a real as-traded gap, like a demerger, nothing diverges. So a second scan looks for large single-day moves on calm index days. That one is review-only, never auto-quarantine, because a large single-day move is also what a genuine crash looks like. There's a pharma company in my data whose real regulatory disasters are numerically indistinguishable from an unadjusted split. A heuristic that can't tell those apart doesn't get to act alone.

The adjustment heuristic itself is tested against 93 names with known-correct series. It has to recover their real events without inventing events on crash days, or the build stops.

And one QC window got widened after a real miss: a mis-dated split sat inside the signal's lookback but outside the check's window. The failure and the fix are documented next to each other in the code, which is where that kind of thing belongs.

## Seal the evaluation before you see the results

Here's the uncomfortable part. Once the data is clean, the biggest remaining threat is the researcher. Given the chance, I will re-run things until they look good and remember it as diligence.

So the chance gets removed structurally.

Slates, windows, and decision rules are declared before results are seen. The production configuration shipped with a dated pre-registration written before the first live trade. Its review criteria are sealed: two dated checkpoints, one standing tripwire, all mechanical conditions, none of it open to reinterpretation afterward.

What the strategy has to beat is also fixed in advance: a 100-seed Monte Carlo of random selections sharing the same universe, calendar, modeled costs, and regime state. The comparison isolates selection skill and nothing else. A mediocre first year is pre-committed to be ignored, in writing, because expected noise shouldn't get to cause tinkering. Demotion is one-way. A demoted configuration doesn't come back on the strength of the same backtest.

The app enforces this where willpower would fail. No strategy selector, no settings panel: a live switch between alternatives is the forking-paths machine, so it doesn't exist. No monthly scoreboard for the experimental sleeve either; performance surfaces at the sealed checkpoints and not before. Deposits mid-cycle are sizing events against frozen picks, never re-ranking events. Separate code paths.

The pre-registration also keeps an honest ledger of its own evidence. Each component is labeled by validation status, including one adopted at face value as an in-sample discovery, winner's curse named as such. It records where the pipeline's recommendation and my decision differed. Writing down your disagreement with your own tooling is uncomfortable. It's also the cheapest audit trail I know.

## The app at the end

A local FastAPI-and-SQLite app produces a monthly briefing. It never trades; I execute at the broker myself. Its source of truth is an append-only journal. Positions, cash, and cost basis are derived by aggregation, never stored. Corrections flip a void flag and everything recomputes.

The app imports the same selection library the backtests use, held together by a parity test: identical eligibility, scores, and picks at historical decision dates, exit non-zero on any mismatch.

For calibration: ~4,300 daily exchange files parsed into a 6.7-million-row table, 2009 to now, eight research generations, each one built because the previous one failed at something specific. No CI, no packaging. It's a single-operator system, and the rigor budget went to the data and the protocol.

The strategy it runs is live. And no, this post doesn't say what it is.

---

*Stack: Python, pandas, numpy, scipy, scikit-learn, FastAPI, Jinja2, SQLite with no ORM, plain files, exit-code test gates.*
