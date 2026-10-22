## SelectIQ

**Original product:** [Aftershoot](https://aftershoot.com/)

**Pitch:** Before a photographer manually reviews hundreds of near-duplicate images, SelectIQ learns what that photographer values in this specific project and builds a checked Keep, Review, and Reject queue.

Aftershoot already detects technical problems and learns from previous selections. SelectIQ narrows the job further. The photographer provides one creative brief and makes 20–40 pair choices. The model then ranks an unseen gallery according to that photographer’s demonstrated preferences and the story they want the gallery to tell.

The problem is not “photo management.” The problem is deciding which technically acceptable photographs matter.

A wedding photographer may have 15 sharp images from the same moment. One has the cleanest composition. Another has the strongest expression. A third includes a family member important to the client. Generic quality scoring cannot decide which frame belongs in the final story.

SelectIQ does only this job. It does not edit photographs, create albums, deliver galleries, schedule clients, process payments, or replace Lightroom. Those are separate products. Adding them would make the initial experiment harder to evaluate.

### 1. SelectIQ solves one problem extremely well

SelectIQ reduces the time photographers spend making subjective Keep or Reject decisions after a high-volume shoot.

The input is intentionally limited:

- One project containing 200–500 JPEG photographs
- A short creative brief, such as “prioritize candid family connection”
- Twenty to forty pair choices made by the photographer
- Technical measurements from local image processing
- A generated preference profile based on the photographer’s choices

The output is one validated selection object containing:

- A Keep, Review, or Reject category for every photograph
- A confidence score
- The brief and preference signals used
- A short explanation
- The reason an uncertain photograph entered Review

The photographer can correct every recommendation. Original photographs remain unchanged and nothing is automatically deleted.

Success means SelectIQ reduces total culling time by at least 40 percent while recovering at least 90 percent of the images the photographer later identifies as essential. Keep plus Review cannot contain more than 50 percent of the gallery. Without that limit, the system could produce high recall by asking the photographer to review almost everything.

### 2. We can directly reach the target customer

The initial customer is an independent wedding or event photographer who regularly returns from a shoot with hundreds or thousands of photographs.

We do not need access to national studios or enterprise photography companies for the first test. We can recruit the initial 12 participants through:

- Local wedding and event photographers
- Photography studios
- Photography clubs and university photography programs
- Local Facebook, Reddit, and Discord photography communities
- Personal referrals from photographers

The pilot gives each photographer two comparable, unculled galleries. The photographer culls one manually and one with SelectIQ. We randomize which gallery receives the tool and time the complete workflow from import through final decisions.

This access is direct enough to observe the real failure. We can watch whether the photographer trusts the Review queue, whether 20–40 pair choices feel excessive, and whether the final selection misses an image the photographer considers essential.

Our first users are not casual phone photographers with 40 vacation photographs. Existing photo applications already serve that broad audience. We are testing people for whom culling is recurring professional work and saved time has clear value.

### 3. We can learn and operate the required infrastructure

The MVP uses infrastructure a student software team can build and deploy:

- Next.js for the web interface
- Python and FastAPI for the image-processing and model pipeline
- OpenCV for sharpness, exposure, and basic technical checks
- CLIP embeddings for image similarity and grouping
- A lightweight ranking model for preference learning
- A hosted multimodal model API for brief-aware reasoning
- Pydantic or JSON Schema for the first validation gate
- PostgreSQL for projects, preferences, and accepted recommendations
- S3-compatible temporary storage for compressed JPEG copies
- Docker for repeatable deployment

We do not need to train a foundation model. We use existing image embeddings and a multimodal API, then build the product-specific preference, validation, and correction pipeline ourselves.

The pieces we need to learn are narrow: creating similarity groups, fitting a simple ranker from pair choices, constructing structured multimodal prompts, and running background image-processing jobs.

We leave out infrastructure we cannot honestly complete for the first version. SelectIQ v1 does not support RAW decoding, video, Lightroom plugins, cloud synchronization across professional studios, or permanent storage of full-resolution client galleries. The pilot uses JPEG copies with a fixed project-size limit and automatic deletion.

### 4. SelectIQ reimagines the problem

Existing culling tools generally ask:

> Is this a technically strong photograph?

SelectIQ asks:

> Is this a photograph this creator would choose for this project?

That difference changes the workflow. A technically imperfect photograph may still be essential because of its emotion, subject, or role in the story. A perfectly sharp photograph may be redundant or inconsistent with the photographer’s brief.

SelectIQ combines two kinds of evidence:

1. The photographer’s demonstrated taste through pair choices
2. The project’s creative goal through a written brief

The system also admits uncertainty. A photograph does not have to be forced into Keep or Reject. Review is a deliberate category for creative tradeoffs, conflicting signals, or low confidence.

“Significantly better” means the photographer reaches a usable first selection faster without surrendering the moments they consider essential. A fast system that misses the bride’s interaction with a grandparent has failed. SelectIQ is willing to return uncertainty rather than hide that uncertainty behind a confident explanation.

### 5. A multimodal generative model is core to the product

The model is not a chatbot beside a normal photo-management screen. Its structured output is the first selection the photographer came to receive.

Traditional image processing handles technical and repetitive work:

- Detecting blur
- Measuring exposure
- Grouping visually similar photographs
- Finding near-duplicates
- Extracting image embeddings

The lightweight preference model learns which visual characteristics the photographer repeatedly favors.

The multimodal generative model performs the part those systems cannot handle alone. It reads the creative brief, examines shortlisted images in context, compares them with the photographer’s preference profile, and explains why a photograph belongs in Keep, Review, or Reject.

For example, the brief might say, “Prioritize candid family connection over posed portraits.” The model must recognize which photographs contain candid interaction, compare several usable frames, and explain why one better supports that direction.

The model returns JSON rather than unrestricted prose:

```json
{
  "projectId": "project_14",
  "recommendations": [
    {
      "photoId": "IMG_2048",
      "category": "review",
      "confidence": 0.71,
      "briefSignals": ["candid interaction", "family connection"],
      "preferenceSignals": ["natural expression", "close framing"],
      "technicalClaims": [],
      "explanation": "Strong match for the candid family brief, but a similar frame received a higher preference score."
    }
  ]
}
```

Prose outside this structure is discarded.

### 6. No model artifact reaches the user without validation

The model’s draft is the product, but the draft is untrusted.

A deterministic validator runs on the server before the API returns any recommendation. The client can render only a selection object marked as accepted by the validator. Failed drafts remain internal and never appear in the interface.

The validator enforces the following rules:

- The response matches the required schema.
- Every photo ID exists in the current project.
- Every project photo appears exactly once.
- The model cannot introduce, duplicate, or omit photographs.
- Category must be Keep, Review, or Reject.
- Confidence must be between zero and one.
- Any recommendation below the confidence threshold must enter Review.
- Keep plus Review cannot exceed the configured queue limit.
- Technical claims must agree with OpenCV measurements.
- A photograph cannot be described as blurred if its measured sharpness passed the threshold.
- An explanation can reference only signals supplied in the brief, preference profile, image metadata, or deterministic analysis.
- Images protected by a rule in the creative brief cannot enter Reject when confidence is insufficient.
- No response may contain an instruction to delete or overwrite an original file.
- Every displayed number must match a value computed by the application.

Worked check: suppose the model places `IMG_2048` in Reject and says the photograph is out of focus. The OpenCV sharpness score shows that the image passed the focus threshold, and the model reports confidence of only 0.61. The validator rejects the draft for an unsupported technical claim and insufficient confidence for Reject. The photographer never sees that version.

The validator does not pretend it can prove subjective artistic quality. It verifies the facts, references, constraints, and uncertainty rules that can be checked deterministically. The photographer remains the final validator of creative judgment.

### 7. Validation failures automatically return to the model

The correction loop runs on the server before a response reaches the photographer.

1. The model receives the creative brief, preference profile, shortlisted images, deterministic measurements, and output schema.
2. The model returns a complete selection object.
3. The validator checks the object.
4. If validation succeeds, the server stores the accepted version with the validator version and returns it to the interface.
5. If validation fails, the server sends the model the original input, previous draft, and structured error list.
6. The model returns a full replacement object rather than a patch.
7. The validator runs again from the beginning.

A validation error might look like this:

```json
{
  "code": "UNSUPPORTED_TECHNICAL_CLAIM",
  "path": "recommendations[18].explanation",
  "photoId": "IMG_2048",
  "expected": "Do not describe this image as out of focus",
  "actual": "Rejected because the subject is out of focus"
}
```

Another error might report:

```json
{
  "code": "QUEUE_LIMIT_EXCEEDED",
  "expectedMaximum": 250,
  "actual": 317
}
```

The loop permits three model attempts. Temperature remains low so retries behave as corrections rather than entirely new creative responses.

If the third attempt fails, the photographer does not receive the broken draft. The affected images move to an unranked Review state using the deterministic fallback, and the interface explains that SelectIQ could not produce a validated recommendation.

Photographer corrections create a second learning loop. Moving a photograph from Reject to Keep updates the preference dataset associated with that creator. Future rankings use the corrected decision rather than the model’s original answer.

### 8. We can fully deploy the MVP

A deployable SelectIQ v1 includes:

- Account creation
- JPEG project upload
- A 200–500 photo project limit
- Image preprocessing
- Similarity grouping
- Creative-brief entry
- Twenty to forty pair choices
- Preference-profile generation
- Multimodal ranking
- Deterministic validation
- Automatic retry after validation failure
- Keep, Review, and Reject results
- Manual correction
- Exportable selection decisions
- Automatic deletion of uploaded images after the retention period

The web application and API can run in Docker containers on a platform such as Render, Railway, Fly.io, or AWS. PostgreSQL stores user preferences and accepted recommendation metadata. An S3-compatible service stores compressed working copies temporarily. The model provider key remains on the server.

Users cannot bypass validation by calling the model directly from the browser. The server owns the entire generation, validation, correction, and release loop.

Processing cost remains bounded:

- The system groups images with local or server-side open-source models.
- It sends only shortlisted and compressed candidates to the multimodal model.
- Each project has a fixed maximum image count.
- Each model request receives at most three attempts.
- Uploaded photographs expire automatically.
- The application never stores full camera-card archives indefinitely.

What “fully deployed” means for v1 is a photographer signing in, uploading a JPEG gallery, entering a brief, making pair choices, and receiving only validated Keep, Review, and Reject recommendations that can be corrected and exported.

It does not mean supporting every camera format, editing photographs, processing full video shoots, or replacing the photographer’s existing delivery platform. Those features are not required for SelectIQ to perform its single job in production.
