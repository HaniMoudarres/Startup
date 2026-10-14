# Cleared

Original product: [Rocket Money](https://www.rocketmoney.com)

**Pitch:** Before a student spends money, Cleared answers one question: is this specific purchase safe against cash that already reconciles?

Rocket Money shows subscriptions, categories, and net worth after the money has moved. Cleared ignores that dashboard. A student names one purchase. The model writes a clearance: yes or no, the dollars left afterward, and the obligations that decision consumes. A deterministic checker rebuilds that arithmetic from the source ledger. The clearance reaches the student only after the checker passes.

## 1. Solves one problem extremely well

The problem is the purchase in front of you, not "financial wellness."

A student is looking at a $48 concert, a $90 textbook, or a $30 group dinner. The useful answer is whether that charge leaves enough for rent share, a phone bill, and the buffer they already chose. A category chart does not answer it. A subscription cancel button does not answer it. Cleared does only this job.

The input is small on purpose:

- A ledger of transactions the student imported (date, signed amount in cents, merchant, account nickname).
- Recurring obligations the student has confirmed (rent share, phone, transit pass).
- One proposed purchase: amount, date, and a short description.
- A buffer they set, such as "never drop my checking plan below $100."

The output is one clearance object: decision, remaining discretionary cents, the obligations that were counted, and a short explanation that cites ledger ids. Cleared does not negotiate bills, track a credit score, project investments, or coach a whole life plan. Those are separate products. Shipping them would make the clearance worse.

## 2. You know the target customers or have direct access to them

The customer is a college student whose cash arrives in irregular pieces: a campus job, a family transfer, a refund, a club reimbursement. They share costs with roommates. They already feel the decision in group chats ("can everyone do $28 for dinner?"). We are that customer, and we sit next to more of them.

We can reach the first users without an ad budget:

- Our own roommate and project-group chats, where shared expenses already happen.
- Classmates who will export a month of bank CSV and walk through it with us in person.
- A student organization treasurer who already tracks dues and reimbursements in a spreadsheet.

Access is direct enough to watch the real failure. We can sit with five people, import their CSV, and see whether a clearance matches the purchase they were actually considering that week. If the yes/no is useless, we hear it immediately because the user is someone we can text.

We are not trying to acquire households with mortgages. Rocket Money already serves that broader market. Our first ten users are people whose ledgers we can ask about face to face.

## 3. You are familiar with (or can quickly learn) most of the required infrastructure

The stack is a web app a CS student team can operate:

- Next.js on Vercel for the interface and the API route that runs the loop.
- Supabase for auth and Postgres. The ledger is rows, not a document dump.
- CSV import with a parser such as Papa Parse. Students can export CSV from their bank today.
- Integer cents for every amount, so totals are exact.
- A hosted model API that returns JSON. We already use this pattern for coursework.
- Zod, or an equivalent schema parser, as the first gate on model output.

The pieces we would learn quickly are narrow: mapping a few common CSV headers, writing the reconciliation function, and keeping provider keys in environment variables.

We are leaving out infrastructure we cannot honestly deploy this semester. Production bank linking through Plaid requires a company review. Bill negotiation requires call-center operations. Moving money requires money-transmitter rules. Cleared v1 never holds funds and never links a live bank until a later, optional integration. The student uploads a CSV or types the few upcoming deposits they are willing to stand behind.

The model sees normalized transactions only: date, amount, merchant, and an id. Full account numbers are not stored and are not sent in the prompt.

## 4. Reimagines the problem space to make it significantly better

Rocket Money treats the job as visibility. Link every account, discover every subscription, categorize the past, and offer a human who will cancel or negotiate. That is a monitoring product. The student still has to interpret the chart at the moment of spending.

Cleared treats the job as a clearance against a reconciled plan. The month is a set of figures that must add up before any advice exists:

`confirmed inflows − confirmed obligations − buffer − this purchase = remainder`

"Significantly better" here means the student can act on a single checked decision in under a minute, and a wrong total cannot be shown to them. A beautiful dashboard that is off by $80 is worse than silence. The product is willing to return "we could not clear this" when the ledger and the draft disagree.

The reframe also matches how students actually spend. Income is lumpy. Many "budgets" fail because they assume a salary every two weeks. Cleared counts only inflows the student has confirmed, so a hoped-for paycheck cannot silently fund a yes.

## 5. An LLM or generative model is core to the solution

The model does the work the student came for. It reads the messy ledger and the purchase, decides what is discretionary versus already committed, writes the clearance in plain language, and cites the ledger rows it used. Merchant names are inconsistent ("AMZN Mktp", "SQ *CAMPUS COFFEE"). The model is what turns that into a coherent plan: which rows are groceries, which look like the rent share the student already confirmed, and which explanation a person will trust.

That draft is the product, and it is also untrusted. The model is not a help bot beside a normal budget screen. There is no budget screen. If the model cannot produce a draft that survives the checker, the student gets a failure state and no recommendation.

The model is constrained to a schema: `decision` (`yes` or `no`), `remainingCents`, `citedTransactionIds`, `obligationIds`, and `explanation`. Prose outside that object is discarded.

## 6. Validation guardrail

No clearance reaches the student unless deterministic code accepts it. The checker is ordinary TypeScript, not a second model. The API returns the artifact only when `ok` is true. The client renders that accepted object. Drafts stay in server logs for us and are never included in the response body.

The checker enforces all of the following:

- The payload matches the schema. Amounts are integers. `decision` is only `yes` or `no`.
- Every cited transaction id exists in this user's ledger for this request.
- Every cited amount and date in the explanation matches that row exactly. The explanation is scanned for currency amounts; each one must equal a cited row or a subtotal the checker itself computed.
- `remainingCents` equals the checker's own formula: confirmed inflows, minus confirmed obligations, minus the buffer, minus the purchase on a `yes`. A `no` must show the same formula without pretending the purchase was free.
- The purchase date falls inside the planning window. Cited inflows dated after the purchase cannot fund it.
- A `yes` is allowed only when remainder is greater than or equal to zero and the buffer still holds. The model cannot talk the checker into a yes the arithmetic rejects.
- Obligations marked confirmed must be ones the student confirmed. The model cannot invent a $0 rent to make a purchase look safe.

Worked check: the ledger has a $300 paycheck (`txn_19`), rent share of $700, phone $40, groceries plan $160, buffer $100, and a $48 concert. The checker computes remaining discretionary room before the concert and the remainder after it. If the model cites `txn_19` but claims the paycheck was $430, or reports a remainder we did not compute, the draft is rejected. The student never sees that draft.

## 7. Self-correction loop

The loop runs on the server, automatically, before the response is sent.

1. The model receives the ledger, the confirmed obligations, the purchase, and the schema.
2. The checker runs. On success, the accepted clearance is stored with the checker version and returned.
3. On failure, the server sends the model a new request containing the original inputs, its previous draft, and a structured error list. Example: `{ "code": "REMAINDER_MISMATCH", "path": "remainingCents", "expected": 29200, "actual": 40000 }` or `{ "code": "UNKNOWN_TRANSACTION", "path": "citedTransactionIds[1]", "id": "txn_88" }`.
4. The model must return a full new clearance, not a patch. The checker runs on that object from scratch.
5. The loop allows three model attempts. Temperature stays at 0 so retries are repairs, not new personalities.

If the third attempt still fails, the student sees a single unverified state: Cleared could not produce a checked answer for this purchase. They can fix the CSV or confirm an obligation and submit again, which starts a new loop. They are not shown the broken draft and asked to judge it themselves. That would put the validator in the user's head.

The same loop covers explanation drift. If the prose says "you still have about $400" and the accepted remainder is $292, the amount scan fails and the prose has to be regenerated. The decision and the sentence stay tied together.

## 8. Deployability

We can deploy the whole v1 ourselves.

- A Vercel project serves the web app and the route that owns the loop. Users cannot skip the checker by calling the model from the browser. The provider key stays on the server.
- Supabase provides email auth and the tables for users, ledgers, obligations, and accepted clearances.
- CSV upload is a file the student already knows how to export. No bank partnership is required to go live.
- Cost is hosting plus model calls. Three attempts maximum keeps a single clearance bounded. We can cap each account at a modest number of clearances per day.
- A public URL, a seeded demo ledger, and our own real CSVs are enough to run the product end to end.

What "fully deploy" includes for v1 is a student signing in, importing a month, confirming rent and phone, asking about one purchase, and seeing only a checked yes or no. What it excludes is moving money, canceling subscriptions on their behalf, and pulling live balances from every bank in the country. Those features define Rocket Money. They are not required for Cleared to do its job in production.
