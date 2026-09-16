# SignLive Architecture Design

**Date:** 17 September 2026  
**Status:** Architecture approved for specification; implementation not yet authorised  
**Project:** SignLive / Innova SignBridge-compatible real-time signed-language translation engine  
**Initial target language:** Auslan  
**Design intent:** Apply the public architectural premises behind large-scale multilingual speech models and low-latency multimodal realtime systems to signed-language translation without treating signed language as word substitution.

## 1. Executive summary

SignLive is a proposed multimodal translation system that accepts live speech or text and produces continuous signed-language motion suitable for real-time avatar rendering. The first production language is Auslan, while every core interface is language-identified and designed for later extension to other signed languages.

The architecture deliberately avoids a naive `speech -> transcript -> English words -> dictionary clips` pipeline. Instead, it separates:

1. source capture and streaming,
2. semantic interpretation,
3. sign-language planning,
4. lexical and phonological grounding,
5. continuous motion generation,
6. avatar rendering,
7. human-centred quality evaluation.

The central architectural contract is **Sign IR**, a sign-native intermediate representation that describes communicative meaning, discourse state, spatial referents, lexical choices, manual articulation requirements and non-manual grammatical features. Sign IR is not a gloss transcript and is not tied to one renderer.

The central motion contract is **Sign Motion Tokens**, a learned discrete or hybrid representation of temporally aligned body, hand, head, gaze, mouth and facial-expression movement. A SignAvatars-style SMPL-X representation is the initial research target for motion generation because it exposes whole-body pose, both hands, jaw and facial-expression parameters. SignAvatars itself remains a research/evaluation dependency only unless its licence is separately shown to permit the intended use.

Signbank is used as a lexical and linguistic grounding source, not as the translation engine. The system records source provenance and licence state for every lexical and motion asset used in training, evaluation or runtime generation.

The first engineering milestone is a testable vertical slice:

`live English speech/text -> semantic stream -> Auslan Sign IR -> motion plan -> SMPL-X frames -> accessible avatar preview`

The first milestone does **not** claim production-quality automatic Auslan translation. Human comprehension and Deaf-community review are release gates.

---

## 2. Design principles

### 2.1 Signed language is language

The system must not assume that English word order, lexical boundaries or grammatical categories map directly onto Auslan. Translation is mediated through meaning and sign-language planning.

### 2.2 Manual and non-manual information are co-equal

Hands alone are insufficient. Facial expression, eyebrow movement, gaze, mouth actions, head movement, torso posture, signing-space placement and timing can carry linguistic information. These channels must be represented in Sign IR and in the motion layer.

### 2.3 Realtime means incremental, not sentence-at-a-time

The runtime accepts partial input, updates hypotheses incrementally and emits cancellable motion chunks. It keeps a tunable look-ahead window so low latency does not force premature grammatical commitments.

### 2.4 Communication access is safety-critical

The system must preserve typed text and captions alongside generated sign output. A failure to generate sign must degrade to accessible text rather than silence. In accessibility or emergency contexts, the engine must not infer cognition, consent or comprehension from signing speed, speech intelligibility, AAC latency, hesitation or motor performance.

### 2.5 The renderer cannot change meaning

Rendering is downstream of the authoritative linguistic and motion state. A renderer may interpolate geometry, apply camera transforms and style an avatar, but it may not invent signs, alter lexical choices, remove non-manual markers or mutate discourse referents.

### 2.6 Provenance travels with data

Every training example, lexical record, exemplar, motion sequence and generated evaluation item carries provenance and licence metadata. Research-only or non-commercial data is technically separable from production-eligible data.

### 2.7 Community evaluation outranks automatic metrics

Automatic metrics are useful for regression detection, but release readiness depends on comprehension, naturalness and linguistic validity judged by Deaf Auslan users and qualified linguistic reviewers.

---

## 3. Public architectural premises adapted from modern speech systems

SignLive adopts public design ideas rather than proprietary implementation details.

### 3.1 Large heterogeneous supervision

