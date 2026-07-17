# Real-Time Translated Phone Calls — Product & Implementation Plan

**Research date: 17 July 2026.** Everything marked ⚠ is volatile: model names, prices, language lists, preview status, and telecom regulations below were verified on that date against official sources but change frequently. The implementation agent must re-verify each ⚠ item against the linked provider docs before building on it.

This is a planning and critical-review document only. No production code, no infrastructure, no paid resources, no phone numbers were created as part of producing it.

---

## 1. Executive summary (the opinionated version)

1. **The product premise is sound and the timing is unusually good.** The recipient-side constraint (ordinary phone call, no app, no link, no account) forces a PSTN architecture, and that is exactly the niche OS vendors do not cover: Apple's iOS 26 Live Translation for phone calls supports ~10 languages and **not Hebrew**; Samsung Galaxy AI Live Translate also **does not support Hebrew** (confirmed against Apple's feature-availability page and Samsung support/community pages). Hebrew↔French is a real, underserved corridor with a large bilingual-need population (≈150k+ French speakers in Israel, plus Israeli businesses dealing with France/Belgium/Switzerland).
2. **Do not promise the caller's own number as caller ID. This is the single biggest place the premise must be challenged.** France (since 1 Jan 2026) masks French mobile CLIs on unauthenticated international calls, and Israel has regulations in progress requiring carriers to block international calls presenting Israeli numbers. The reliable, commercially honest design is: **a dedicated local virtual number per user (or per account) in the destination country**, presented consistently, with callback routing back through the platform. Details in §5.
3. **Architecture: two call legs bridged by a real-time media agent.** The initiating user connects via the mobile web app over WebRTC (cheap, wideband audio); the recipient gets an ordinary PSTN call. Use a voice-agent framework (recommendation: **LiveKit Agents + SIP trunk**, with Pipecat as runner-up) rather than hand-rolling a websocket media server — barge-in, endpointing, jitter, and reconnection are solved problems there and this matters enormously when a mid-tier AI model writes most of the code.
4. **Translation: build a cascaded pipeline (streaming STT → LLM translation → streaming TTS) as the product backbone, with pluggable speech-to-speech engines behind the same interface.** Direct S2S is genuinely attractive now — OpenAI shipped `gpt-realtime-translate` (May 2026, $0.034/min) and Google shipped `gemini-3.5-live-translate-preview` (June 2026, Hebrew supported both directions) — but ⚠ OpenAI's translate model **cannot output Hebrew** (13 output languages only), and Google's is a preview with no SLA and 15-minute audio session caps. The cascade is the only path that is fully controllable for Hebrew today (number/name accuracy, glossaries, consistent voices) and it keeps you vendor-portable. Benchmark S2S continuously; switch per-direction when one clearly wins.
5. **Turn-based (consecutive) interpretation for v1, not simultaneous.** On a narrowband phone call with no visual channel for the recipient, consecutive interpretation with tight latency (target: translated audio starts ≤2.0s after end of utterance at P50) is more intelligible and more testable than overlapping simultaneous audio. Simultaneous mode is an experimental track.
6. **No recording in v1.** Recording is the future revenue engine (transcripts, summaries, CRM), but France is effectively all-party consent (Code pénal art. 226-1) and GDPR + Israel's Amendment 13 (in force since Aug 2025) make stored audio the highest-liability asset in the system. Ship live translation with ephemeral processing first; add recording as an explicit, consent-gated paid feature in a later milestone with legal review.
7. **No voice cloning in v1.** Standard high-quality synthetic voices, one consistent voice per direction. The TTS layer is an interface so cloned voices can be added later without touching call logic. EU AI Act Article 50 transparency obligations apply from **2 Aug 2026**, so the call-start disclosure ("this call is translated by AI") must be in v1 anyway.
8. **Business: start as a prosumer/SMB tool for the Israel↔France corridor**, priced as a subscription with included minutes (~₪49–99 / €12–25 per month), not per-minute consumer pricing. Expected COGS ≈ **$0.10–0.22 per conversation minute** (§9), supporting 60–75% gross margin at those prices. The "two telephony charges" worry is real only if both legs are PSTN; the recommended design makes leg A a ~$0.004/min app leg, so you pay roughly **one** PSTN termination per conversation minute plus AI costs.
9. **Cloud-only development is fully realistic.** GitHub (private) + Cursor Cloud Agents for all coding; Vercel for the web app; Railway (or Fly.io) for the always-on voice worker; Neon Postgres; GitHub Actions CI; approvals from the founder's phone via GitHub PR reviews and environment-protection gates. ~85% of the build effort is agent-executable; the irreducible human ~15% is accounts, billing, identity/number verification, listening to real calls, and legal review (§13).
10. **Model routing:** top-tier model (e.g., Claude Opus-class or GPT-5.6-class in Cursor) for architecture, the realtime audio core, and post-incident debugging; a mid-cost model (Grok 4.5 or Composer-class — both acceptable; pick by a paid calibration bake-off on Milestone 2, §11) for the bulk of implementation; a cheap fast model for tests, docs, and lint fixes; an *independent* model for final review of each milestone.

---

## 2. Challenging the premise — what should change

The core idea survives scrutiny, with four corrections:

- **Caller-ID promise → downgrade.** "Recipient sees your own number" cannot be promised for the launch corridor (§5). Replace with "recipient sees a consistent local number that reaches you when called back." This is arguably *better* commercially: local numbers get answered more, and callback routing through the platform means the return call can also be translated.
- **"Feels like a normal call" → set expectations honestly.** With consecutive interpretation, the call feels like talking through a fast human interpreter, not like a normal monolingual call. Marketing and onboarding must frame it that way; users who expect zero-delay conversation will judge the product a failure even at excellent latency.
- **Don't build for "any language" — build for corridors.** A language pair is not just a settings row: each direction needs a validated STT/MT/TTS (or S2S) chain, a benchmark set, and telephony/regulatory validation for the destination country. The UI should offer languages from a server-side capability matrix, so adding a pair is a config-and-validation exercise, not a code change — but each new pair still requires a validation gate.
- **The simplest viable MVP is even simpler than described.** Before building the full two-leg bridge, Milestone 2 (§12) is a "translated voice note" loop — the founder speaks Hebrew into the web app and hears French back — which validates the entire AI pipeline for ~zero telephony cost and produces the benchmark harness. Only then wire in PSTN.

