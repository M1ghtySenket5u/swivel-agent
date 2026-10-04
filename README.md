# Swivel Agent

Mid-scam detection, prevention, and voice education for **Swivel Transactions,
LLC** users. Paste a message, email, or call transcript; Swivel Agent verifies
every email, link, website, and phone number against Swivel's real records, then
*says out loud* what to do about it.

Python 3.10+. Free-tier only.

---

## Two API keys — both verified working

| API | Status | Evidence |
|---|---|---|
| **ElevenLabs** | Working (free) | Voice `pNInz6obpgDQGcFmaJgB` → HTTP 200, decoded to **15.7s of real speech** (RMS 4711, peak 28698). |
| **Google Gemini** | Working (free) | `gemini-flash-lite-latest`, `gemini-3.1-flash-lite`, `gemini-flash-latest`, `gemini-3-flash-preview` all → HTTP 200. JSON output is clean and parseable. |

Both keys stay on the free tier; billing is not required.

### Free-tier findings that shaped the code

- **The Search tool is NOT included on free.** Every call carrying
  `tools: [{"google_search": {}}]` returns `429 RESOURCE_EXHAUSTED`, on every
  model, while plain calls succeed. Swivel Agent therefore **never claims web
  search** and adjudicates only from facts it collected itself. This is stated
  in the judge prompt as an explicit rule, not left implicit.
- **`gemini-2.5-flash` and `2.5-flash-lite` return 404** ("no longer available to
  new users") on this key. The model ladder is built from models verified to
  work, with the retired ones kept last as a harmless fallback.
- **The free tier rate limit is real** and bites on multi-case demos. Calls that
  hit 429 sleep and retry rather than failing, so a backlog of requests degrades
  into slow-but-correct instead of wrong.

---

## Run it

```bash
pip install fastapi "uvicorn[standard]" google-genai elevenlabs phonenumbers python-dotenv pydantic
uvicorn swivel_agent.app:app --reload --port 8077
```

Then open <http://127.0.0.1:8077>.

```bash
python -m pytest tests/ -q      # 46 tests, no API keys needed
python scripts/smoke_test.py   # end-to-end, uses real keys
```

---

## How it decides

```
extract  ->  deterministic checks  ->  Gemini adjudicates  ->  voice alert
```

**1. Extraction** pulls emails, links, domains, phone numbers, and NMLS IDs out of
the text. It is speech-aware: transcription drops punctuation and inserts filler
("our NMLS ID *is* 2543894"), so identifier matching never depends on adjacency,
and spelled-out digits ("one dash eight three three…") are decoded.

**2. Deterministic checks** verify contacts against Swivel's verified records:

| | Swivel's real details |
|---|---|
| Legal | Swivel Transactions, LLC |
| HQ | 9311 San Pedro Ave., Suite 600, San Antonio, TX 78216 |
| NMLS | 2543894 |
| Domains | `getswivel.io` |
| Phone | 1-833-615-7770, 1-888-610-2574 |

and flag scam language: gift-card demands, money/crypto requests, requests for
passwords/PINs/OTPs, remote-access asks, urgency pressure, wrong-city claims.

**3. Gemini** then *adjudicates the collected evidence* — it is given facts
produced by code, and is explicitly forbidden from inventing lookups it did not
perform. The free tier has **no Google Search grounding**, so the app never
pretends to have web search.

**4. ElevenLabs** speaks the warning.

### Demo mode — no preparation required

The page demos itself. Press **Check this contact** with an empty box, or use the
buttons: **Surprise me** / **Scam example** / **Real Swivel example**.

`GET /api/demo-sample?kind=scam|real|random&source=email|message|call`

Samples come from Gemini when available and fall back to a curated offline
library otherwise, so a live demo never dies on a rate limit. Verified across all
six kind × channel combinations:

| Sample | Verdict | Voice | "Why this is genuine" |
|---|---|---|---|
| Scam (email / message / call) | `scam` 73–82 | **on** | hidden |
| Real (email / message / call) | `authentic` 0–10 | **off** | listed |

**Voice is a warning device only.** A genuine Swivel contact never triggers audio —
interrupting a customer with "Warning from Swivel Agent" while they are safely
talking to real Swivel would be alarming and wrong. This is enforced in three
places: the alert is marked `speakable: false`, the UI hides the panel, and
`/api/speak` returns HTTP 409 for a non-warning case.

For genuine contacts the app shows **why it is real** instead — the specific
details verified against Swivel's records (real domain, real toll-free lines,
matching NMLS, San Antonio HQ), so the user learns the positive signal, not just
the red flags.