Whisper demonstrated the value of training on a large and diverse multilingual, multitask supervised corpus instead of optimising only for a narrow clean speech benchmark. SignLive applies the same principle to heterogeneous sign data while maintaining explicit dataset provenance and language identity.

Training tasks may eventually include:

- speech -> Sign IR,
- text -> Sign IR,
- Sign IR -> motion,
- gloss -> Sign IR,
- HamNoSys -> motion,
- motion -> Sign IR,
- sign video -> semantic representation,
- sign video -> motion representation,
- motion continuation,
- sign-language translation between text and motion domains.

### 3.2 Native streaming representations

Modern realtime voice systems stream media through persistent low-latency sessions rather than waiting for a complete recording and serially invoking unrelated components. SignLive adopts a persistent session model with incremental semantic state, incremental Sign IR revisions and cancellable motion output.

### 3.3 Interruption and revision

Realtime communication requires barge-in. Future motion that has not yet been displayed can be cancelled when a speaker changes course, corrects themselves or interrupts. Already rendered sign is immutable historical output and remains in the session transcript/audit state.

### 3.4 Multimodal context

A future SignLive session can accept text, audio, image and sign video context, but the first release only requires text and audio input. The architecture reserves modality-neutral session events from the beginning.

---

## 4. System context

```text
Speaker / typist / AAC user
          |
          v
+--------------------------+
| Realtime Session Gateway |
+------------+-------------+
             |
             v
+--------------------------+
| Source Encoders          |
| audio / text             |
+------------+-------------+
             |
             v
+--------------------------+
| Semantic Stream          |
| meaning + uncertainty    |
+------------+-------------+
             |
             v
+--------------------------+
| Auslan Planner           |
| Sign IR                  |
+------+-------------------+
       | lexical grounding
       v
+--------------------------+
| SignLex / Signbank       |
+------+-------------------+
       |
       v
+--------------------------+
| Motion Generator         |
| Sign Motion Tokens       |
+------+-------------------+
       |
       v
+--------------------------+
| SMPL-X Motion Decoder    |
+------+-------------------+
       |
       v
+--------------------------+
| Renderer Adapter         |
| WebGPU / Unity / other   |
+------+-------------------+
       |
       v
Avatar + captions + text
```

---

## 5. Repository architecture

A dedicated repository is recommended rather than embedding this project inside MapAble or DiverseSpeech.

```text
signlive/
  apps/
    web-demo/                 # accessible realtime demonstration client
    reviewer-console/         # human linguistic/comprehension review
  services/
    session-gateway/          # websocket/webrtc session orchestration
    speech-ingest/            # realtime speech transcript adapter
    sign-planner/             # semantic stream -> Sign IR
    lexicon-gateway/          # Signbank + local governed lexicon adapters
    motion-service/           # Sign IR -> motion tokens -> SMPL-X
  packages/
    contracts/                # versioned event, Sign IR, provenance schemas
    sign-ir/                  # validation and transformation utilities
    sign-lex/                 # normalized lexical model
    motion-contracts/         # token/frame interfaces
    accessibility-kit/        # captions, keyboard/switch/eye-gaze interaction patterns
    provenance/               # licence/provenance policy engine
    evals/                    # automatic + human-evaluation formats
  models/
    semantic/                 # model adapters/checkpoint metadata
    planner/
    motion/
    tokenizers/
  data/
    manifests/                # metadata only; no restricted datasets committed
    schemas/
    eval-fixtures/            # synthetic/licence-safe test fixtures
  research/
    signavatars/              # adapters/scripts, no redistributed restricted data
    how2sign/
    signbank/
  docs/
    architecture/
    data-governance/
    evaluation/
    community-governance/
  tests/
    contracts/
    planner/
    motion/
    streaming/
    provenance/
    accessibility/
```

The repository must not vendor SignAvatars annotations, How2Sign videos, Signbank databases or any third-party asset unless redistribution rights are explicitly verified.

---

## 6. Realtime session contract

### 6.1 Transport

The session gateway exposes a transport-neutral event protocol. WebRTC is preferred for low-latency browser audio. WebSocket is supported for server-to-server and test clients. The translation protocol does not depend on OpenAI-specific event names.

