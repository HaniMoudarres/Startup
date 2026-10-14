# Curtain

Original product: [Letterboxd](https://letterboxd.com/apps/)

**Pitch:** Curtain names the one movie to start tonight that fits the minutes you have and a service you already pay for.

Letterboxd is a social diary: log a film, rate it, write a review, follow friends, keep a watchlist. Curtain uses a diary only as evidence. The product is a single tonight pick. The model proposes it from your history and your constraints. Deterministic checks confirm the film exists, fits the runtime, is on a service you selected, and that every past film the blurb mentions is one you actually logged. You see the pick only after those checks pass.

## 1. Solves one problem extremely well

The problem is the ten minutes before you either start a movie or pick up your phone and lose the evening.

You have a runtime window ("I have 95 minutes"), the services you subscribe to, and a history of what you finish and what you abandon. You need one title, with a backup that survives the same checks, and a reason tied to your own log. You do not need a new social network, a review feed, or a universal ranking of cinema.

Curtain's output is fixed: `primary` and `backup`, each with a catalog id, title, year, runtime, provider, and a short reason. The reason may cite a small number of your past logs. There is no endless row of "you might also like." Choice paralysis is the failure we are removing, so the screen does not reintroduce it.

We do not clone profiles, comments, lists, or festivals. Letterboxd remains a better diary. Curtain reads a diary (yours, imported or kept lightly in-app) and spends it on tonight's decision.

## 2. You know the target customers or have direct access to them

The customer is a student who watches movies in short, irregular windows and already thinks in terms of Letterboxd: ratings, rewatches, "I never finished that." We are in that group, and our friends are too.

We can reach the first users directly:

- Friends who have Letterboxd accounts and will export or re-enter a few dozen logs.
- Roommates deciding what to watch on a weeknight, whom we can ask the next morning what they actually started.
- A campus film club whose members already argue about selections and will tell us when a pick ignored the runtime.

Access includes the follow-up, which is the real test. The morning after, we can ask "did you start it, and did the reason mention movies you have actually seen?" If the pick was on a service they do not have, we will hear about it because the user lives with us or sits in class with us. We are not buying a national audience of cinephiles. The first version is for people whose watch habits we can ask about in person.

## 3. You are familiar with (or can quickly learn) most of the required infrastructure

We can run this on familiar pieces:

- Next.js on Vercel and Supabase for auth, logs, service selections, and accepted picks.
- The TMDB API for identity, runtime, year, and regional watch providers. Free API keys and documented REST are enough. We do not build a movie database.
- A hosted LLM API that returns two catalog ids and a blurb, as JSON.
- Zod and our own constraint checks against rows we stored from TMDB at request time.
- A simple importer: CSV or a manual "I have seen this" search against TMDB, so we are not blocked on a Letterboxd partnership.

We know how to call a third-party API, cache a response, and store user rows. The part we will learn is TMDB's provider endpoint: request the user's country, store the provider ids returned for that title, and compare them to the services the user toggled. That is documented integration work, not a new platform.

We will not run recommendation training jobs, host video, or scrape streaming sites. If TMDB has no provider data for a title, that title is ineligible unless the user has explicitly marked "I can watch this without a listed service" on their account, and the pick is then labeled with that override. The checker reads the flag from our database. The model cannot invent availability.

## 4. Reimagines the problem space to make it significantly better

Letterboxd made film memory social. The diary, the follow graph, and the lists are the product. Discovery is a side effect of other people's taste and of what is popular this week. The weeknight question is harsher than popularity: the movie must fit the time left, be playable on a service already paid for, and resemble films this person finishes rather than films they bookmark.

Curtain treats tonight as a constraint problem with a generated explanation. The catalog and the user's log are the facts. The model proposes a title and a reason. The checker throws away proposals that violate the facts. A recommendation that is "right" in taste and wrong on runtime is a failed product, because the user will not finish it. A reason that cites a beloved film they never logged is also a failure, because it is flattering and false.

The social graph is optional input, not the experience. A user may mark a few friends whose high ratings they care about. Those ratings enter as facts the blurb may cite only if the friend id and the film id exist in our tables. We are not building a feed to scroll instead of watching.

One primary and one backup is the whole decision. Both must pass. If only one title in the draft passes, the draft fails and the loop tries again, so we never pad the screen with an unchecked alternate.

## 5. An LLM or generative model is core to the solution

The model is what turns a diary and a clock into a human decision. It reads ratings, abandons, genres the user consistently finishes, the runtime window, and the services, then chooses. It writes the reason: why this title is the one to start, pointing at specific earlier logs ("you rated these three under-100-minute character pieces highly and left a 165-minute film unfinished").

A sort by average rating does not know that the user abandons long movies on weeknights. A chat box beside Letterboxd would answer once, unverified, and would still leave the user with a paragraph of options. Curtain's model is required to commit to two catalog ids. The commitment is the product. The checker decides whether the commitment is showable.

If the model cannot place two titles inside the constraints, the user gets an unverified state and a narrower question ("add a service" or "allow 130 minutes"), not a list of near misses the model wished were available.

## 6. Validation guardrail

The checker is server-side and deterministic. It uses TMDB payloads and database rows we fetched for this request, saved alongside the decision. The model does not get to supply runtime or provider as facts. It supplies ids and prose. We look up the ids.

Checks on both `primary` and `backup`:

- The id exists in the TMDB response we stored. Title and year on the artifact match that record. A renamed or mis-yeared film fails.
- Runtime is the TMDB runtime, and it is less than or equal to the user's window. Missing runtime fails closed.
- The user has not logged the film as watched, unless they turned on rewatches for this request.
- At least one TMDB watch-provider id for the user's country is in the user's selected services, or the account-level override is on and the artifact labels `availability: "user-override"`. A Hulu-only title fails when the user selected Netflix and Max.
- `primary` and `backup` are different ids.
- The genre or rating filters the user set as hard (for example, exclude horror) match TMDB genres. The model cannot slip an excluded genre in because the blurb is persuasive.

Checks on the reason:

- Every film title the reason puts in a citation field exists in the user's log, and any rating it states equals the stored rating. We require citations as structured ids, not as free text we hope is honest. Prose may mention a film only through those ids; the rendered sentence is filled from our titles, so the model cannot sneak a third film into the paragraph.
- Friend ratings, if cited, match a stored friend log for that film.
- The blurb's length stays inside a fixed bound so the page remains a decision, not an essay.

The API returns the pick only when both titles and the reason pass. Drafts are not rendered.

Worked check: the user has 95 minutes, Netflix and Max, and a log that includes three short dramas rated above 4, plus an abandoned 165-minute film. The model picks a 140-minute film. `RUNTIME_EXCEEDS_WINDOW` rejects it. Another draft cites a love for *Portrait of a Lady on Fire* with a log id that is not in the diary. `CITED_FILM_NOT_IN_DIARY` rejects it. Another puts a film on Hulu in `primary` while Hulu is unselected. `PROVIDER_MISMATCH` rejects it. The user sees none of these. They see two titles that fit, or a hold.

## 7. Self-correction loop

The server runs the loop before the page updates.

1. We fetch candidate facts first: the user's logs, services, window, and a TMDB search constrained as far as the API allows (runtime and providers filtered into a candidate id list we build). The model may choose only from that candidate list. Giving it the open web of all cinema makes retries slower and failures noisier.
2. The model returns two ids, structured citation ids, and blurb fields.
3. The checker compares against TMDB and the log. Success stores the pick and the TMDB payload hash, then returns it.
4. Failure sends the model the candidate list again, its draft, and errors such as `{ "code": "RUNTIME_EXCEEDS_WINDOW", "id": 123, "runtime": 140, "window": 95 }`, `{ "code": "PROVIDER_MISMATCH", "id": 456, "providers": ["Hulu"] }`, or `{ "code": "CITED_FILM_NOT_IN_DIARY", "id": 789 }`.
5. The model returns a new pair from the candidate list and a new reason. Three attempts. Each attempt is fully rechecked. A good primary cannot carry a bad backup.

If the third attempt fails, the user sees that Curtain could not certify a pick under these constraints. The form keeps their window and services. They can widen the window or add a service they forgot. That is a new request and a new loop. We do not show the rejected 140-minute film as a "soft" suggestion. The guardrail would be decorative if the fallback were an unchecked list.

Automatic retry matters because the model's taste can be right while its constraint following is sloppy. The error tells it the runtime and the provider it missed. The next draft is how the product improves itself without the user becoming the checker.

## 8. Deployability

Curtain is deployable by this team as a public web app.

- Vercel hosts the interface and the server loop. TMDB and model keys stay in the environment.
- Supabase holds users, logs, service toggles, and accepted picks.
- TMDB's free tier covers a student pilot. We cache title and provider lookups so a retry does not multiply catalog calls.
- Model cost is bounded: a candidate list already filtered, two ids out, three attempts.
- Import is manual search or CSV, so we can go live without Letterboxd's permission. Users who love Letterboxd can keep it. Curtain needs the log, not the social graph.
- We can deploy a URL, seed our own diaries, and run the tonight flow the same day.

A complete deployment is a signed-in user setting 95 minutes and two services, receiving two playable titles with reasons that match their log, or receiving a clear hold. We do not need native apps, a review community, or streaming licenses. Letterboxd's apps remain the diary. Curtain is the checked decision in front of the blank screen.
