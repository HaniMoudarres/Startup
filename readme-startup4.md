# Dialectic.ai

Original product: [Debate Arena](https://www.debatearena.app)

**Pitch:** Dialectic.ai sits in a live conversation, checks the claims both people actually made, and shows both of them the same sourced record of what held up.

Debate Arena is built for practice rounds: an AI opponent, timed formats, and a judge for competitive debate. Dialectic.ai is built for the argument you are already in — a dorm disagreement, a dinner-table fight, a club meeting that turned personal. Two people talk. Speech-to-text keeps a speaker-labeled transcript. The model turns that transcript into a live moderation record: the claims each person made, a fact-check against retrieved sources, and, in the deeper mode, the argumentation style and any logical fallacies. A studying mode, used before or after the conversation, builds a sourced prep sheet through a chat that can cite only documents the retriever returned. Nothing the model writes reaches either person until a validator accepts it.

The product is for social debates first. The same loop still works on a prepared round.

## 1. Solves one problem extremely well

The problem is an argument that ends with no shared picture of what was claimed or what was true.

People leave those conversations angry and no better informed. One side remembers a statistic that was never checked. The other remembers an insult. Nobody can point to a source. Dialectic.ai does one job: give both people a live, sourced record of the claims in *this* conversation. The record says who said the sentence, whether a retrieved source supports it, contradicts it, or leaves it unverified, and — when they ask for a result — which checked claims the consensus rests on.

Three modes are three depths of that same record, not three products:

- **Basic arbitration** listens, fact-checks evidence, and states a consensus from the claims that survived checking.
- **In-depth moderation** adds argumentation style and logical fallacies, each one tied to a quote from the transcript.
- **Studying** is the same standard before or after the live session. The user prepares a position and collects credible sources. The chat exists to produce that sourced brief.

Dialectic.ai does not match opponents, run a tournament, or score speaking points for a season. It also does not try to settle every value disagreement. A claim with no retrieved source stays `unverified`. The screen says so. Inventing a correction would make the original problem worse.

## 2. You know the target customers or have direct access to them

The customer is someone in a real argument with someone they know: roommates, friends, classmates, a student organization in a heated meeting. That is us. The conversations Dialectic.ai is for already happen in our group chats, apartments, and project teams.

We can reach the first users directly:

- Our own arguments. Two of us can run a session on a topic we actually disagree about and compare the record to what we remember saying.
- Friends who will try one conversation on their phones and tell us, the same night, whether a quote was misheard or a fact-check cited something neither person trusts.
- A campus club or class discussion that wants a shared record instead of a louder person winning by default.

Competitive debaters are a second audience, not the first one. A Public Forum round can use the same transcript-and-check loop. We do not need a national debate circuit to learn whether the product works. We need a few conversations we are already part of, and permission to look at the transcript afterward.

## 3. You are familiar with (or can quickly learn) most of the required infrastructure

The intended stack is one we can learn as a team and run ourselves:

- **TypeScript, React, and Next.js** for the session UI: live transcript, claim cards, and the studying chat.
- **Go** for the session gateway. It accepts audio or transcript events over WebSockets and fans the accepted cards back out to both clients.
- **Python, FastAPI, and Pydantic AI** for the moderation agent. Pydantic models are the schema. Output validators are the guardrail. A failed validator raises a retry with the error text.
- **PostgreSQL and pgvector** for users, sessions, speaker segments, claim cards, and embeddings of the source corpus.
- **Docker** so the gateway, the API, and the database come up together.

Speech-to-text is a hosted streaming API. We do not train an audio model. Speaker identity in v1 comes from the device: each person joins on their own phone, so each audio stream already belongs to one side. A shared-laptop mode uses an explicit turn button for the same reason. Room-wide diarization from one microphone is a later upgrade, not a launch dependency.

Fact-checking is retrieval, not a model that "knows." Studying mode and a curated corpus (reference pages, datasets, and sources the user attaches) are embedded into pgvector. A live claim may also trigger a web retrieval; the fetched page is stored as a chunk before the model is allowed to cite it. The model receives chunk ids and text. It does not browse on its own.

We already have coursework-level experience with web apps and API calls. The pieces we will learn on purpose are Pydantic AI's typed agent loop, pgvector queries, and a small Go WebSocket service. We are not training a base model. After real sessions, with consent, we can fine-tune the fallacy and style labeler on transcripts we have already validated by hand. That fine-tune slots in behind the same checker. Launch does not wait on it.

## 4. Reimagines the problem space to make it significantly better

Debate Arena treats debate as a sport you practice against a bot: pick a motion, speak in a format, receive a decision. That serves competitors. It leaves the social argument untouched. Those arguments fail for a different reason. People talk past each other, cite numbers nobody looks up, and score the exchange as a personal win.

Dialectic.ai treats the argument as a shared transcript that has to become a shared record. Both people see the same cards at the same time. A fact-check is a status plus a quote from a stored source, not a paragraph the moderator prefers. A consensus names the claim ids it used. Style and fallacy notes point at the words that were said. The studying chat can only add sources by retrieving them into the same corpus the live session uses, so prep and the live record stay on one standard.

The result is better because the conversation can keep going while the record stays honest. The transcript streams immediately, because speech-to-text is not an LLM judgment. Claim cards, fallacy notes, and the consensus appear only after they pass. A false "correction" is withheld. An unverified claim stays visible as unverified, which is often the thing worth learning: nobody in the room actually had a source.

"Who won" stays in basic arbitration because people ask for it. The verdict is a claim-level consensus rendered from the structured fields, so the moderator cannot add a put-down. Both sides read one result, grounded in the checks, instead of leaving with two memories of the fight.

## 5. An LLM or generative model is core to the solution

The model is the moderator. Rules cannot listen to a messy argument and decide which sentences are factual claims, which retrieved passage bears on a claim, or how to explain a contradiction in one sentence both people can read. That is the generative step, in every mode.

In **basic arbitration** the model reads the latest speaker segments and proposes claim objects: a verbatim quote, a speaker id, a normalized claim, and a fact-check status (`supported`, `contradicted`, or `unverified`) with the chunk id it used. On a cadence, it proposes a consensus object whose reasons point at those claim ids.

In **in-depth moderation** the same model also labels argumentation style and fallacies. Each label carries the quote it applies to. The label set is closed. The explanation of *why* that quote fits the label is generated.

In **studying** the user works in a chat to prepare. Each user turn is embedded, the retriever returns chunks, and the model drafts the next prep turn: a claim worth making, the chunk ids that support it, and the quote to cite. The conversation is the interface. The deliverable is still a sourced brief. A turn that cites nothing in the retrieval is not shown.

A keyword list of fallacies would miss the sentence. A static fact database would miss the way people actually phrase a claim. The model has to propose the mapping. It does not get to publish the mapping. There is no side channel where an unchecked chat answer appears next to the cards.

## 6. Validation guardrail

Every user-visible LLM artifact is a Pydantic object that has passed deterministic checks in the API process. The Go gateway forwards a card only after FastAPI marks it accepted. The browser renders that payload. Drafts stay in server logs.

The validator is code. A second model does not grade the first one.

Transcript and claim checks:

- `quote` is an exact substring of one stored speaker segment from this session.
- `speakerId` is the speaker on that segment. A quote stolen from the other person fails `SPEAKER_MISMATCH`.
- The claim refers to a segment id that exists. Empty quotes fail.

Fact-check checks:

- `status` is only `supported`, `contradicted`, or `unverified`.
- `supported` and `contradicted` require a `chunkId` stored for this request, from pgvector or from a page just fetched and saved.
- The source quote on the card is an exact substring of that chunk. A cleaner sentence than the source fails `QUOTE_NOT_IN_CHUNK`.
- If no chunk was retrieved, the only legal status is `unverified`. The model cannot fill the gap from memory.
- Numbers in the card's correction line must appear in the chunk. A restated statistic that the source did not print fails.

Consensus checks, basic arbitration:

- Every `claimId` in the verdict exists on an accepted card from this session.
- The favored side is one of the two session speakers, or `none` when the checked claims do not support a side.
- The verdict body is rendered from the fields (side, claim ids, source titles). Free-form insults have nowhere to go.

In-depth checks:

- `fallacy` is in a fixed taxonomy we ship, such as `ad-hominem`, `straw-man`, `false-cause`, `anecdotal`, `appeal-to-authority`, and `slippery-slope`.
- `style` is in a fixed set, such as `evidence-led`, `anecdotal`, `personal`, and `question-seeking`.
- Each label includes a quote that passes the same substring test.

Studying checks:

- Every cited chunk id is in the retrieval for that turn.
- Every quoted source line is a substring of that chunk.
- A prep turn with zero citations fails when the user asked for evidence. The hold is "no sourced card yet," and the loop retries against the chunks that were actually returned.

Worked check: speaker A says "unemployment doubled after the increase." The model attributes that quote to speaker B. `SPEAKER_MISMATCH` rejects the card. A second draft marks the claim contradicted and quotes "the rate fell by half," but those words are not in the stored chunk. `QUOTE_NOT_IN_CHUNK` rejects it. A fallacy tag of `gish-gallop` fails `FALLACY_NOT_IN_TAXONOMY` because it is not in the set. A consensus that cites claim `C9` after that card was rejected fails `VERDICT_CITES_UNKNOWN_CLAIM`. None of those drafts are pushed to either phone.

## 7. Self-correction loop

Pydantic AI runs the loop inside the request that wants to publish a card or a studying turn. The user does not press retry. Both clients keep showing the last accepted record.

1. The agent receives the new speaker segments (or the studying question), the chunk texts retrieved for this call, the taxonomy, and the output model.
2. It returns a structured draft.
3. The output validator runs the checks in section 6. Success persists the card and notifies the gateway.
4. Failure raises a model retry whose message is the structured error, for example `{ "code": "QUOTE_NOT_IN_CHUNK", "path": "sourceQuote", "chunkId": "chk_14" }`, `{ "code": "SPEAKER_MISMATCH", "speakerId": "B" }`, or `{ "code": "UNGROUNDED_STATUS", "detail": "no chunk retrieved; status must be unverified" }`. The model also receives its rejected draft and the original segments.
5. It returns a full new object. Three attempts. Each attempt is validated from scratch.

If the third attempt fails, that claim is stored as `withheld` and the UI shows the transcript line with an unverified hold. It does not show the broken fact-check. The next utterance starts a new loop. Studying behaves the same way: a failed turn is not appended to the chat; the user sees that no sourced reply passed, and can attach a link or rephrase.

The retry is what makes live moderation safe enough to show. The first draft often has the right claim and a source quote that is almost the chunk. The error names the chunk. The next draft has to use the words that are actually there. Latency stays bounded because attempts are capped and the transcript itself never waits on the model.

## 8. Deployability

We can deploy the full loop ourselves.

- Docker Compose runs Next.js, the Go gateway, FastAPI, and PostgreSQL with pgvector on one host we control. A public URL is enough for two phones to join a session.
- Speech-to-text and the language model are hosted API keys on the server. Embeddings for the corpus use the same kind of hosted call and are written into pgvector.
- v1 sessions are two authenticated participants. Audio stays on each person's device stream. We store the transcript and the accepted cards.
- Cost is capped by short claim windows, a chunk limit per retrieval, and three model attempts per card. A session that runs long does not re-check the entire history on every sentence.
- Studying and live moderation share the corpus and the validator, so we deploy one system, not two backends.
- Fine-tuning the fallacy labeler is a follow-on we can run once we have labeled sessions. The deployed product is the RAG agent plus the checker. Swapping in a fine-tuned labeler later does not change what users see.

A finished deployment is two people opening a session, talking, and watching claim cards appear only when the quotes and the sources check out — or watching a hold when they do not. Tournament software, a trained foundation model, and single-mic diarization are out of the launch path. Dialectic.ai is deployed when a real conversation produces a record both people can trust.