### 6.2 Canonical session events

```ts
type SessionEvent =
  | AudioChunkEvent
  | TextInputEvent
  | TranscriptDeltaEvent
  | SemanticDeltaEvent
  | SignIRDeltaEvent
  | MotionChunkEvent
  | RenderAckEvent
  | InterruptEvent
  | ClarificationEvent
  | ErrorEvent;
```

Every event contains:

```ts
interface EventEnvelope<T> {
  eventId: string;
  sessionId: string;
  sequence: number;
  createdAt: string;
  payload: T;
  provenance?: ProvenanceRef[];
}
```

`sequence` is monotonically increasing per session.

### 6.3 Revision rules

- Transcript hypotheses may revise uncommitted text.
- Semantic hypotheses may revise uncommitted meaning units.
- Sign IR may revise only units marked `provisional`.
- Motion chunks become immutable after `RenderAckEvent`.
- An interruption cancels all unacknowledged future motion chunks and requests a neutral transition from the renderer.

---

## 7. Sign IR v0.1

Sign IR is the authoritative linguistic interchange format between the sign-language planner and motion generation.

### 7.1 Design goals

Sign IR must be:

- sign-language identified,
- independent of English word order,
- explicit about uncertainty,
- explicit about spatial reference,
- able to encode non-manual grammar,
- compatible with lexical and phonological annotations,
- suitable for partial streaming updates,
- serializable as JSON,
- inspectable by human reviewers,
- extendable without invalidating existing consumers.

### 7.2 Top-level schema

```ts
interface SignIRDocument {
  schema: "signlive.sign-ir";
  version: "0.1";
  utteranceId: string;
  targetLanguage: SignLanguageTag;
  source: SourceContext;
  discourse: DiscourseState;
  units: SignUnit[];
  confidence: number;
  status: "provisional" | "committed";
  provenance: ProvenanceRef[];
}
```

`SignLanguageTag` initially supports `aus-AU` and requires an explicit region/dialect field when known rather than silently collapsing regional variants.

### 7.3 Source context

```ts
interface SourceContext {
  modality: "speech" | "text" | "aac";
  sourceLanguage: string;
  originalText?: string;
  transcriptConfidence?: number;
  timing?: { startMs: number; endMs?: number };
}
```

`originalText` is preserved for review but is not treated as the grammatical template for signing.

### 7.4 Discourse state

```ts
interface DiscourseState {
  referents: SpatialReferent[];
  topic?: SemanticRef;
  perspective?: "speaker" | "addressee" | "reported";
  register: "neutral" | "formal" | "informal";
}

interface SpatialReferent {
  id: string;
  semanticType: "person" | "place" | "object" | "group" | "abstract";
  locus?: { x: number; y: number; z: number };
  active: boolean;
}
```

Coordinates are normalized signing-space coordinates, not renderer-world coordinates.

### 7.5 Sign unit

```ts
interface SignUnit {
  id: string;
  semantic: SemanticFrame;
  lexical?: LexicalChoice;
  manual?: ManualPlan;
  nonManual?: NonManualPlan;
  spatial?: SpatialPlan;
  timing?: TimingPlan;
  fingerspelling?: FingerspellingPlan;
  alternatives?: AlternativeChoice[];
  confidence: number;
  status: "provisional" | "committed";
}
```

A unit may represent a lexical sign, classifier construction, pointing construction, fingerspelling sequence, discourse marker or multi-sign expression.

### 7.6 Semantic frame

```ts
interface SemanticFrame {
  predicate: string;
  roles: Record<string, SemanticRef>;
  tenseAspect?: string[];
  polarity?: "positive" | "negative";
  modality?: string[];
  informationStructure?: {
    topic?: boolean;
    focus?: boolean;
    contrast?: boolean;
  };
}
```

The semantic frame is intentionally language-neutral enough to be generated from speech/text while allowing the Auslan planner to choose language-specific expression.

### 7.7 Lexical choice

