# Posthand

Original product: [Gambit](https://www.gambit.com)

**Pitch:** Posthand reviews one decision in a hand you already finished, and the explanation is allowed to ship only if it agrees with the chart and the calculator.

Gambit, from the Chess.com family, teaches Texas Hold'em without real-money gambling: lessons, bots, tournaments, a rating, and a coach that reviews hands. Posthand takes only the review. You enter one decision point from a completed hand. The model writes a short audit of that decision. A rules engine checks the cards, the legal actions, the preflop chart, and any equity figure. You see the audit only after it matches.

## 1. Solves one problem extremely well

The problem is a single decision you already made: call, fold, bet, or raise at one moment, and whether that action holds up.

Beginners do not lose because they lack a poker website. They lose the lesson because a review says something confident and slightly wrong. "You should have called" when the chart says fold, or "you're a favorite" with an equity number that was invented. Posthand answers one decision per submission. It names the action the chart or the calculator supports, says whether the action you took is in that set, and explains the difference in a few sentences.

It does not offer a full school, a public rating, daily tournaments, or a table to play at. Gambit already built that arcade. Playing out a hand inside our app would also create pressure to advise the user while the hand is still live. Posthand accepts hands only after they are over: a manual entry or a pasted history. The review cannot change cards that are still coming.

One decision, fully checked, is the product. If the user wants another street reviewed, that is a new submission with its own loop.

## 2. You know the target customers or have direct access to them

The customer is a beginner who plays socially or on a free learning site and wants to know whether a specific spot was a mistake. That includes us and people we already play with or watch play.

We can reach them directly:

- Friends who play kitchen-table or free-chip games and will recount a hand from last night.
- Classmates who have tried Gambit or watched poker content and still cannot tell if a call was disciplined or curious.
- A standing offer in our group chats: send one hand, get one checked decision back, tell us if the explanation matched the hand you remember.

We know this user because the hand history they care about is recent and personal. They can tell us immediately if we reviewed the wrong street or mis-heard the bet size. We do not need to acquire the global poker audience. The first reviews are hands we can re-deal on a table and compare to the audit.

We are explicit about who we will not serve in v1: people looking for real-time help during an online cash game. The form asks for a completed hand, and the copy frames the result as a retrospective lesson.

## 3. You are familiar with (or can quickly learn) most of the required infrastructure

Most of the system is ordinary web infrastructure plus one library we can learn from its docs:

- Next.js on Vercel and Supabase for accounts and saved reviews.
- A hand evaluator and equity library such as `pokerkit` or `treys`, called from a server function. We do not train a poker model.
- A static preflop chart we store as JSON, keyed by position, stack depth bucket, and the action faced. We can encode a small beginner chart ourselves and cite its version on every review.
- A hosted LLM API for the explanation, forced to a JSON schema.
- Zod, plus our own legal-action and card-parsing checks.

We already know how to build a form, store a record, and call an API. The poker-specific learning is bounded: represent a deck without duplicate cards, enumerate legal actions given the current bet, look up one chart cell, and run one equity calculation against a stated range. That is a weekend of reading, not a solver lab.

We are not operating a gambling service. No deposits, no rake, no chip store. We are not running multiplayer infrastructure or a mobile client. Users type the hand. That keeps the infrastructure inside what we can deploy and keeps the product on the learning side of the line Gambit drew.

## 4. Reimagines the problem space to make it significantly better

Gambit asked how poker could feel like Chess.com: lessons, bots, ratings, and a coach beside a place to play. That makes practice available. It still leaves the coach as a voice the student has to trust.

Posthand asks a stricter question. Given this exact decision, what does a published rule say, and does the write-up reproduce that rule? The explanation is not allowed to outvote the chart or the arithmetic. If the chart's allowed set is `{fold}` and the model prefers a call, the call does not ship. If the model says the hand has 62% equity and the calculator returns 48% against the named range, the sentence does not ship.

The reframe is from "a place to learn poker" to "an audit of one action." Beginners get better when a single leak is named correctly. They get worse when a fluent review invents a number. Silence after three failed checks is a better product than a smooth paragraph that teaches the wrong fold.

The chart is versioned and visible. Two users who submit the same spot get the same allowed action set, because that set comes from the table, not from the model's mood. The model still earns its place by explaining the spot in language tied to those fixed results.

## 5. An LLM or generative model is core to the solution

The model writes the audit. It identifies which decision the user means when the history is told in messy prose ("I was on the button, he opened, I had king-queen suited, I three-bet, was that dumb?"), maps that onto structured seats and actions, and explains the checked result in words a beginner can use at the next home game.

A chart lookup alone does not do that. Users do not speak in chart coordinates. A calculator alone returns a float, not a lesson. The generative step is the translation both ways: prose hand into a structured spot, and structured verdict back into a short explanation that mentions the real cards and the real bet.

The model fills a schema: `street`, `heroCards`, `board`, `position`, `actionTaken`, `actionRecommended`, `equityCited` (optional), `rangeDescribed`, and `explanation`. The library and the chart decide whether `actionRecommended` and `equityCited` are acceptable. The model does not get a side panel. If its audit cannot be reconciled, the page shows an unverified state and no strategy tip.

## 6. Validation guardrail

The checker is deterministic code around the library and the chart. It runs on the server. The browser never receives a draft. A review is stored and rendered only when every check passes.

Checks:

- Schema and card grammar. Cards are a closed rank-and-suit format. No card repeats across hole cards, board, and any mentioned dead cards.
- The street and the board agree. Preflop has zero board cards. The flop has three. The turn has four. The river has five.
- The action taken is legal in the spot. A check is illegal facing a bet. A raise size, if given, is at least a minimum raise we compute from the pot and the current bet.
- Every card named in the explanation appears in the structured hand. Inventing an ace on a king-high board fails.
- If the spot is preflop and covered by the chart, `actionRecommended` is inside the chart's allowed set for that position, stack bucket, and facing action. The user's `actionTaken` may fall outside that set; the review must say so, and the recommended action must not.
- If `equityCited` is present, it is within a small tolerance of the library result for `heroCards` against `rangeDescribed` on the given board. The range must be one the library can parse. A vague "a tight range" without a parseable range cannot carry a numeric equity.
- Pot odds, if stated as a ratio or a required equity, match the bet and the pot we compute from the action history. We do not accept a restated percentage that the model found convenient.
- Stack and pot sizes are non-negative integers in big blinds or chips, and they match the inputs. The model cannot quietly change the bet from 3bb to 6bb.

Worked check: the button holds KQs at 100 big blinds, facing an under-the-gun open, and three-bets. The chart's allowed set includes the three-bet. The model recommends fold. The checker returns `ACTION_NOT_IN_CHART` and withholds the review. In another submit, the model says the hand is 62% against a parseable calling range and the library returns 48%. That is `EQUITY_MISMATCH`. In a third, the explanation mentions the ace of hearts on a board of Kd7c2s. That is `CARD_NOT_IN_HAND`. The user sees none of these drafts.

## 7. Self-correction loop

The server loops without a user-facing retry button.

1. The model receives the raw hand notes, the structured fields it must fill, the chart version, and the schema. It proposes the audit.
2. The checker runs the library and the chart. Success stores the review with the chart version, the library version, and the inputs, then returns it.
3. Failure sends the model its draft and errors such as `{ "code": "ACTION_NOT_IN_CHART", "allowed": ["call", "raise"], "actual": "fold" }`, `{ "code": "EQUITY_MISMATCH", "expected": 0.48, "actual": 0.62 }`, or `{ "code": "CARD_NOT_IN_HAND", "card": "Ah" }`.
4. The model returns a new full audit. Recommendations move to the allowed set. Cited equity is copied from the expected value the error already computed, or omitted. Phantom cards leave the explanation.
5. Three attempts. Each one is checked from scratch, so a fixed equity cannot hide an illegal action that remains.

If the third attempt fails, the user sees that Posthand could not certify a review of this decision. The hand they typed remains on the form so they can correct a typo, such as a duplicated card. They do not see speculative strategy. Automatic feedback is what makes the model usable here: the first draft is often fluent and wrong by one action, and the error payload says the exact repair.

We keep the loop retrospective. Nothing in the retry path is wired to a live table or to screen capture of an in-progress game.

## 8. Deployability

We can deploy Posthand as a public web app.

- Vercel serves the form and the server route that owns the model, the chart, and the library. The provider key is not in the client.
- Supabase stores accounts and accepted reviews. A demo mode can run without an account for one sample hand.
- The chart JSON and the evaluator run in our process. We do not depend on a poker operator, a gaming license, or an app store.
- Model spend is capped by one decision per submission and three attempts. Equity runs are a single library call per attempt, which is cheap.
- We can ship with a beginner preflop chart and flop pot-odds checks, then add turn and river chart rows later without changing the guardrail. v1 is already a complete product for the decision it claims to review.

Fully deployed means a beginner opens the URL, enters last night's button three-bet, and reads an explanation whose action and numbers survived the checker. It does not mean we operate a poker room. Gambit's playable site can remain where people practice. Posthand is the checked debrief afterward.