---

## 3. Feasibility triage

**Technically possible and dependable now**
- Bridging an app/WebRTC leg with a PSTN leg through a media agent (mature: LiveKit SIP, Twilio Media Streams, Telnyx bidirectional streaming).
- Cascaded streaming translation Hebrew↔French with ~1.5–2.5s end-of-speech→translated-audio (ElevenLabs Scribe v2 Realtime does Hebrew STT at ~150ms model latency; LLM translation; Azure/ElevenLabs TTS).
- Consecutive (turn-based) interpretation UX with barge-in handling.
- Local virtual numbers in Israel and France (with documented KYC requirements, §5).
- In-call audible AI disclosure, DTMF consent capture.

**Possible but experimental (build behind an adapter, don't promise)**
- Continuous simultaneous S2S translation: `gemini-3.5-live-translate-preview` (Hebrew both directions, preview, no SLA, 15-min audio caps ⚠) and `gpt-realtime-translate` (GA-priced at $0.034/min but **no Hebrew output** ⚠).
- Prosody/emotion carry-over from speaker to synthetic voice.
- Full-duplex overlapping translation on a narrowband phone leg.

**Not reliable enough for a commercial promise**
- Displaying the caller's own personal number into France or Israel from international routes (§5 — regulators actively block/mask this pattern).
- "Perfect accuracy for numbers, prices, and names" — mitigate (§4.4) and measure, never promise.
- Guaranteed >60-minute uninterrupted AI sessions on any single provider (session caps exist everywhere; the bridge must support mid-call engine reconnection).

**Postpone deliberately**
- Voice cloning (v2+, §7). Call recording and stored transcripts (v1.5+, §8). Native mobile apps (mobile web/PWA first — no app-store review cycle, faster agent-driven iteration). CRM integrations, API product, call-center features.

**Likely to get easier soon (design for swap-in)**
- S2S translation quality/coverage (both OpenAI and Google shipped dedicated translation models within 10 weeks of the research date; Hebrew output on OpenAI's model is a plausible near-term addition ⚠).
- Cheaper realtime audio (OpenAI's mini realtime tier is already 3× cheaper than flagship).
- EU AI Act machine-readable audio marking tooling (Code of Practice published June 2026; ecosystem tooling immature today).

**Must be tested before committing to architecture (the "Spike List" — each is a small agent task with a measurable output)**
1. Hebrew STT WER on real 8kHz telephone audio: Scribe v2 Realtime vs Azure STT vs `gpt-realtime-whisper` (marketing figures are for wideband; telephone-band Hebrew is the real question).
2. Hebrew TTS quality/latency: Azure neural voices vs ElevenLabs v3/Conversational (Hebrew is not in ElevenLabs' low-latency Flash v2.5 ⚠) vs OpenAI TTS — MOS-style listening test over a phone leg.
3. Gemini live-translate preview: instruction fidelity (does it ever "answer" instead of translating?), latency, session-cap behavior, Hebrew quality both directions.
4. Number/name/date accuracy test set (200 utterances, Hebrew and French, with prices, phone numbers, addresses, dates) run through each candidate chain.
5. LiveKit SIP ↔ Telnyx/Twilio trunk: end-to-end audio latency and DTMF reliability into Israeli and French mobiles.
6. Echo/feedback behavior when translated audio plays into the recipient leg while they speak (barge-in tuning).

---

## 4. System architecture

### 4.1 Call topology

```
Founder's phone (mobile web app, WebRTC)          Recipient's phone (ordinary PSTN call)
        │                                                     │
        ▼                                                     ▼
   [Leg A: app/WebRTC audio, wideband]              [Leg B: SIP trunk → PSTN, 8kHz]
        └────────────► LiveKit room ◄────────────────────────┘
                           │
                 Translation Agent (Python worker)
                 ├─ Direction A→B: STT(he) → LLM translate → TTS(fr) → play to B
                 ├─ Direction B→A: STT(fr) → LLM translate → TTS(he) → play to A
                 ├─ Turn manager (endpointing, queueing, barge-in policy)
                 └─ Engine adapters (cascade | openai-s2s | gemini-s2s per direction)
```

- **Leg A is app audio, not PSTN.** Wideband Opus gives materially better STT for the caller, costs ~$0.004/min instead of PSTN rates, shows live captions/status in the UI, and gives the caller controls (repeat, slow down, end). PSTN dial-out to the caller's own phone is a fallback mode (worse audio, still works — and needed if app audio fails mid-call).
- **Leg B is a normal outbound PSTN call** via an elastic SIP trunk, presenting the platform's local number for the destination country.
- **One agent process per call**, stateless across calls; call state in Postgres, transient state in the process. Region: EU (Paris/Frankfurt) to minimize RTT to both France and Israel.

### 4.2 Why a framework instead of raw media streams

Raw Twilio Media Streams (base64 μ-law over websocket) is fully workable and well-documented, but you then own endpointing, jitter buffers, barge-in, echo control, and reconnection — precisely the code where a cheaper implementation model produces subtle, expensive bugs. **LiveKit Agents** (1.x, mature, open source, Python/Node) gives turn detection, interruption handling, STT/LLM/TTS plugin architecture, SIP support (inbound/outbound trunks, DTMF, dispatch rules), and is a named integration partner for Gemini live-translate. Pipecat (Daily) is a close equivalent. Trade-off: one more vendor (LiveKit Cloud ~⚠ $0.005–0.01/participant-min, or self-host later), in exchange for deleting the highest-risk 30% of the codebase.

**Recommendation: LiveKit Cloud + LiveKit Agents (Python) + Telnyx elastic SIP trunk** (Telnyx: cheaper trunk media at $0.002/min + $0.0035/min streaming if ever needed, good EU coverage; Twilio is the fallback — better docs for AI agents, higher prices: e.g., Israel mobile $0.1868/min vs. with Israeli CLI $0.0646/min ⚠). Get pricing for the corridor from both before the trunk decision (agent task, no account needed for published rates).

### 4.3 Translation engine comparison

| Criterion | Cascade (STT→LLM→TTS) | OpenAI `gpt-realtime-translate` | Gemini `3.5-live-translate-preview` | OpenAI `gpt-realtime-2.1` (prompted as interpreter) |
|---|---|---|---|---|
| Hebrew in / out | ✔ / ✔ (Scribe v2 RT: he "good" tier; TTS via Azure/Eleven v3) | ✔ / **✘ no Hebrew output** ⚠ | ✔ / ✔ ⚠ | ✔ / ✔ |
| French in / out | ✔ / ✔ (excellent everywhere) | ✔ / ✔ | ✔ / ✔ | ✔ / ✔ |
| Latency | ~1.2–2.5s end-of-speech→audio (sum of stages) | Continuous, "keeps pace with speaker" | Continuous, "a few seconds behind" | Turn-based, ~1–2s |
| Numbers/names control | **Best** — text is inspectable, glossary + regex guards possible | None (no custom prompting ⚠) | Limited | Prompt-level only; risk of paraphrase |
| Interpreter fidelity (never "answers back") | Guaranteed by construction | Purpose-built interpreter | Purpose-built interpreter | **Known failure mode** — assistants sometimes respond instead of translating; must be tested hard |
| Voice control / future cloning | Full (any TTS) | None (no voice selection ⚠) | Limited (SynthID watermark built in) | 10 stock voices |
| Session limits | None inherent; reconnect per component | 60-min realtime session cap ⚠ | 15-min audio cap unless extended ⚠; preview, no SLA | 60-min cap ⚠ |
| Cost /min /direction | ~$0.03–0.10 (see §9) | $0.034 flat ⚠ | Preview pricing unstable ⚠ | ~$0.04–0.10 blended ⚠ |
| Vendor lock-in | Low (three swappable parts) | Medium | Medium | Medium |
| Commercial readiness | High | High (GA) but wrong languages for launch pair | Low (preview) | Medium |

**Decision:** cascade is the v1 backbone for both directions; `TranslationEngine` is an interface with `cascade`, `openai_s2s`, `gemini_s2s` implementations selectable per direction per call (config, not code).

Default cascade components to benchmark first:
- **STT = ElevenLabs Scribe v2 Realtime** (he+fr, ~150ms, accepts 8kHz/μ-law) with Azure STT as the alternate.
- **MT = a fast LLM** (GPT-4.1-mini/GPT-5-mini-class or Gemini Flash-class ⚠) with rolling conversation context, a per-account glossary, and streaming output.
- **TTS = ElevenLabs Flash v2.5 for French** (~75ms, ~$0.06/min) and **Azure neural Hebrew voice** (verify current he-IL voice list ⚠) or ElevenLabs v3-family for Hebrew — Hebrew TTS choice is Spike #2, decided by listening test.

### 4.4 Turn-taking, duplex, and the hard conversational problems

- **Half-duplex consecutive mode (v1):** VAD endpointing (~400–700ms silence threshold, tunable per language), translate the completed utterance, play it to the other side. While translation audio plays to B, B's own speech is still captured; if B barges in, playback ducks then stops, and the interrupted remainder is re-queued as text on A's screen (never lost silently).
- **Overlap policy:** if both speak simultaneously, first-committed utterance wins the channel; the other is queued. Queue depth is capped (max 2 pending utterances per direction); beyond that, the agent plays a brief earcon ("please pause") in the appropriate language rather than building unbounded delay.
- **Audio cues matter more than people expect:** a soft tick while the counterpart is speaking (so silence isn't read as a dead line), a short chime before translated playback, and a spoken intro on answer: recipient hears, in their language, "This is a translated call from [name]. You will hear their words in French after they speak." This one sentence removes most recipient confusion and doubles as the AI-disclosure obligation (§8).
- **Numbers/names guardrails:** the MT prompt pins digits ("copy all numbers verbatim as digits"), a post-check compares digit sequences between source and translation and triggers a re-translate on mismatch, and the caller's UI shows the outgoing translation as text so they can catch errors ("repeat" button re-plays; "fix" lets them restate).
- **Accents, noise, fast speech:** handled at STT selection (benchmark on telephone-band, accented test audio, Spike #1) plus an in-call "confidence low" behavior — if STT confidence collapses, the agent says (in the speaker's language) "I didn't catch that, please repeat" rather than emitting garbage translation.

### 4.5 Sessions, reconnection, long calls

Every upstream connection (STT ws, LLM stream, TTS ws, or S2S session) must be wrapped in a supervisor that reconnects with backoff and replays in-flight utterance state; the call legs stay up independently of AI-session health. Mid-call engine failover order: primary cascade → alternate STT/TTS vendor → grace message ("one moment, reconnecting the interpreter") after 5s of no capability → offer callback if >30s. Long-call target: 60 minutes without degradation (forces at least one planned S2S session rotation if S2S is in use — rotate during silence).

---

## 5. Caller ID — the honest analysis

**Is presenting the user's own number technically possible?** Yes at the API level: Twilio's Verified Caller ID (OTP verification of number ownership) and equivalents at Telnyx/Vonage allow setting a verified non-platform number as CLI. (Twilio is even sunsetting the looser "Transit Caller ID" path in June 2026 in favor of verified-only ⚠.)

**Will it be displayed to the recipient in the launch countries? Mostly no.**
- **France:** the MAN authentication chain (law of 24 July 2020, generalized from Oct 2024) requires operators to authenticate French CLIs and cut/mask unauthenticated calls. Since **1 Jan 2026**, ARCEP explicitly requires that international-origin calls presenting a French mobile (06/07) number that cannot be authenticated be delivered with **caller ID masked**; ARCEP guidance also states a 06/07 CLI may not be used for calls generated independently of a French mobile access. A French user's own mobile number, presented via a foreign cloud platform, will show as "hidden number" at best. France is not in STIR/SHAKEN — MAN is its equivalent.
- **Israel:** the Ministry of Communications has draft regulations (⚠ verify current in-force status — expected effective within the year of publication) requiring international providers to **block** inbound international calls presenting Israeli landline CLIs, and mobile operators to block ones presenting Israeli mobile CLIs except for genuine roamers. This follows the CEPT/ECC recommendation pattern. Presenting an Israeli user's own mobile from a cloud route into Israel is exactly the blocked pattern.
- **Callback behavior:** even where own-number CLI works, callbacks go to the user's real phone *outside* the platform — untranslated. That breaks the product loop.

**Recommended design (and what to promise):**
1. **Per-account local virtual numbers** in the destination country: an Israeli number for calls into Israel, a French number for calls into France. These originate "locally" from the carrier's perspective, avoid the anti-spoofing filters, get materially better answer rates than foreign CLIs, and — bonus — cut Twilio termination prices sharply (Israel mobile $0.1868→$0.0646/min with Israeli origin; France mobile $0.1603→$0.0404/min with EEA origin ⚠).
2. **Callbacks to that number route into the platform**: an inbound IVR in the recipient's language ("connecting you to [name] with translation"), ringing the user's app/phone and starting a translated session. This turns the caller-ID constraint into a feature.
3. KYC reality check: Israeli numbers require identity + worldwide address (Telnyx) or a regulatory bundle (Twilio); **French local numbers require a French address** (Twilio guideline: no PO boxes ⚠) — for the beta, a French mobile-type number or the founder's own French business connection may be needed; if unobtainable, calls into France can present the account's Israeli number (a foreign CLI is displayed normally under current rules — only *French* CLIs from abroad are masked), accepting a lower answer rate. Test both (Spike #5 extension).
4. **Fallback ladder when the preferred CLI can't be used:** platform local number → platform foreign number + pre-call SMS/WhatsApp heads-up option ("[Name] will call you in 1 minute via a translation service from +972-…") → clear in-app warning to the caller that the recipient may see an unknown number.
5. **Commercial promises:** promise "consistent dedicated number + callback continuity." Do not promise own-number display anywhere; where a corridor supports verified own-number CLI cleanly (e.g., US/Canada destinations under STIR/SHAKEN with verified caller ID), enable it as a per-corridor capability flag later.

Anti-spoofing hygiene on our side: only E.164-validated, OTP-verified numbers ever placed in a `From` field; CLI selection is server-side only, never client-supplied; audit-log every CLI decision.

---

## 6. Quality targets (measurable definitions)

"End-of-speech → translated playback" (EOS→TP) = last speech frame of an utterance to first translated audio frame delivered on the other leg. Measure in the agent with per-stage timestamps; report P50/P90/P95 per direction.

| Metric | Prototype | Paid beta | GA | Enterprise |
|---|---|---|---|---|
| Call setup success (dial attempts reaching ringing, excl. invalid numbers) | ≥90% | ≥97% | ≥99% | ≥99.5% |
| EOS→TP latency P50 / P90 / P95 (s) | ≤2.5 / 4.0 / 5.0 | ≤2.0 / 3.0 / 4.0 | ≤1.5 / 2.5 / 3.5 | ≤1.2 / 2.0 / 3.0 |
| Time to first translated audio after answer | ≤6s | ≤4s | ≤3s | ≤3s |
| Word-level accuracy (WER, telephone-band, per language) | ≤18% | ≤12% | ≤9% | ≤7% + custom vocab |
| Semantic adequacy (COMET-style score on test set, and human 1–5 rating) | ≥3.7/5 human | ≥4.0/5 | ≥4.3/5 | ≥4.4/5 |
| Number/date/name exact-match accuracy | ≥90% | ≥96% | ≥98.5% | ≥99% + glossary guarantees |
| Utterances lost or never translated | ≤3% | ≤1% | ≤0.3% | ≤0.1% |
| Mid-call AI-session drop requiring audible gap >5s | ≤10% of calls | ≤3% | ≤1% | ≤0.5% |
| Reconnect time after component failure | ≤10s | ≤5s | ≤3s | ≤2s |
| 30-min call completes without manual retry | ≥80% | ≥95% | ≥98% | ≥99% |
| Calls requiring user-initiated retry | ≤15% | ≤5% | ≤2% | ≤1% |
| User satisfaction (post-call 1–5 prompt) | founder-only | ≥4.0 | ≥4.3 | ≥4.5 + NPS |
| Noisy-condition degradation (WER delta at 10dB SNR) | measured only | ≤+8pts | ≤+6pts | ≤+5pts |

Do not publish latency or accuracy numbers in marketing until beta targets are met on two consecutive weekly benchmark runs.

---

## 7. Voice identity — now and later

**V1:** two fixed, high-quality stock voices (one per output language), chosen by listening test; voice consistency per direction so the recipient learns "that voice = the caller." The TTS adapter interface (`synthesize(text, lang, voice_profile, style_hints)`) is the only contract call logic knows.

**Future capability analysis:**
1. *Voice-character preservation without enrollment* (style transfer from live audio): experimental everywhere; prosody hints (question intonation, emphasis) can be partially carried via SSML/style controls now — cheap win, do in v1.5.
2. *Instant cloning* (seconds of audio): works (ElevenLabs IVC and peers) but is the highest-abuse-risk feature in the product; never from call audio.
3. *Professional cloning* (30+ min, verification): best quality, works cross-lingually (ElevenLabs professional clones speak all supported languages), adds ~zero per-call latency once created; the right eventual choice for paying users.
4. *Direct cross-lingual voice conversion* (S2S preserving speaker timbre): research-grade for phone-band production use; revisit yearly.
5. *Prosody/emotion preservation:* partial today (style tags, emotive TTS models); treat as "best effort," never a promise.
6. *Telephone bandwidth:* 8kHz μ-law destroys much of what makes a cloned voice recognizable — set expectations; cloned voices matter more for the app-side listener (wideband) than the PSTN side.
7. *Latency:* personalized voices on current APIs add little marginal latency vs stock (same model families), but verify per provider ⚠.
8. *Fallback:* every cloned-voice call must degrade silently to the stock voice on provider failure.

**Enrollment/consent design (for the future milestone, specified now so architecture reserves the hooks):** separate enrollment flow, never in-call; disclosure text + recorded verbal consent phrase + OTP to the enrolled person's own phone number (prevents enrolling someone else); liveness check (read a dynamic sentence); samples encrypted, retention-limited, deleted from provider on revocation (verify provider deletion APIs ⚠); revocation immediate (flag flips to stock voice); full audit history; impersonation report channel.

EU AI Act Art. 50 deepfake-labelling obligations (from 2 Aug 2026) reinforce: synthetic/cloned speech must be disclosed to the human hearing it — the call-start announcement already does this; machine-readable marking of generated audio (Art. 50(2)) applies to the *AI-system provider* — using vendors like Google (SynthID embedded) helps; track the Commission's Code of Practice as tooling matures ⚠.

---

## 8. Recording, transcription, and the legal envelope

Consent map (not legal advice; counsel required before launching any stored-recording feature):

- **Israel:** one-party consent (Secret Monitoring Law 5739-1979) — a call participant may record. But Amendment 13 to the Privacy Protection Law (in force 14 Aug 2025) adds GDPR-like duties with real teeth (PPA fines in millions of ₪, statutory damages without proof of harm, possible DPO duty for systematic large-scale processing ⚠).
- **France:** effectively all-party consent — Code pénal art. 226-1 criminalizes recording private words without consent (1 year / €45,000), and GDPR requires a lawful basis, notice, minimization, retention limits.
- **Cross-border rule:** always apply the strictest law on the call → **all-party consent, always, everywhere** is the only policy worth engineering.

**Policy decisions (recommended):**
- Recording **off by default**; v1 ships without it entirely. Live translation processes audio ephemerally (in-memory; provider retention set to minimum/zero-retention options where offered ⚠ verify per vendor DPA).
- When recording ships: caller opts in per call before dialing; recipient hears a notice in their language and consents by keypad (DTMF 1) or clear verbal yes captured as evidence; refusal → **translation continues, recording stays off** (this keeps the product usable and the consent honest). Consent evidence (who, when, prompt version, DTMF/verbal) stored durably, separate from content.
- **Transcript-only storage is materially safer than audio** (smaller biometric/impersonation surface, easier redaction/search) — offer transcripts+summary as the paid tier's default artifact; audio storage as a separate, later, higher-tier option.
- Data handling: EU region storage for EU data subjects; encryption at rest (per-tenant keys via cloud KMS) and TLS everywhere; retention default 90 days with user-set 0–365; user access/export (PDF/text) and one-tap deletion including provider-side artifacts; phone numbers masked in logs; processing agreements (DPA) with every AI vendor, with training-use opt-outs ⚠.
- **Initially prohibited categories:** medical, legal-advice, and financial-advice calls excluded from *recorded* products in ToS (translation-only is fine); revisit with counsel.
- App-store/AI disclosures: privacy nutrition labels list call audio/transcripts; the audible "translated by AI" notice at call start covers EU AI Act Art. 50 interaction transparency from 2 Aug 2026.
- **Greatest-liability features, in order:** stored audio of non-consenting parties > voice cloning misuse > stored transcripts > live-only translation. Sequence the roadmap accordingly (exactly what this plan does).
- **Counsel required before:** any recording feature launch (IL + FR counsel), ToS/privacy policy publication, voice-cloning launch, and any marketing claim about legality of recording.

---

## 9. Commercial strategy

**Beachhead (narrowest viable market):** French-speaking residents of Israel (olim, cross-border families, small-business owners) and Israeli SMBs dealing with French-speaking suppliers/clients/institutions — plus the mirror (French residents dealing with Israeli institutions). Strong pain (bureaucracy, suppliers, property, family logistics), willingness to pay (current alternative is a bilingual friend or a paid interpreter), repeat usage, one legal regime pair to master, tight-knit community channels for acquisition (French-Israeli associations, Facebook/WhatsApp olim groups, accountants/lawyers serving olim), and — decisively — **no OS-vendor competition for Hebrew** today. Start as a *personal/prosumer tool → paid beta → small-business product*. Consumer-mass-market, developer API, and call-center products are later stages, not launch shapes.

**Differentiation honesty table:**

| Claimed differentiator | Defensible? |
|---|---|
| Calls any normal phone, recipient installs nothing | Yes vs. app-to-app; OS vendors partially match (caller-side device translation) but not from web/any device, and not for Hebrew today |
| Hebrew quality (and corridor-tuned glossaries) | Defensible for 1–3 years; erodes when Apple/Samsung/Google add Hebrew ⚠ — build the business layer before then |
| Local caller ID + translated callback continuity | Yes — genuinely hard, telecom-regulatory moat, OS features can't do this |
| Transcripts, summaries, action items, CRM, retention controls | Yes vs. OS features; copyable by CPaaS incumbents — defensibility is workflow depth + trust, not tech |
| Recording with proper two-country consent machinery | Yes — compliance is a feature businesses pay for |
| Voice identity | Not defensible alone (vendor feature); nice retention hook |
| Raw translation quality | Not defensible — you rent it; assume competitors rent the same models |

**Pricing (recommended):** subscription with included minutes + overage, prepaid-only at start (fraud control doubles as cash-flow control): e.g., Personal €12/mo — 60 min; Pro €29/mo — 200 min + transcripts/summaries (when shipped); Business €79/mo — 600 min + 3 seats + business number; overage €0.25–0.35/min; annual discount. Pure pay-per-minute kills habitual use; pure flat-rate invites abuse.

**Unit economics per conversation-minute (recommended architecture: app leg + one PSTN leg to France mobile, cascade translation), July 2026 prices ⚠:**

| Component | Conservative | Expected | High |
|---|---|---|---|
| PSTN leg B (FR mobile, EEA-origin CLI) | $0.040 | $0.040 | $0.160 (foreign CLI/other routes) |
| Leg A app audio + media infra (LiveKit-class) | $0.010 | $0.015 | $0.030 |
| STT ×2 directions | $0.020 | $0.035 | $0.060 |
| LLM translation | $0.003 | $0.006 | $0.015 |
| TTS ×2 directions | $0.030 | $0.075 | $0.150 (all-ElevenLabs premium) |
| Hosting/DB/monitoring amortized | $0.005 | $0.010 | $0.020 |
| Payment fees + fraud/failed-call allowance | $0.005 | $0.015 | $0.040 |
| **Total COGS/min** | **~$0.11** | **~$0.20** | **~$0.48** |

At €0.30/min effective revenue (€29 for 200 min ≈ €0.145/min — note: subscription breakage typically lifts effective/used-minute revenue well above face value), expected gross margin is ~55–70%.

**Largest margin risks:** (1) TTS choice for Hebrew (Azure ~$0.013/min vs ElevenLabs ~$0.06–0.12/min — a 5–10× swing), (2) losing local-CLI pricing (4× telephony swing), (3) S2S engines double-billing both directions continuously if adopted naively, (4) fraud (international revenue-share fraud is the classic CPaaS killer).

**Answer to the explicit "two call legs = two telephony charges" question:** with two PSTN legs you would indeed pay ~two terminations per minute; the app-leg design reduces that to ~one PSTN termination + ~$0.004–0.015 of app/media cost — one of the strongest reasons to keep the caller in the app.

**Cost-abuse controls (all in v1):** prepaid balance only; per-call hard cap (30 min default) and daily per-user minute cap; destination allowlist (IL, FR only at launch) with premium-rate/satellite prefixes blocked outright; concurrent-call limit (1/user at launch); provider spend alerts + our own metered kill switch (auto-suspend outbound when daily spend exceeds budget ×1.5); velocity-based fraud rules (new account + many distinct destinations = block); SMS-verified accounts; rate limits on call initiation; no number entry via URL parameters.

---

## 10. Cloud-only development workflow (empty repo → production)

- **Repo:** single private GitHub monorepo: `apps/web` (Next.js PWA), `services/agent` (Python LiveKit agent), `packages/shared` (types/contracts), `docs/` (ADRs, milestones, runbooks), `infra/` (IaC where used). Branch protection on `main`; trunk-based with short-lived `cursor/*` branches and PRs; every PR from an agent, every merge approved by the founder (GitHub mobile app — one tap).
- **Environments:** `dev` (preview deployments per PR: Vercel previews for web; Railway PR environments or a shared staging worker), `staging` (real vendors, test numbers, fake billing), `production`. Separate provider accounts/projects and separate phone numbers for staging vs prod; separate API keys with spend limits.
- **Hosting:** Vercel (web/API/webhooks — mobile-friendly dashboard, previews, logs) + **Railway** for the always-on agent worker (chosen over Fly.io for dashboard-first, phone-friendly ops; agents can operate either) + **Neon Postgres** (branching databases pair naturally with preview envs; migrations via a migration tool run in CI with approval gate) + Upstash Redis if queueing needs emerge.
- **CI (GitHub Actions):** lint, typecheck, unit tests, contract tests against recorded vendor fixtures, then integration tests that run a synthetic call through staging (audio fixtures injected, latency/accuracy asserted). Deploy to staging on merge; **production deploy requires a GitHub Environment approval** — a push-notification tap on the founder's phone. Rollback = redeploy previous release (both Vercel and Railway support one-click); DB migrations must be backward-compatible one release back (rule in the handoff doc).
- **Secrets:** GitHub Environments secrets + platform env vars (Vercel/Railway) + Cursor Cloud Agent secrets for agent-run tests. Never in chat, code, or logs; a CI secret-scanner (gitleaks) gate; short-lived tokens where vendors support them; least-privilege API keys (e.g., Telnyx key scoped to voice only, no number-purchasing permission on the deploy key — number purchases go through a human-approved path only).
- **Monitoring:** Sentry (web+agent errors), Axiom or Better Stack (structured logs, phone-readable dashboards), UptimeRobot-class ping on webhook endpoints, a `#alerts` channel (Telegram/Slack) receiving deploys, error spikes, spend alerts, failed-call-rate alerts. Agents can read logs via CLI/API tokens (read-only) to self-debug.
- **Founder ops panel (small page in the app, phone-first):** today's calls, success/latency tiles, spend vs budget, kill switch, feature flags, approve-deploy deep link. Weekly automated quality report (benchmark run results, cost per minute, failure taxonomy) posted to the channel.
- **Incident handling:** runbooks in `docs/runbooks/` (one per failure class from §14); the on-call is the founder + a "diagnose" agent prompt template that ingests the runbook, logs, and recent deploys; anything requiring a paid resource change goes through the approval gate.

---

## 11. AI-agent execution strategy

**Roles → mechanics.** Do not create ten standing agents; use **sequential phases with role-scoped review passes**: one implementation agent per milestone task (fresh context, reads the handoff docs), then two review passes on each PR — (a) security/privacy checklist pass, (b) quality/test pass — by a *different model* than the implementer, plus a telephony/audio-specialist pass only on milestones touching the media path. DevOps, cost review, and UX review are checklists executed as review passes, not separate long-lived agents.

**Model routing (against models actually available in Cursor at research date ⚠ — availability and pricing change; re-verify at kickoff):**
- **Architecture, ADRs, media-path core, gnarly debugging, incident forensics:** a top-tier reasoning model (Claude Opus-class or GPT-5.6-class, high thinking).
- **Bulk implementation (UI, CRUD, webhooks, tests-first features):** **Grok 4.5 is a reasonable choice and the founder's default is acceptable**; Composer-class and GPT-5.x-codex-class are the comparators. Rather than trusting anyone's benchmark marketing, run a **calibration bake-off**: give two candidate models the same fully-specified Milestone-2 task in isolated worktrees; score on acceptance-test pass rate, review findings, and cost; pick the winner as the standing implementer. Evaluation criteria: plan adherence, tool use, test discipline, tendency to invent APIs (the classic cheap-model failure — mitigated by pinning provider doc excerpts into the task), and recovery behavior on failed deploys (require "read logs → hypothesize → smallest fix → re-run" loops in the task template).
- **Tests, docs, lint, small fixes:** cheapest fast tier.
- **Final milestone review:** an independent model that did not write the code.
- **Guardrail for all agents:** no agent ever holds credentials that can purchase numbers, raise spend limits, or delete production data; those actions live behind founder-approved paths only.

**Handoff format (files in `docs/`, written by the planning agent, consumed by implementers):**
- `docs/adr/ADR-00x-*.md` — decisions with alternatives + reversal conditions (telephony provider, framework, engine defaults, hosting…).
- `docs/milestones/M<x>.md` — goal, in/out of scope (**non-goals explicit**), exact acceptance tests (runnable commands + expected outputs), security constraints, cost constraints ("this milestone may not create resources costing >$X/mo"), known unknowns, links to the exact provider doc pages, rollback procedure.
- `docs/acceptance/` — the benchmark test sets (audio fixtures, number/name test corpus, expected-latency budgets).
- `docs/runbooks/` — per failure class.

---

## 12. Milestone plan (each = one or few PRs, agent-executable, with acceptance criteria)

- **M0 — Repo & rails.** Monorepo scaffold, CI, lint/test, preview deploys, secrets wiring, ops-panel stub. *Accept:* PR previews build; secret scanner green; founder can approve a deploy from phone.
- **M1 — Accounts & vendor sandboxes (human-heavy).** Founder creates GitHub/Vercel/Railway/Neon/LiveKit/Telnyx-or-Twilio/OpenAI/ElevenLabs/Azure accounts with billing caps (checklist §13). *Accept:* staging env vars present; test-credential smoke scripts pass.
- **M2 — Translation loop, no telephony.** Web page: speak Hebrew → hear French (and reverse), cascade pipeline, per-stage latency telemetry, benchmark harness + number/name corpus. **This is the calibration bake-off task.** *Accept:* EOS→TP P50 ≤2.5s on fixtures; number accuracy ≥90%; harness runs in CI.
- **M3 — One PSTN leg.** Agent joins room; outbound call to founder's own phone via trunk; echo test then one-direction translation. *Accept:* founder receives call, hears translated speech; call teardown clean; webhook signatures verified.
- **M4 — Full bridge.** Two legs, both directions, turn manager, barge-in, queue caps, intro announcement, DTMF, reconnection supervisors. *Accept:* scripted two-phone test call (founder + a helper) completes 15 min; failure-injection tests (kill STT ws mid-call, etc.) pass per the §14 matrix.
- **M5 — Product app.** Auth (SMS OTP), contacts, language pick, call screen with live captions/status/repeat, call history (metadata only), post-call rating. *Accept:* founder completes a real Hebrew→French call to a real recipient entirely from phone.
- **M6 — Hardening & cost controls.** Caps, allowlists, kill switch, fraud rules, alerting, load test (10 concurrent synthetic calls), long-call (60-min) stability, engine failover drills. *Accept:* beta targets (§6) on weekly benchmark; abuse tests blocked.
- **M7 — Billing & paid beta.** Stripe prepaid/subscriptions, minute metering, invoices; ToS/privacy (counsel gate); onboarding for ~10 beta users. *Accept:* a stranger can sign up, pay, call; margin dashboard live.
- **M8+ —** transcripts/summaries with consent machinery (legal-gated), additional pairs (he↔en first — largest demand, best model support; then fr↔en, ru↔he given demographics ⚠ validate demand), S2S engine adoption where benchmarks win, voice identity, business features.

---

## 13. Human involvement (the honest split)

**Cannot/should not be automated — founder checklist (phone-friendly, in order):**
1. Create accounts + accept ToS + enable billing with low caps (GitHub, Vercel, Railway, Neon, LiveKit, Telnyx/Twilio, OpenAI, Google AI, Azure, ElevenLabs, Stripe, Sentry).
2. Complete identity/KYC where asked.
3. Enter secrets into GitHub/Vercel/Railway/Cursor secret UIs (never chat).
4. Approve number purchases and provide KYC docs for the Israeli (and later French) numbers.
5. Verify his own phone numbers (OTP).
6. Set geographic calling permissions to IL+FR only.
7. Approve each production deploy.
8. Make and judge real test calls (he is the Hebrew/French quality oracle).
9. Commission legal review before public beta.
10. Final go/no-go.

**Realistic automation share:** ~85% of engineering work (code, tests, deploys, log-reading, benchmarks, docs) is agent-executable with this plan; ~15% is irreducibly human — dominated not by effort but by *elapsed time* (KYC reviews, number-compliance approvals can take days ⚠). Quality judgment on translations is genuinely human work at this stage and should be scheduled as short daily listening sessions during beta.

---

## 14. Security plan (mapped to requirements)

- Least-privilege vendor keys per environment; no key with number-purchase or account-admin scope in any agent or CI context; short-lived tokens where supported.
- Secrets only in platform secret stores; gitleaks in CI; log scrubbing middleware (E.164 numbers masked to `+972-52-***-**67`, no audio payloads in logs).
- All webhooks signature-verified (Telnyx/Twilio/Stripe) with replay-window nonce checks; WebSocket sessions authenticated by short-lived signed JWTs bound to call ID; authorization checks on every call-control action (user owns call).
- Rate limits on auth, call-start, verification endpoints.
- Encryption in transit everywhere; at rest via managed-DB encryption + KMS-held keys for any future recordings (per-tenant). Data retention controls per §8; secure deletion includes provider-side artifacts.
- Audit log (append-only table) for logins, calls, CLI selections, consent events, config changes, deletions.
- Abuse/ATO: SMS OTP + device binding, concurrent-session limits, alert on destination-pattern anomalies.
- Supply chain: lockfiles, Dependabot, `npm audit`/`pip-audit` gates, no unvetted deps in the media path.
- Agent safety: agents get staging creds only; the production deploy and any resource-creating IaC change require the founder's explicit approval gate; a documented "expensive action" list (buy number, raise cap, new vendor, region change) is hard-blocked for agents.
- Incident response runbook: kill switch → rotate affected keys → preserve audit logs → user notification duties per GDPR/Amendment 13 timelines (counsel input).

---

## 15. Failure behavior matrix (what each party experiences)

| Failure | Caller (app) hears/sees | Recipient hears | Call continues? | Retry/fallback |
|---|---|---|---|---|
| No answer / busy | Status + "try later / send SMS heads-up" | — | n/a | No auto-redial (spam risk); one-tap manual retry |
| Voicemail detected (AMD) | Prompt: leave translated message? | Translated voicemail if chosen | n/a | Optional feature; off by default at launch |
| STT/MT/TTS outage (one component) | Banner "interpreter reconnecting" | Soft hold tone after 5s, spoken notice at 15s in their language | Yes, legs stay up | Auto-failover to alternate vendor; supervisor retries ≤3 |
| Full AI outage | Clear failure message + apology credit | Spoken apology in their language, call ends politely | No | Incident alert; kill switch stops new calls |
| S2S session cap / timeout | Invisible if rotation works; else "reconnecting" | ≤2s gap | Yes | Planned rotation during silence; cascade fallback |
| Network blip on leg A (app) | App auto-reconnects; captions preserved | Brief silence + soft tick | Yes (30s grace) | Offer PSTN dial-out to caller's phone |
| Leg B drops | "Recipient disconnected — call back?" | — | No | One-tap redial |
| Both speak at once | Caption: "waiting for pause" | Earcon | Yes | Queue with cap=2, then polite prompt |
| Recipient interrupts playback | Caption: "they responded before finishing"; untranslated remainder shown as text | Playback ducks/stops | Yes | Remainder re-offerable via "resend" |
| Growing audio queue / long monologue | "Please pause to translate" prompt | Chunked delivery every ~15s of speech | Yes | Hard chunking at endpointing layer |
| Long silence (>45s both sides) | "Still there?" prompt | Same in their language | Yes; auto-end at 2 min silence | Prevents runaway billing |
| Unsupported language detected mid-call | Warning caption; translation best-effort or paused | Notice | Caller decides | Language pair locked pre-call to prevent most cases |
| CLI presentation fails (masked/blocked) | Pre-call warning next time; log | Sees "unknown/hidden" | Yes | Switch corridor to local-number mode; SMS heads-up option |
| Consent declined (future recording) | "Recording off; translation continues" | Confirmation in their language | Yes | Never retry consent in same call |
| Billing cap reached | 60s warning, then graceful wrap-up message both sides | Wrap-up in their language | Ends cleanly | Top-up deep link |
| Duplicate webhook / duplicate call initiation | Invisible | Invisible | — | Idempotency keys on call-create; webhook dedupe by event ID |
| Worker restart / deploy during call | ≤5s gap if agent handover works; else treated as AI outage | Same | Best effort | Drain-before-deploy: no deploys while calls active in v1 (call cap makes this practical) |

General degradation ladder: **full translation → alternate vendor → text-only captions to caller + apology to recipient → graceful call end with credit.** Never silent failure; every abnormal end produces a user-visible explanation and a structured incident record.

---

## 16. Known unknowns (top of the risk register)

1. Real-world Hebrew STT quality on 8kHz PSTN audio with Israeli/French accents — decides the cascade's viability margin (Spike #1, before M3 exit).
2. Israeli anti-spoofing regs' current enforcement status and whether Israeli virtual numbers via CPaaS originate "domestically" from carriers' viewpoint ⚠ (ask Telnyx/Twilio support directly — human task).
3. French local-number KYC without a French address — determines the France-side CLI story ⚠.
4. Gemini live-translate preview → GA timeline, pricing, SLA ⚠ (would change the engine default if strong).
5. Whether OpenAI adds Hebrew output to `gpt-realtime-translate` ⚠.
6. LiveKit Cloud per-minute economics at scale and EU region latency to Israeli PSTN ⚠ (Spike #5).
7. Apple/Samsung Hebrew timeline — competitive clock for the consumer wedge ⚠.
8. Real user tolerance of 2s consecutive-interpretation rhythm — only beta users answer this.

---

## Bottom line

Build the cascaded, framework-based, two-leg bridge for Hebrew↔French with local-number caller ID and no recording; validate the AI pipeline before touching telephony; keep every engine and voice behind an adapter; gate every expensive or legally-sensitive step on a one-tap founder approval; and spend premium-model budget only on the media path, architecture, and reviews. The plan above — ADRs, milestones M0–M8, spike list, targets, and constraint documents — is structured so a mid-cost implementation agent can execute it task-by-task with verifiable acceptance criteria at every step.

---

## Appendix: Key sources checked (17 July 2026)

- ARCEP — masking of unauthenticated French mobile CLIs on international calls, effective 1 Jan 2026 (arcep.fr; French National Numbering Plan update).
- Israel Ministry of Communications — draft regulations blocking international calls presenting Israeli CLIs (reported via Jerusalem Post; verify current status).
- Twilio — Verified Caller ID / OutgoingCallerIds API; Transit Caller ID sunset (June 2026); voice pricing pages for Israel and France; regulatory guidelines for FR/IL numbers.
- Telnyx — Voice API pricing ($0.002/min + trunk fees; media streaming $0.0035/min); Israel DID requirements.
- OpenAI — Realtime API docs: `gpt-realtime-2.1` / `-mini`, `gpt-realtime-translate` ($0.034/min, 70+ input langs, 13 output langs — no Hebrew output), `gpt-realtime-whisper` ($0.017/min); 60-min session cap.
- Google — Gemini Live API `gemini-3.5-live-translate-preview` (launched 9 June 2026; Hebrew in/out; preview, 15-min audio caps; SynthID watermarking).
- Microsoft — Azure Voice Live API language support (Hebrew via gpt-realtime models; Azure STT/TTS catalog).
- ElevenLabs — model/language docs: Eleven v3 (74 langs incl. Hebrew), Flash v2.5 (32 langs, no Hebrew, ~75ms, ~$0.06/min), Scribe v2 Realtime STT (~150ms, 90+ langs incl. Hebrew).
- Apple — iOS 26 feature availability: Live Translation (Phone/FaceTime) language list — no Hebrew.
- Samsung — Galaxy AI Live Translate supported languages — no Hebrew (community confirmation July 2026).
- Israel — Secret Monitoring Law 5739-1979 (one-party consent); Privacy Protection Law Amendment 13 (in force 14 Aug 2025).
- France — Code pénal art. 226-1 (recording consent); GDPR.
- EU AI Act — Article 50 transparency obligations applicable 2 Aug 2026; draft Commission guidelines (May 2026) and Code of Practice (June 2026); Omnibus transitional relief for machine-readable marking to 2 Dec 2026 (pending formal adoption).