```ts
interface LexicalChoice {
  lexemeId: string;
  dataset: string;
  glosses: string[];
  variant?: string;
  region?: string;
  phonology?: PhonologyRef;
  sourceProvenance: ProvenanceRef[];
}
```

`lexemeId` must be stable inside the normalized SignLex layer even if an upstream Signbank identifier changes.

### 7.8 Manual plan

```ts
interface ManualPlan {
  dominantHand: "left" | "right" | "both" | "context";
  handshape?: string[];
  orientation?: string[];
  location?: string[];
  movement?: string[];
  contact?: string[];
  symmetry?: "symmetric" | "asymmetric" | "none";
}
```

Values may be lexical references, normalized phonological labels or notation-system references. Sign IR does not require one phonological notation standard.

### 7.9 Non-manual plan

```ts
interface NonManualPlan {
  eyebrows?: string[];
  eyes?: string[];
  gaze?: string[];
  head?: string[];
  mouth?: string[];
  cheeks?: string[];
  torso?: string[];
  scopeUnitIds?: string[];
}
```

Non-manual markers may scope over multiple sign units.

### 7.10 Spatial plan

```ts
interface SpatialPlan {
  sourceLocus?: string;
  targetLocus?: string;
  path?: string;
  classifier?: string;
  indexing?: string[];
}
```

### 7.11 Fingerspelling

```ts
interface FingerspellingPlan {
  text: string;
  alphabet: "auslan";
  reason: "proper-noun" | "unknown-lexeme" | "explicit-request" | "technical-term";
}
```

Fingerspelling is a controlled fallback, not the default representation for unknown translation.

---

## 8. SignLex normalized lexical layer

SignLex mediates between Sign IR and upstream lexical systems such as Global Signbank or an Auslan-specific governed dataset.

```ts
interface SignLexEntry {
  id: string;
  language: string;
  dataset: string;
  upstreamId?: string;
  glosses: string[];
  spokenLanguageEquivalents: Record<string, string[]>;
  region?: string[];
  register?: string[];
  manualFeatures?: ManualPlan;
  nonManualFeatures?: NonManualPlan;
  hamnosys?: string;
  signwriting?: string;
  media: MediaRef[];
  motion: MotionRef[];
  provenance: ProvenanceRef[];
}
```

SignLex performs lexical retrieval and validation. It does not decide sentence grammar.

### 8.1 Retrieval policy

The planner may query SignLex by:

- semantic concept,
- lexical gloss,
- region,
- register,
- known phonology,
- provenance/usage permission.

The production planner must exclude lexical assets that fail the runtime-use provenance policy.

---

## 9. Motion representation

### 9.1 Why SMPL-X initially

A SignAvatars-style SMPL-X representation provides a practical research target because it jointly represents root/body pose, both hands, jaw and facial expression. This makes it possible to preserve manual and non-manual channels through one time-aligned frame sequence.

The engine must not assume that SignAvatars data itself can be used commercially. The representation concept and compatible open implementations may be used independently of restricted annotations.

### 9.2 Canonical motion frame

```ts
interface SignMotionFrame {
  tMs: number;
  rootPose: number[];
  bodyPose: number[];
  leftHandPose: number[];
  rightHandPose: number[];
  jawPose: number[];
  expression: number[];
  gaze?: number[];
  confidence: MotionConfidence;
}
```

The initial SMPL-X adapter maps compatible model output into the dimensional structure required by the selected SMPL-X implementation. The canonical contract keeps named channels so later body models can be substituted.

### 9.3 Sign Motion Tokens

The research model learns discrete or hybrid latent codes over aligned channel groups:

```ts
interface SignMotionToken {
  step: number;
  body: number[];
  leftHand: number[];
  rightHand: number[];
  face: number[];
  headGaze: number[];
  durationMs: number;
}
```

The arrays allow one or more codebook IDs per channel. A joint temporal transformer predicts the channel bundle at each motion step.

### 9.4 Tokenizer objectives

The tokenizer must optimize more than reconstruction loss. Evaluation includes:

