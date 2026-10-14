# OnBrief

Original product: [Bounty](https://create.bounty.app/home?category=ugc)

**Pitch:** OnBrief pays student creators for a short video only after the footage proves it hit the brief.

Bounty lets a creator pick a user-generated-content task, follow a script, and wait for someone to approve the upload. OnBrief turns the brief into a contract the video has to satisfy. The model writes the talking points and, after the creator submits, writes an approval packet. Deterministic checks compare that packet to the transcript. The brand and the creator see the packet only after those checks pass. Payment status can change only from a passed packet.

## 1. Solves one problem extremely well

The problem is false approval: a video that looks done, gets paid, and says the wrong thing.

Campus brands and student organizations need a 20–40 second clip with specific claims. "Free with a student email." "Office hours on Thursdays." "Paid partnership" when the clip is an ad. They also need certain claims never to appear. "Guaranteed A." "Official university program" when it is not.

Bounty's hard part is not finding a creator. It is knowing, without a long email thread, whether this file earned the payout. OnBrief does that job. It does not rank creators, run ad auctions, edit the video automatically, or book talent. The creator still films on their phone. We decide, with evidence, whether the cut is acceptable.

The product has two artifacts, both generated and both checked:

- The brief, produced before filming: required claim ids, allowed phrasing, forbidden phrases, required disclosure, target length.
- The packet, produced after upload: for each required claim, either a verbatim quote with a timestamp or an explicit miss; a reshoot script if anything missed; a payout eligibility flag.

A human at the brand can still reject a passed packet for taste. They cannot receive a packet that quotes words the creator never said.

## 2. You know the target customers or have direct access to them

There are two sides, and both live on our campus.

Buyers are student organizations, campus ambassadors, and small businesses near campus that already pay students for short promo videos. They have a real claim to get right and a small budget. They do not have a creator-ops team.

Creators are students who will film a guided clip for a clear base rate. Many of us already do this informally for clubs.

We can put the first loop in front of people we know:

- One organization we belong to, or one we can ask this week, as the buyer.
- Creators recruited from the same org and from classmate group chats.
- A live debrief: watch the video together, read the packet, and mark any quote the checker should have caught.

That access matters because the failure mode is factual. Either the quote is in the video or it is not. We do not need a national creator marketplace to learn whether the packet is trustworthy. Ten videos from people we can text are enough to see missed claims, bad timestamps, and forbidden phrases.

## 3. You are familiar with (or can quickly learn) most of the required infrastructure

We can build and run this as a web product:

- Next.js on Vercel for briefs, upload, and the review screen.
- Supabase Auth, Postgres, and Storage for accounts, briefs, videos, and packets.
- A speech-to-text API (Whisper or Deepgram) to produce the transcript with timestamps. That is a hosted call, not a model we train.
- A hosted LLM API that returns JSON for the brief and the packet.
- Zod for schema checks, plus plain string and timestamp checks we write ourselves.

We already know web CRUD, file upload, and calling an API from a server route. The new work is the transcript contract: store the transcript as text with offsets, and never let the packet invent a quote past those offsets.

We are not building a video-editing suite, a mobile camera app, or an ad network. Creators upload a file they already filmed. Payouts in v1 can be a recorded status ("eligible" / "not eligible") that a treasurer pays manually. Stripe Connect can come later. The core loop does not depend on it.

Brand policy is data we control: a table of required claim ids and a forbidden-phrase list the buyer typed. The model does not get to expand that list in secret.

## 4. Reimagines the problem space to make it significantly better

Bounty reimagined hiring: skip the onboarding call, pick a bounty, follow the script, get paid when a person approves. The approval itself is still a judgment call. Creators learn they were rejected through a delay. Brands learn a claim was missing only if a reviewer was careful.

OnBrief reimagines the brief as something footage can prove. A claim is not "covered" because a paragraph says so. It is covered because a quote is an exact substring of the transcript and the timestamp falls inside the media duration. A reshoot script is not a new creative assignment pulled from nowhere. It may use only allowed claims, it must include every missed claim, and it must exclude every forbidden phrase.

That is a better product for both sides. The creator gets a specific reshoot, tied to claim ids, as soon as the loop finishes. The buyer gets an approval packet they can audit by clicking a quote and hearing that moment. Subjective taste remains human. Factual coverage does not.

The same structure fixes a common UGC failure where the ad disclosure is missing. If the brief marks the clip as sponsored, the packet cannot be eligible unless the disclosure phrase appears in the transcript. The checker looks for the phrase. The model cannot assert that the creator "basically disclosed it."

## 5. An LLM or generative model is core to the solution

The model is the production system, not a writing assistant on the side.

Before filming, it turns a buyer's rough notes ("we need students to know the first session is free and that office hours exist, never promise grades") into a claim-level brief with allowed paraphrases. Buyers do not write structured claim ids by hand, and a static form cannot interpret a messy policy paragraph.

After upload, it reads the transcript and writes the packet: which claims appear to be present, the quote it believes is evidence, what missed, and a reshoot script that tries to repair only the gaps. That judgment over messy speech is the generative step. People say "it's free if you have the .edu email" rather than the canonical sentence in the brief. The model proposes the mapping from spoken words to claim ids.

The mapping is a proposal. It becomes the product only after the checker proves the quote, the timestamps, the forbidden-phrase list, and the payout flag. There is no separate chat where a brand asks "is this video fine?" and pastes the answer onto a payment.

## 6. Validation guardrail

The user-facing artifact is the brief or the packet. Each one is released by server code after a deterministic pass. The route that calls the model does not return the draft. Transcript text in the packet must be copied from the stored transcript, not paraphrased and then trusted.

Brief checks, before a creator is allowed to film against it:

- Schema: claim ids unique, each claim has a canonical sentence, forbidden phrases are a list of strings, max duration is a number of seconds.
- The sponsored flag is boolean. If true, a disclosure sentence is present and non-empty.
- No required claim's canonical sentence contains a forbidden phrase. The buyer sees a brief that is internally consistent.
- Claim ids are stable strings the later packet must reuse. The model cannot renumber them between drafts without failing the schema.

Packet checks, before either side sees it:

- Every `claimId` exists on the brief. Unknown ids fail.
- Every required claim is either `present` or `missing`. Silent omission fails.
- For `present`, `quote` is an exact substring of the transcript, and `startMs` / `endMs` fall inside the media duration with `startMs < endMs`.
- A second scan verifies the quote actually occurs in that timestamp window of the transcript, not elsewhere in the file.
- The reshoot script contains every missing claim's canonical sentence and contains no forbidden phrase, case-insensitive.
- If the brief is sponsored, the transcript contains the disclosure sentence, or `payoutEligible` is false. The checker sets eligibility. The model's boolean is ignored if it disagrees; disagreement is a validation error so the model must come back aligned with the rule.
- `payoutEligible` is true only when every required claim is `present` and disclosure passes.

Worked check: the brief requires claim `C1` ("free for students") and `C2` ("Thursday office hours") and forbids "guaranteed A." The transcript says "It's free if you have a student email, and you can come Thursday to office hours." A packet that quotes "totally free for all students" fails `QUOTE_NOT_IN_TRANSCRIPT`, even if the paraphrase is directionally fair. A packet that quotes "Thursday" with a timestamp past the end of the file fails `TIMESTAMP_OUT_OF_RANGE`. A reshoot script that adds "guaranteed A if you attend" fails `FORBIDDEN_PHRASE`. None of those drafts are shown.

## 7. Self-correction loop

The server runs the loop. The creator does not press "regenerate," and the brand does not repair quotes by hand inside a failed draft.

1. For a brief, the model receives the buyer's notes, the structured policy fields, and the schema. The checker runs. Errors such as `FORBIDDEN_IN_REQUIRED_CLAIM` or `MISSING_DISCLOSURE` go back with the draft.
2. For a packet, the model receives the accepted brief, the full transcript with timestamps, the media duration, and the schema. It never receives a bare "approve this video" instruction with no transcript.
3. A failure returns the draft plus errors, for example `{ "code": "QUOTE_NOT_IN_TRANSCRIPT", "path": "claims[0].quote", "claimId": "C1" }` or `{ "code": "MISSING_CLAIM_IN_RESHOOT", "claimId": "C2" }`.
4. The model returns a complete new brief or packet. The checker evaluates the new object independently. Earlier quotes are not assumed good.
5. Three attempts maximum. A passing packet is stored, linked to the transcript hash and the brief version, then shown. Eligibility is what the treasurer sees.

If all attempts fail, both sides see an unverified hold: OnBrief could not certify this cut. The video remains uploaded. No payout flag flips to eligible. The creator can submit a new file, which starts a new loop against the same brief. Holding the video is the safe outcome. Showing a confident, unquoted approval is not.

The retry is useful here because the repair is local. The model often has the right claim and the wrong slice of transcript. Feeding the exact error lets it select a substring that is actually present instead of writing a cleaner sentence than the creator said.

## 8. Deployability

The full loop can ship on accounts we control.

- Vercel hosts the app. Model and transcription keys stay in server environment variables.
- Supabase stores users, briefs, transcripts, and video objects behind auth. A creator sees their uploads. A buyer sees their briefs and the packets that passed.
- Transcription and LLM calls are pay-per-use. We cap video length (for example 60 seconds) and retries (three) so one submission cannot run away in cost.
- v1 payouts are an eligibility record a human treasurer can pay. That is deployable without becoming a payment processor. The record is written only after `ok === true`.
- We can run a pilot with one organization and a handful of creators on a public URL in the same week the app is up.

We are able to deploy this because the hard part is a checked document about a short file, not a global creator economy. Camera apps, in-app editing, marketplace rankings, and automated ad buying are out of scope. The deployed product lets a buyer publish a checked brief, lets a creator upload a clip, and lets both of them read a packet whose quotes are real.
