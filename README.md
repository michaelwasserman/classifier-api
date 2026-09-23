# Explainer: Web Classification, Decision & Ranking API

This living document explores a web platform primitive (`window.Classifier`) for **fast, type-safe, on-device classification, decision-making, and ranking**. We aim to understand the problems faced by web users and developers when building intelligent, privacy-preserving user experiences, and how recent breakthroughs in non-autoregressive decision encoders and parallel diffusion architectures can solve them without the pitfalls of other approaches.

- **Proponents:** Google Chrome Built-in AI Team
- **Participate:** [Discussion forum & issue tracker](https://github.com/explainers-by-googlers/classifier-api/issues)

---

## 1. The Problem

Web applications often need to make semantic decisions over unstructured content: *What is this page or message about? Which action matches the user's intent? Is this input safe? Which items are most relevant?*

Today, developers face an unappealing trade-off:
1. **Brittle Heuristics:** Regexes and keyword rules are fast and local, but fail on nuance, phrasing variations, and multilingual text.
2. **Cloud AI APIs:** Sending DOM text, drafts, or user queries to remote servers compromises user privacy, adds network latency (hundreds of milliseconds to seconds), and incurs ongoing per-request infrastructure costs.
3. **Generative On-Device LLMs (e.g., Prompt API):** Autoregressive text generation (token-by-token sampling) is overkill for structured decisions. It consumes nontrivial device resources (RAM/VRAM, compute, battery), takes seconds to generate tokens, requires bespoke client-supplied constraints (to avoid hallucinations, refusals, or format parsing errors), and lacks calibrated confidence estimates.

### Why Now? Breakthroughs in Parallel Non-Autoregressive Models

Recent machine learning advances demonstrate that giving up open-ended string generation unlocks massive gains in speed, reliability, and efficiency. This is exemplified by "System One" decision models (such as [Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev), [Kev](https://github.com/jaredpalmer/kev), [Laya](https://github.com/NandhaKishorM/laya), and [Open-Jev](https://github.com/nico-martin/open-jev)) and discrete diffusion / parallel canvas decoding architectures:

- **Parallel, Single-Pass Execution:** Instead of generating text token-by-token, these architectures evaluate an input state against multiple independent questions and option sets in **a single forward pass**—either via a bidirectional encoder with a learned decision head or via a 1-step bidirectional diffusion read over designated decision slots.
- **Zero Hallucinations & Guaranteed Type Safety:** Because the runtime scores caller-supplied options directly at designated decision slots rather than generating unconstrained characters, it **cannot hallucinate out-of-schema values or syntax errors**.
- **Calibrated Probabilities & Confidence:** Using Reinforcement Learning for Calibrated Decisions (RLCD) and post-hoc calibration, every decision includes a calibrated probability distribution, expected ordinal score ($E[S]$), and confidence score—enabling software to branch reliably when confident, and defer when uncertain.
- **Tiny On-Device Footprint:** Compact models (300M–800M parameters, ~300–400 MB quantized) achieve high accuracy on decision tasks in 10–50 milliseconds, making them viable across a broad spectrum of consumer CPUs, GPUs, and NPUs.

---

## 2. Core On-Device Use Cases

By keeping inference local, fast, and probabilistic, web developers can build responsive features that respect user privacy and reduce cognitive friction:

### Highlighted Use Cases

1. **Smart Client-Side Routing & Intent Triage**
   A support portal, command palette, or multi-agent web app immediately routes a user's draft or query to the right workflow, department, or local tool—while simultaneously detecting urgency and sentiment—before a single byte leaves the browser.
2. **Adaptive UI, Personalization & Accessibility**
   Web apps categorize the current content (products, articles) and available offerings (other products, articles, or features) in real time (using custom schemas or standard taxonomies like the IAB Content Taxonomy) to suggest relevant items or groupings, surface contextual reading aids, or simplify dense interfaces.
3. **Real-Time Input Verification & Local Guardrails**
   Productivity and forum apps evaluate user drafts or LLM outputs against policy rules on-keystroke (e.g., *"Does this contain personal contact info?"*, *"Is the tone constructive?"*, *"Does this bug report include reproduction steps?"*), offering instant nudges without server round-trips.
4. **Client-Side Ranking & Semantic Filtering**
   E-commerce sites, documentation portals, and local-first apps re-rank search results, candidate links, or UI actions against a user's natural-language goal directly on-device.

### Broader Task Taxonomy

This paradigm unifies eight foundational decision primitives under three question modalities (`binary`/`boolean`, `categorical`/`choice`, and `ordinal`/`score`):

| Task Primitive | Modality | Description | Web Example |
| :--- | :--- | :--- | :--- |
| **Classification** | `categorical` / `choice` | Assign content to mutually exclusive or multi-label categories. | Tagging an article into a developer schema or IAB taxonomy. |
| **Routing** | `categorical` / `choice` | Select the next handler, tool, or UI branch for an input. | Dispatching a customer query to `billing`, `bug`, or `feature_request`. |
| **Scoring** | `ordinal` / `score` | Rate content along an ordered, graduated scale with expected value $E[S]$. | Measuring draft readability, frustration level, or impact (`1` to `5`). |
| **Ranking** | `categorical` / `ordinal` | Order a list of candidates by relevance to application current state. | Re-ordering command palette actions or local search results. |
| **Detection** | `binary` / `boolean` | Identify the presence of specific risks, entities, or patterns. | Flagging phishing lures, toxic comments, or accidental PII exposure. |
| **Verification / Judging** | `binary` / `ordinal` | Evaluate whether an input satisfies explicit criteria or guardrails. | Checking if a review meets community guidelines before submission. |
| **Matching** | `categorical` / `choice` | Map unstructured input to structured application fields or filters. | Translating *"under $50 waterproof jacket"* into catalog filter options. |
| **Discrete Outcome Prediction** | `binary` / `boolean` | Predict calibrated `true`/`false` (`P(true)`) probabilities for assertions. | Estimating whether a user requires urgent escalation or a refund. |

---

## 3. Prospective Developer Pattern

We are exploring a declarative, **three-step developer pattern** on `window.Classifier` (exposed in `Window` and `Worker` contexts):
1. **Define a question schema** (`context` plus `questions` using `binary`/`boolean`, `categorical`/`choice`, or `ordinal`/`score` modalities) and initialize a session via `Classifier.create(schema)`.
2. **Pass the input state** (`DOMString` text, serialized DOM state, or JSON, plus optional per-call `context`) to `classifier.classify(input, options)`.
3. **Execute in parallel** and receive a calibrated result object mapping each question `id` to its decision (probabilities, top label, expected score, and confidence).

### Example 1: Parallel Triage, Scoring, and Verification (Cohesive Result Object)

```js
// 1. Define the structured question schema
const schema = {
  context: "Enterprise customer support ticket router.",
  questions: [
    {
      id: "is_urgent",
      type: "binary", // or "boolean"
      prompt: "Does this ticket require immediate incident response?"
    },
    {
      id: "category",
      type: "categorical", // or "choice"
      prompt: "Select the primary support department.",
      options: [
        { label: "bug", description: "Production crash or software defect" },
        { label: "billing", description: "Invoice or double-charge issue" },
        { label: "feature_request", description: "Enhancement request" }
      ]
    },
    {
      id: "severity",
      type: "ordinal", // or "score"
      prompt: "Rate the business impact from 1 (minimal) to 5 (critical).",
      options: [
        { label: "1", description: "Minimal impact" },
        { label: "2", description: "Low impact" },
        { label: "3", description: "Moderate impact" },
        { label: "4", description: "High impact" },
        { label: "5", description: "Critical production outage" }
      ]
    }
  ]
};

// Check model availability ("available", "downloadable", "downloading", "unavailable")
const status = await Classifier.availability(schema);

if (status === "available" || status === "downloadable") {
  const classifier = await Classifier.create(schema);

  // 2. Pass the input state to evaluate
  const input = document.querySelector("#ticket-input").value;
  // e.g., "Urgent: our production database pipeline crashes with a fatal segfault!"

  // 3. Execute in a single parallel pass and inspect the cohesive result object
  const result = await classifier.classify(input);

  console.log(result);
  // {
  //   is_urgent: {
  //     id: "is_urgent",
  //     label: "true",
  //     probability: 0.97, // Calibrated P(true)
  //     confidence: 0.94,
  //     probabilities: [{ label: "true", probability: 0.97 }, { label: "false", probability: 0.03 }]
  //   },
  //   category: {
  //     id: "category",
  //     label: "bug",
  //     confidence: 0.92,
  //     probabilities: [
  //       { label: "bug", probability: 0.94 },
  //       { label: "billing", probability: 0.03 },
  //       { label: "feature_request", probability: 0.03 }
  //     ]
  //   },
  //   severity: {
  //     id: "severity",
  //     label: "5",
  //     expectedScore: 4.78, // Continuous E[S] = sum(level_i * p_i)
  //     confidence: 0.89,
  //     probabilities: [...]
  //   }
  // }

  // Act autonomously when confident; fall back to human choice when ambiguous
  if (result.category.confidence > 0.85) {
    routeTicket(result.category.label, result.severity.expectedScore);
  } else {
    renderDepartmentPicker(result.category.probabilities);
  }

  classifier.destroy();
}
```

### Example 2: Client-Side Action Matching (Direct Destructuring by Question ID)

```js
const actionMatcher = await Classifier.create({
  context: "Document editor command palette",
  questions: [
    {
      id: "command",
      type: "choice",
      prompt: "Which command best fulfills the user's goal?",
      options: [
        { label: "export_pdf", description: "Download or save the document as a PDF" },
        { label: "share_link", description: "Invite collaborators or copy a sharing link" },
        { label: "archive_doc", description: "Move the document to trash or archive" }
      ]
    }
  ]
});

// Because the result object is keyed by question `id`, callers can destructure directly:
const { command } = await actionMatcher.classify("let my coworkers view this file");
// command -> { id: "command", label: "share_link", confidence: 0.93, probabilities: [...] }
```

---

## 4. User & API Client Requirements

To make on-device decision models dependable primitives for web software, the API and underlying browser runtime must address key functional and operational constraints:

- **Strict Output Conformance:** The API must never throw syntax parsing errors or return hallucinated out-of-vocabulary strings. Every `categorical`/`choice` decision `label` is strictly one of the caller's `options`; every `binary`/`boolean` or `ordinal`/`score` outputs bounded probabilities (`[0.0, 1.0]`) and well-defined expected values (`expectedScore`).
- **Calibrated Confidence:** Raw neural network logits are frequently overconfident. Runtimes must apply calibration (such as sequence-bucketed RLCD temperature scaling or multi-pass variance estimation) so that returned `probabilities` and `confidence` scores reliably reflect true certainty.
- **Single-Pass Parallelism & Question Isolation:** Multiple questions evaluated over the same input state share the encoded state representation while remaining isolated from each other. Packing multiple questions into a schema evaluates in a single pass rather than sequential model invocations.
- **Taxonomy Limits & Option Cardinality:**
  - *Option Count & Capacity:* Single-pass decision heads and diffusion canvases bound option cardinality per question (e.g., 2–16 options in compact heads or up to 26–255 options in larger backbones) and total questions per schema. Higher-cardinality ranking tasks can compose independent `score` evaluations or two-stage filtering.
  - *Distinctiveness & Ambiguity:* When caller-defined options overlap (e.g., `"fees"` vs. `"billing"`), lack sufficient `description` detail, or are tangential to the input, calibrated models spread probability mass across plausible options—yielding higher entropy and lower `confidence`.
  - *Order Invariance:* Training with option permutations and isolated slot readouts minimizes positional bias across option orderings.
- **Input Length & Context Bounds:** On-device encoders and diffusion backbones have bounded context windows (e.g., 64–1,024 tokens for ultra-fast checkpoints up to 8,192 tokens for larger backbones). The API should handle shared `context` and `input` bounds predictably.
- **Multi-Language Support:** Web content spans hundreds of languages and scripts. An English-only encoder can fail with dangerously high confidence on non-Latin scripts. The browser runtime must either pair a multilingual encoder (such as `mmBERT`) or route across language-appropriate checkpoints based on fast script/language detection.

---

## 5. Open Questions & Future Explorations

We invite community feedback on several open design questions:

1. **Standardized vs. Client-Defined Taxonomies:** Alongside custom developer-defined `questions` schemas, should `Classifier.create()` also accept shorthand identifiers for standardized, interoperable web taxonomies (e.g., `taxonomy: "iab-v3.1"`) that return stable taxonomy IDs?
2. **Result Ergonomics, Additive Confidence Diagnostics & Uncertainty Controls:**
   - *Keyed Record vs. Positional Array / Wrapper:* Having `classify()` resolve to a `record<DOMString, ClassifierDecision>` keyed by question `id` allows developers to either hold all decisions in a single cohesive `result` object (`result.category.label`) or destructure by name (`const { command } = await classifier.classify(...)`). Alternative shapes under consideration include also accepting `questions` as a keyed object in `Classifier.create()`, returning a positional `sequence<ClassifierDecision>` array, or returning a wrapper dictionary (`{ decisions }`) if top-level call metadata is needed.
   - *Additive Confidence & Sampling Controls:* The baseline API shape intentionally starts minimal, returning `probabilities`, `label`, `expectedScore`, and `confidence` on each `ClassifierDecision`. Because both input options (`ClassifierCreateOptions`, `ClassifierClassifyOptions`) and per-question decisions (`ClassifierDecision`) are extensible WebIDL dictionaries, richer uncertainty controls and diagnostics can be layered on as **strictly additive, non-breaking enhancements** if warranted. Should implementation-specific diffusion sampling controls and metrics (such as `maxSamples` / `samples`, `canvasWidth`, `drawsExecuted`, `agreement`, or `standardError`) be exposed as optional dictionary members for advanced callers, or kept internal in favor of model-agnostic effort hints (`classify(input, { effort })`) and uncertainty fields?
3. **Batching States (`classifyBatch`):** Evaluating *many questions over one input* is naturally parallelized in a single pass. For ranking or filtering *many inputs against one schema* (e.g., scoring 50 feed items or tabs), should `Classifier` expose a first-class `classifyBatch(inputs)` method?
4. **Multi-Modal Inputs (Image, Audio, DOM):** Could `classify()` eventually accept `ImageBitmap`, `HTMLCanvasElement`, or `AudioBuffer` inputs alongside text (e.g., classifying image accessibility traits, verifying visual UI state, or detecting spoken intent)?

---

## Appendix A: Alternatives Considered

| Approach | Pros | Cons / Why Insufficient Alone |
| :--- | :--- | :--- |
| **Autoregressive Prompt API (`LanguageModel`)** | Highly flexible; can generate arbitrary prose and explanations. | **10–100x slower and heavier** (requires multi-GB models); sequential token generation wastes energy when only a discrete decision or probability is needed; prone to uncalibrated probabilities. |
| **Server-Side Decision APIs** | Access to massive frontier models; zero client download size. | **Privacy & data sovereignty risk** (sensitive DOM/user input leaves the device); network latency blocks real-time UI; recurring API costs for developers. |
| **Bring-Your-Own Model (WebGPU / WASM + ONNX)** | Works today via libraries like Transformers.js (`open-jev`, `laya-ts`). | **Redundant downloads** (every origin downloads 300MB+ of weights); no cross-origin model caching or OS/browser-level hardware scheduling (LiteRT / XNNPACK / Accelerate). |
| **Fixed-Taxonomy-Only Classifier API** | Tiny footprint; stable numeric IDs for a single schema (e.g., IAB). | **Too narrow** for most app-specific routing, verification, scoring, and UI adaptation tasks where developers must define their own domain options. |

---

## Appendix B: Security, Privacy, Accessibility, i18n & Ecosystem Considerations

- **Privacy & Data Sovereignty:** All inference executes locally on the user's device. The API is strictly **stateless**: inputs are never transmitted over the network, stored on disk, or used to update model weights across calls. Access is gated by `SecureContext` and the `classifier` Permissions Policy (`PermissionsPolicyFeature::kClassifier`, defaulting to `Self`).
- **Security & Injection Resistance:** User-supplied `input` text is strictly separated from developer-supplied `questions` and `options` during schema compilation, preventing untrusted input from injecting fake options or corrupting adjacent decision slots. Hardware timing and floating-point precision variations must be bounded to prevent device fingerprinting.
- **Accessibility (a11y):** Fast local classification enables cognitive accessibility tools—such as automated tab grouping, reading-level estimation, and real-time form guidance—while calibrated `confidence` metrics allow UIs to avoid jarring automatic actions when user intent is uncertain.
- **Internationalization (i18n):** Browsers must ensure equitable behavior across languages and scripts. If a language is unsupported by the active on-device checkpoint, the API must explicitly report unavailability (`Classifier.availability()`) or route to a multilingual checkpoint rather than returning uncalibrated guesses.
- **Ecosystem Effects & Device Equitability:** Unlike 4B–8B generative LLMs that require high-end desktop GPUs, non-autoregressive decision architectures (such as 300M–420M ModernBERT/mmBERT with RLCD heads) run in tens of milliseconds via native CPU/NPU acceleration (LiteRT/XNNPACK/BLAS) and modest WebGPU hardware—bringing on-device AI to a wide global distribution of devices without exhausting RAM or battery.

---

## References & Prior Art

- **System One Models & Jev:** [Introducing System One Models & Jev (TypeSafe AI)](https://typesafe.ai/blog/introducing-system-one-models-and-jev) & [Nikolas Martin's Overview](https://www.linkedin.com/posts/nicodotdev_everyone-in-my-timeline-is-talking-about-ugcPost-7507689872749527040--K05/)
- **Open Implementations & Browser Runtimes:**
  - [Laya: Multilingual Non-Autoregressive System 1 Decision Engine](https://github.com/NandhaKishorM/laya)
  - [Open-Jev: Browser-Focused TypeScript Library for Typed Decisions](https://github.com/nico-martin/open-jev)
  - [Kev: Open Jev-like Decision Models on Qwen3/3.5](https://github.com/jaredpalmer/kev) & [kev-0.6b-ONNX on Hugging Face](https://huggingface.co/onnx-community/kev-0.6b-ONNX)
- **Web Standards & Explainer Guidelines:**
  - [W3C TAG: Writing Effective Explainers](https://www.w3.org/TR/explainer-explainer/)
  - [Web Platform Design Principles](https://www.w3.org/TR/design-principles/)