- handshape reconstruction,
- hand trajectory reconstruction,
- orientation preservation,
- facial/non-manual reconstruction,
- timing and co-articulation,
- temporal smoothness without over-smoothing linguistically meaningful transitions.

### 9.5 Motion chunk contract

```ts
interface MotionChunk {
  chunkId: string;
  utteranceId: string;
  startMs: number;
  endMs: number;
  frames: SignMotionFrame[];
  sourceUnitIds: string[];
  status: "provisional" | "committed";
  cancellable: boolean;
}
```

A chunk becomes non-cancellable after renderer acknowledgement.

---

## 10. Model decomposition

### 10.1 Phase-one model family

**SignLive-Semantic**  
Input: speech transcript deltas or text.  
Output: semantic frames with uncertainty and discourse entities.

**SignLive-Auslan**  
Input: semantic frames + discourse state + SignLex retrieval.  
Output: Sign IR.

**SignLive-Motion**  
Input: committed/provisional Sign IR window.  
Output: Sign Motion Tokens.

**SignLive-MotionDecoder**  
Input: Sign Motion Tokens.  
Output: canonical motion frames / SMPL-X adapter output.

### 10.2 Future unified model

A future SignLive-Omni model may jointly learn speech, text, sign video and motion representations. The first release deliberately keeps semantic/planner/motion boundaries because they enable inspection, targeted evaluation, dataset isolation and replacement of weak subsystems.

---

## 11. Streaming algorithm

### 11.1 Input buffering

The runtime maintains four windows:

1. **audio window** for realtime transcription,
2. **semantic hypothesis window** for incomplete meaning,
3. **linguistic look-ahead window** before committing Sign IR,
4. **motion buffer** large enough to keep rendering continuous.

### 11.2 Commit policy

The planner commits a Sign IR span when at least one of the following applies:

- semantic boundary confidence exceeds a configured threshold,
- punctuation/prosodic evidence indicates a stable clause boundary,
- latency budget requires progress and the planner can produce a reversible continuation,
- explicit user turn completion is received.

The planner never treats low transcript confidence as permission to fabricate a lexical choice.

### 11.3 Barge-in

On interruption:

1. mark all unacknowledged motion chunks cancelled,
2. send `InterruptEvent` to renderer,
3. renderer moves through a short linguistically neutral transition,
4. preserve acknowledged historical output,
5. resume planning from the updated semantic stream.

### 11.4 Latency policy

Latency is surfaced as a measurable quality dimension rather than hidden. The system exposes modes such as:

- `responsive`: shorter look-ahead, more provisional motion,
- `balanced`: default,
- `accuracy-first`: longer linguistic look-ahead.

Mode names describe runtime behaviour and must not imply that lower latency is universally better communication.

---

## 12. Speech input adapter

The source speech adapter is replaceable.

Initial supported adapters may include:

- OpenAI GPT-Realtime-Whisper for low-latency transcript deltas,
- a locally hosted Whisper-family or other ASR model,
- a specialised disordered-speech recognizer when available.

The canonical adapter output is:

```ts
interface TranscriptDeltaEvent {
  type: "transcript.delta";
  text: string;
  startMs: number;
  endMs?: number;
  confidence?: number;
  final: boolean;
  sourceModel: string;
}
```

Speech recognition is not allowed to erase the original audio timing information needed for later correction and evaluation.

For people with dysarthric or otherwise non-standard speech, the runtime must support correction, text/AAC override and user-specific ASR adapters without treating recognition errors as user errors.

---

## 13. Renderer contract

The renderer is replaceable and is not authoritative for language.

```ts
interface RendererAdapter {
  loadAvatar(config: AvatarConfig): Promise<void>;
  enqueue(chunk: MotionChunk): Promise<void>;
  cancelAfter(chunkId: string): Promise<void>;
  transitionToNeutral(durationMs: number): Promise<void>;
  setPlaybackRate(rate: number): Promise<void>;
}
```

Candidate implementations include:

- browser WebGPU/WebGL avatar,
- Unity,
- Unreal/MetaHuman-compatible adapter,
- research neural renderer.

The first engineering milestone should use the simplest renderer that faithfully displays hands, face and body and can acknowledge rendered chunks.