### What the user actually sees and hears

Gemini returns three customer-facing fields, and the prompt forbids internal
vocabulary in all of them:

| Field | Purpose |
|---|---|
| `headline` | ≤8 words — the single most important takeaway |
| `explanation` | 2–3 plain-English sentences saying **why** it's fake |
| `what_to_do` | 2–4 ordered imperative steps to take right now |

Real output for a look-alike domain attack:

> **Warning: This message is a scam.**
> This message is a scam because it uses a fake website address,
> getswivel-secure.com, instead of our official getswivel.io website. Real
> financial companies will never ask you for your password or demand payment
> using gift cards.
>
> 1. Close the fake website immediately.
> 2. Do not share your password or buy any gift cards.
> 3. Call us directly at 833-615-7770 if you need help.

Two safeguards back up the prompt, because a model will still occasionally leak
internals:

- **`strip_internal_jargon()`** removes `F1`, `rule_id`, `deterministic`,
  `peak score`, etc. from any user-facing text. A customer never reads
  *"Fact F1 shows…"* in their own fraud warning.
- **`_first_complete_sentence()`** trims the explanation to one sentence that
  fits the TTS budget *in full*, falling back to the headline. Without it, a long
  model sentence gets sliced mid-clause and the audio says
  *"…an incorrect license number that do not belong to."*

If Gemini is unavailable, a deterministic fallback writes the explanation from
the flags themselves — the explanation box is **never** empty.

### The score floor — the most important line in the project

```python
final = max(model_score, deterministic_peak)
```

A cautious model can never talk the risk score **below** what the code already
confirmed. It may only raise a verdict, never lower one. This is what stops a
hallucinated or timid model from producing a false all-clear on a message that
literally says *"buy gift cards and send me your password."*

The deterministic layer emits **facts, never verdicts** — it says "this domain is
not `getswivel.io`", not "this is a scam" — so adjudication happens exactly once,
with the evidence attached.

---

## Two hard-won correctness rules

**Toll-free numbers have no geography.** Swivel's real lines are 833/888
toll-free. Claiming they are "not in San Antonio" would be factually wrong, so
an unmatched toll-free line is flagged as *"not one of Swivel's published lines"*,
never as a location mismatch.

**A real phone number does not clear a scam.** Fraudsters reach victims *by
calling them*. A scammer who knows Swivel's real 833 number still gets caught,
because behavioural evidence (gift cards, passwords, money) overrides a matching
contact detail. `verified_swivel` answers *"is this contact detail real?"*, never
*"is this content safe?"*.

Both are covered by regression tests in `TestLegitimateSwivelIsCleared`.

---

## Design

Japanese-inspired: light peach ground (`#FFF1E6`), light pink (`#FDE7EC`), light
red accents (`#D6455D`), generous white space, a 朱 hanko seal, and drifting
sakura petals in the background. Hiragino/Noto Sans JP font stack. No images or
build step — plain HTML, CSS, and JS served by FastAPI.

---

## Notes and limits

- **Free-tier constraints are real and are handled, not hidden.** Gemini free
  tier rotates across a 4-model ladder with backoff and honours the per-minute
  cap; not every ElevenLabs library voice works over the API (many return HTTP
  402), so the voice list is probed with a fallback ladder.
- **The Gemini prompt forbids invented lookups.** Since free tier has no Search
  grounding, "verify this website" is answered from Swivel's known records and
  code-level checks only.
- TTS output is checked for actual speech (duration + RMS), never assumed from a
  HTTP 200. Alert text is sanitized — URLs and emoji are stripped, because TTS
  mangles them aloud — and truncated at a **sentence boundary**, so a spoken
  warning never ends mid-clause. `sanitize()` cuts at the last complete sentence
  that fits the 45-word budget rather than word-slicing.
- The spoken script is always present in `alerts` regardless of the `speak`
  flag, so the UI can render it and `synthesize_alert()` can speak it later.
  Each alert costs roughly 120 ElevenLabs credits (~50 alerts on 10k/month).
- Health endpoint `GET /api/health` reports what is genuinely wired up,
  including `gemini_live`, and never claims a capability it doesn't have.
- Advisory only. Not financial advice. Not a replacement for calling Swivel.