### 13.1 Accessibility controls

The client must provide:

- captions and source text visible with sign output,
- pause/resume,
- replay previous signed phrase,
- adjustable signing speed without dropping frames,
- full keyboard operation,
- switch-compatible controls,
- eye-gaze compatible large targets,
- reduced-motion UI mode that does not reduce linguistic avatar motion,
- high-contrast UI,
- explicit clarification and correction controls.

Reduced-motion preference applies to decorative interface animation, not to grammatical sign movement.

---

## 14. Data architecture and provenance

### 14.1 Provenance object

```ts
interface ProvenanceRef {
  id: string;
  source: string;
  sourceUrl?: string;
  licence: string;
  permittedUses: ("research" | "evaluation" | "training" | "commercial-training" | "runtime")[];
  consentBasis?: string;
  attribution?: string;
  restrictions?: string[];
  verifiedAt: string;
}
```

### 14.2 Data classes

**Research-only**  
May be used to test hypotheses or benchmark research models when licence permits. It cannot flow into production training unless separately cleared.

**Evaluation-only**  
May be used for measurement but not model training.

**Production-training eligible**  
Requires verified rights for the intended training and deployment, plus consent/governance requirements.

**Runtime-eligible**  
May be served or rendered in deployed products.

### 14.3 Candidate research sources

The design can accommodate:

- SignAvatars motion annotations,
- How2Sign and derived landmark datasets,
- WLASL isolated-sign material,
- PHOENIX continuous signing datasets,
- Signbank lexical resources,
- future Auslan-specific community-governed corpora.

Each source must be individually reviewed before use. Dataset presence on GitHub or Hugging Face does not establish production rights.

### 14.4 Production Auslan corpus strategy

A production model should increasingly depend on a purpose-built, consented and governed Auslan corpus with:

- diverse Deaf signers,
- regional variation,
- natural discourse rather than only isolated dictionary signs,
- manual and non-manual capture,
- text/semantic annotations,
- explicit training and deployment consent,
- compensation and withdrawal/governance arrangements defined before collection.

---

## 15. Hugging Face integration

Hugging Face is used as an experiment and model lifecycle surface, not as the architectural centre of the runtime.

Recommended uses:

- private model repositories for research checkpoints,
- gated datasets where licensing permits,
- model cards documenting language, limitations and training provenance,
- dataset cards documenting rights and collection conditions,
- evaluation artifacts,
- optional GPU training jobs,
- reproducible research demonstrations.

No model may be marked production-ready solely because it has strong automated Hugging Face benchmark results.

---

## 16. Signbank integration

The Signbank adapter treats upstream Signbank installations as lexical sources.

The adapter must:

1. keep upstream dataset identity,
2. normalize stable SignLex IDs,
3. preserve variants and region labels,
4. preserve available phonological annotations,
5. preserve media provenance and licence terms,
6. never assume a gloss is a one-to-one translation equivalent,
7. expose retrieval confidence and ambiguity.

Caching is permitted only where upstream terms allow it.

---

## 17. SignAvatars research integration

The SignAvatars repository describes a large-scale 3D sign-motion corpus with SMPL-X annotations and separate natural-language, word and HamNoSys-conditioned subsets. Its published data-access terms indicate a non-commercial research orientation and do not redistribute original RGB video.

Accordingly:

- SignAvatars may seed motion-token experiments,
- its SMPL-X channel decomposition informs the canonical motion adapter,
- its data must live behind a `research-only` provenance gate unless rights change,
- production models must be able to train without SignAvatars data,
- no application runtime should depend on downloading or serving SignAvatars source data.

---

## 18. Evaluation architecture

### 18.1 Automatic evaluation

Automatic tests include:

- semantic preservation against controlled fixtures,
- Sign IR schema validity,
- lexical provenance eligibility,
- temporal alignment,
- motion reconstruction error,
- hand trajectory and handshape proxies,
- non-manual reconstruction proxies,
- motion jerk/smoothness metrics,
- time-to-first-sign,
- end-to-end latency,
- interruption cancellation latency,
- buffer underruns.

### 18.2 Human linguistic evaluation

A structured review interface captures:

- meaning preserved: yes / partly / no,
- grammatical acceptability,
- lexical appropriateness,
- spatial-reference correctness,
- fingerspelling appropriateness,
- non-manual correctness,
- naturalness/co-articulation,
- regional/dialect fit,
- uncertainty or disagreement notes.

### 18.3 Comprehension evaluation

For held-out signed outputs, Deaf participants answer meaning/comprehension questions without seeing the source text first. Comprehension is measured independently of reviewer familiarity with the intended translation.

### 18.4 Release gate

A candidate release cannot be called an automatic Auslan translator unless community-led testing shows that intended users can reliably understand outputs across the defined target domain. Before that threshold, the system must be labelled research, assistive drafting or preview technology as appropriate.

---

## 19. Community governance

An **Auslan Model Council** should have formal authority over language-quality and community-harm release gates.

Membership should include:

- Deaf Auslan users,
- Auslan linguists,
- interpreters or interpreter educators,
- DeafBlind representation,
- regional/community signers,
- accessibility specialists,
- ML researchers,
- people who use multimodal communication or AAC where relevant.

The council reviews:

- corpus collection policy,
- consent and compensation practices,
- dialect representation,
- terminology and fingerspelling policy,
- avatar acceptability,
- human-evaluation design,
- known failure modes,
- deployment claims.

The council does not replace individual consent for data collection.

---

## 20. Safety and failure handling

### 20.1 Failure states

The runtime must distinguish:

- ASR uncertainty,
- semantic ambiguity,
- lexical ambiguity,
- unsupported construction,
- motion-generation failure,
- renderer failure,
- provenance-policy rejection,
- network interruption.

### 20.2 Degradation order

When sign generation cannot be trusted:

1. preserve source audio/session continuity,
2. display live or corrected text/captions,
3. state that signed output is unavailable or uncertain,
4. offer replay/rephrase/clarification,
5. never silently substitute an unrelated sign.

### 20.3 High-stakes domains

The first release must not present machine-generated signing as a replacement for accredited interpreters in legal, medical, emergency or other high-stakes contexts. Research in those contexts may proceed with explicit human supervision and clear labelling.

---

## 21. Privacy and data handling

Live audio, corrected transcripts, sign-video input and biometric motion data may be sensitive. The runtime should default to data minimisation:

- no training from user sessions by default,
- explicit consent for research contribution,
- short-lived streaming buffers,
- separate identifiers from research data,
- configurable local preprocessing,
- access-controlled review datasets,
- deletion/withdrawal workflows for community-contributed corpora.

Biometric pose or face data must not be repurposed for identity inference without a separate explicit lawful and ethical basis.

---

## 22. First engineering milestone: SignLive Vertical Slice v0.1

### 22.1 Scope

Build a local/development system that accepts typed English and live English speech, produces inspectable Auslan Sign IR, generates continuous research motion, and renders an avatar preview with captions.

### 22.2 Included

- transport-neutral session event schema,
- WebSocket reference transport,
- one realtime speech transcript adapter,
- typed-text input,
- Sign IR v0.1 validation,
- mock + adapter-based SignLex service,
- rule/model hybrid Auslan planner prototype,
- research motion adapter using licence-safe fixtures first,
- SMPL-X-compatible motion contract,
- accessible browser renderer/visualizer,
- interruption/cancellation,
- provenance-policy checks,
- automated contract/streaming tests,
- reviewer console for human scoring.

### 22.3 Excluded from v0.1

- production deployment claims,
- sign-video recognition,
- multilingual signed-language output,
- photorealistic neural avatars,
- automated medical/legal/emergency interpreting,
- training on user sessions,
- commercial use of research-only datasets,
- replacing human interpreters.

### 22.4 Success criteria

The vertical slice succeeds when:

1. a user can speak or type an utterance and receive time-aligned captions plus continuous avatar motion;
2. the intermediate Sign IR is inspectable and version-valid;
3. lexical assets used by the planner expose provenance;
4. research-only assets are blocked from production-labelled configurations;
5. interruption cancels future unrendered motion without corrupting session state;
6. hands, face and body are represented through the full motion path;
7. keyboard, switch-friendly and eye-gaze-friendly controls work without mouse-only interactions;
8. reviewers can score semantic fidelity, grammar, non-manuals and comprehension;
9. the product UI describes the output as experimental until human evaluation supports stronger claims.

---

## 23. Development stages beyond v0.1

### Stage A — deterministic contracts

Implement schemas, streaming state machine, provenance gates, synthetic fixtures and renderer contract before training a custom foundation model.

### Stage B — lexical and planner research

Connect governed lexical sources and compare rule-assisted, retrieval-augmented and fine-tuned planning approaches while keeping Sign IR stable.

### Stage C — motion tokenizer

Train and evaluate a channel-aware motion tokenizer on research-eligible data. Establish reconstruction and human-readability baselines.

### Stage D — SignLive-Motion

Train conditional motion generation from Sign IR to motion tokens. Add streaming chunk prediction and cancellation-safe generation.

### Stage E — Auslan corpus programme

Collect a community-governed production-eligible Auslan corpus with explicit consent and rights suitable for model training and deployment.

### Stage F — end-to-end distillation

Where evidence supports it, distil or jointly train semantic, linguistic and motion components while preserving Sign IR for audit, correction and evaluation.

### Stage G — bidirectional signing

Add sign-video input, sign understanding and sign-to-text/speech translation as a separate validated subsystem.

---

## 24. Key architectural decisions

1. **Auslan is the first production language; architecture remains multilingual.**
2. **Sign IR is the stable semantic/linguistic contract.**
3. **Gloss is optional supervision/debug output, not the authoritative representation.**
4. **Manual and non-manual channels remain explicit through motion generation.**
5. **Signbank is lexical grounding, not sentence translation.**
6. **SignAvatars is a research source and representation reference, not a production runtime dependency.**
7. **Renderer is replaceable and cannot change linguistic meaning.**
8. **Streaming revisions are allowed only before linguistic/motion commitment.**
9. **Every data item has provenance and permitted-use metadata.**
10. **Human Deaf-user comprehension is a release gate.**
11. **Captions/text remain available whenever sign generation fails.**
12. **Communication latency or motor performance is never treated as a proxy for cognition or consent.**

---

## 25. Evidence and reference sources

- OpenAI, **Introducing Whisper** — multilingual/multitask supervised speech-learning premise and 680,000-hour training description: https://openai.com/index/whisper/
- OpenAI API, **Realtime API / GPT-Realtime models** — persistent low-latency audio/text realtime interaction, WebRTC/WebSocket/SIP, interruption-oriented architecture: https://platform.openai.com/docs/api-reference/realtime
- OpenAI, **Advancing voice intelligence with new models in the API** — 2026 GPT-Realtime-2, GPT-Realtime-Translate and GPT-Realtime-Whisper product direction: https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/
- Zhengdi Yu et al., **SignAvatars** repository/project — large-scale SMPL-X sign-motion annotations, language/word/HamNoSys conditioning, research data-access terms: https://github.com/ZhengdiYu/SignAvatars and https://signavatars.github.io/
- Signbank, **Global-signbank** — lexical database software and sign-language dataset infrastructure: https://github.com/Signbank/Global-signbank
- How2Sign — continuous ASL multimodal corpus; public mirrors and derived datasets commonly carry non-commercial restrictions that must be preserved: https://how2sign.github.io/
- Hugging Face sign-language ecosystem examples used for discovery, not automatically approved for production training: https://huggingface.co/datasets?other=sign-language and https://huggingface.co/models?other=sign-language-translation

---

## 26. Approval boundary

This document specifies architecture only. No production implementation, dataset ingestion, fine-tuning, external publishing or claims of Auslan translation quality are authorised by this specification alone.

The next step after review is a TDD implementation plan for **Vertical Slice v0.1**, decomposed into independently testable tasks and preserving the interfaces and constraints defined here.
