# Explainer: Web Classification, Decision & Ranking API

This living document explores web platform gaps and potential solutions for fast, type-safe, on-device classification, decision-making, and ranking. We aim to outline the challenges web users and developers face when building responsive, privacy-preserving experiences, and explore how recent advances in non-autoregressive decision encoders and parallel diffusion architectures might address them alongside existing web APIs.

- **Proponents:** Google Chrome Built-in AI Team
- **Participate:** [Discussion forum & issue tracker](https://github.com/explainers-by-googlers/classifier-api/issues)

---

## 1. The Problem

Web applications frequently need to make structured semantic decisions over unstructured content: *What is this page or message about? Which action matches the user's intent? Does this input meet site guidelines? Which items are most relevant?*

Today, developers must choose among imperfect options for these tasks:
1. **Brittle Heuristics:** Regexes and keyword rules are fast and local, but fail on nuance, phrasing variations, and multilingual text.
2. **Cloud AI APIs:** Sending DOM text, drafts, or user queries to remote servers excels at heavy reasoning, but introduces privacy trade-offs, network latency (hundreds of milliseconds to seconds), and ongoing per-request infrastructure costs.
3. **Generative On-Device LLMs (e.g., Prompt API):** Autoregressive text generation (`LanguageModel`) is well-suited for open-ended prose, but is heavyweight for discrete decisions. Generating tokens sequentially consumes significant memory and battery, takes seconds, requires bespoke response constraints to avoid formatting errors, and does not natively produce calibrated confidence scores across options.

### Why Now? Breakthroughs in Parallel Non-Autoregressive Models

Recent machine learning advances show that trading open-ended text generation for structured option scoring unlocks substantial gains in speed, reliability, and efficiency. This is exemplified by "System One" decision models (such as [Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev), [Kev](https://github.com/jaredpalmer/kev), [Laya](https://github.com/NandhaKishorM/laya), and [Open-Jev](https://github.com/nico-martin/open-jev)) and discrete diffusion / parallel slot-decoding architectures:

- **Parallel, Single-Pass Execution:** Instead of generating text token-by-token, these architectures evaluate an input state against multiple independent questions and option sets in **a single forward pass**—either via a bidirectional encoder with a learned decision head or via a 1-step bidirectional diffusion read over designated decision slots.
- **Zero Hallucinations & Guaranteed Type Safety:** Because the runtime scores caller-supplied options directly at designated decision slots rather than generating unconstrained characters, it **cannot hallucinate out-of-schema values or syntax errors**.
- **Calibrated Probabilities & Confidence:** Using Reinforcement Learning for Calibrated Decisions (RLCD) and post-hoc calibration, every decision includes a calibrated probability distribution, expected ordinal score and confidence score, enabling software to branch reliably when confident, and defer when uncertain.
- **Small On-Device Footprint:** Compact models (300M–800M parameters, ~200–800 MB quantized) achieve high accuracy on structured decision tasks in 10–50 milliseconds, making them practical across a broad spectrum of consumer CPUs, GPUs, and NPUs.

---

## 2. Core On-Device Use Cases

By keeping inference local, fast, and probabilistic, web developers can build responsive features that respect user privacy and reduce cognitive friction:

### Highlighted Use Cases

1. **Smart Client-Side Routing & Intent Triage**
   A support portal, command palette, or multi-agent web app immediately routes a user's draft or query to the right workflow, department, or local tool, while simultaneously detecting urgency and sentiment, before a single byte leaves the browser.
2. **Adaptive UI, Personalization & Accessibility**
   Web apps categorize current content and available offerings (products, articles, or features) in real time, using custom schemas or standard taxonomies, to suggest relevant groupings, surface contextual reading aids, or simplify dense interfaces.
3. **Real-Time Input Verification & Local Guardrails**
   Productivity and forum apps evaluate user drafts or model outputs against policy rules on-keystroke (e.g., *"Does this contain personal contact info?"*, *"Is the tone constructive?"*, *"Does this bug report include reproduction steps?"*), offering instant nudges without server round-trips.
4. **Client-Side Ranking & Semantic Filtering**
   E-commerce sites, documentation portals, and local-first apps re-rank search results, candidate links, or UI actions against a user's natural-language goal directly on-device.

### Broader Task Taxonomy

This paradigm unifies eight foundational decision primitives under three question modalities (`binary`, `categorical`, and `ordinal`):

| Task Primitive | Modality | Description | Web Example |
| :--- | :--- | :--- | :--- |
| **Classification** | `categorical` | Assign content to mutually exclusive or multi-label categories. | Tagging an article into a developer schema or IAB taxonomy. |
| **Routing** | `categorical` | Select the next handler, tool, or UI branch for an input. | Dispatching a customer query to `billing`, `bug`, or `feature_request`. |
| **Scoring** | `ordinal` | Rate content along an ordered, graduated scale with expected value $E[S]$. | Measuring draft readability, frustration level, or impact (`1` to `5`). |
| **Ranking** | `categorical` / `ordinal` | Order a list of candidates by relevance to current application state. | Re-ordering command palette actions or local search results. |
| **Detection** | `binary` | Identify the presence of specific risks, entities, or patterns. | Flagging phishing lures, toxic comments, or accidental PII exposure. |
| **Verification / Judging** | `binary` / `ordinal` | Evaluate whether an input satisfies explicit criteria or guardrails. | Checking if a review meets community guidelines before submission. |
| **Matching** | `categorical` | Map unstructured input to structured application fields or filters. | Translating *"under $50 waterproof jacket"* into catalog filter options. |
| **Discrete Outcome Prediction** | `binary` | Predict calibrated `true`/`false` (`P(true)`) probabilities for assertions. | Estimating whether a user requires urgent escalation or a refund. |

---

## 3. Prospective Developer Pattern

We are exploring a three-step developer pattern on `window.Classifier` (in `Window` contexts):
1. **Define a question schema** (`context` plus `questions` using `binary`, `categorical`, or `ordinal` modalities—or aliases `boolean`, `choice`, `score`) and initialize a session via `Classifier.create(schema)`.
2. **Pass the input state** (`DOMString` text, serialized DOM state, or JSON, plus optional per-call `context`) to `classifier.classify(input, options)`.
3. **Execute in parallel** and receive a calibrated result object mapping each question `id` to its decision (probabilities, top label, expected score, and confidence).

### Example 1: Parallel Triage, Scoring, and Verification (Cohesive Result Object)

```js
// 1. Define the structured question schema
const schema = {
  context: "Enterprise customer support ticket router.",
  expectedInputs: [{ type: "text", languages: ["en"] }],
  questions: [
    {
      id: "is_urgent",
      type: "binary",
      prompt: "Does this ticket require immediate incident response?"
    },
    {
      id: "category",
      type: "categorical",
      prompt: "Select the primary support department.",
      options: [
        { label: "bug", description: "Production crash or software defect" },
        { label: "billing", description: "Invoice or double-charge issue" },
        { label: "feature_request", description: "Enhancement request" }
      ]
    },
    {
      id: "severity",
      type: "ordinal",
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
      type: "categorical",
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

To make on-device classification dependable for web applications, a solution must address several practical developer and user needs:

- **Guaranteed Schema Conformance:** Developers need to branch application logic on model outputs without defensive string parsing or handling unexpected out-of-vocabulary values. By scoring the caller's predefined `options` directly rather than generating free-form text, an API can ensure every returned `label` is strictly one of the supplied choices and every `probability` or `expectedScore` falls within known bounds.
- **Trustworthy Certainty Signals:** Applications need to know when a prediction is clear-cut enough to act on automatically versus when the input is ambiguous and warrants user confirmation. Because raw model outputs are often overconfident, the runtime should apply calibration (such as temperature scaling or variance estimation) and surface a normalized `confidence` score alongside option `probabilities`.
- **Fast, Independent Multi-Question Evaluation:** Real-world workflows often evaluate several facets of the same input at once (e.g., checking urgency, category, and severity together), and running separate inference passes per question would multiply latency and battery cost. Allowing callers to group multiple `questions` into a single schema lets the runtime encode the input once and evaluate all questions in parallel while keeping each question's scores isolated.
- **Predictable Question & Option Behavior:** Developers designing custom taxonomies need predictable behavior around option count, ambiguity, and ordering:
  - *Number of Questions & Choices:* Single-pass evaluation places practical bounds on how many questions and `options` fit in one schema (typically 2–16 options per question in compact models, or up to dozens in larger models); apps choosing from larger catalogs can narrow candidates in stages or score items individually.
  - *Clear, Distinct Options:* When options overlap in meaning (e.g., `"fees"` vs. `"billing"`), lack helpful `description` text, or don't match the input, a calibrated model should split probabilities across plausible choices and report lower `confidence` rather than guessing blindly.
  - *Order Independence:* Listing an option first or last in an `options` array should not bias its score.
- **Inspectable Input & Context Limits:** Because on-device models have finite token windows (e.g., 512 to 8,192 tokens), developers need to know how much capacity their schema consumes and how much room remains for per-call inputs, rather than suffering silent truncation. Similar to the [Prompt API](https://github.com/webmachinelearning/prompt-api), a session could expose its total `contextWindow`, a static `contextUsage` reflecting the compiled schema's footprint, and a `measureContextUsage(input, options)` helper—while rejecting with a `QuotaExceededError` in `create()` or `classify()` if the schema or input exceeds the available window.
- **Upfront Language & Modality Verification:** Web content spans hundreds of languages, and an English-only model can fail with high confidence on unsupported scripts if the app cannot verify coverage beforehand. Accepting an `expectedInputs` declaration (e.g., `[{ type: "text", languages: ["en", "es"] }]`) in `availability()` and `create()` would allow the browser to verify support upfront, select or download an appropriate multilingual checkpoint, or report `"unavailable"` when a requested language or modality cannot be served reliably.

---

## 5. Open Questions & Future Explorations

We invite community feedback on several open design questions:

1. **Result Ergonomics, Additive Confidence Diagnostics & Uncertainty Controls:**
   - *Keyed Record vs. Positional Array / Wrapper:* Having `classify()` resolve to a `record<DOMString, ClassifierDecision>` keyed by question `id` allows developers to either hold all decisions in a single cohesive `result` object (`result.category.label`) or destructure by name (`const { command } = await classifier.classify(...)`). Alternative shapes under consideration include also accepting `questions` as a keyed object in `Classifier.create()`, returning a positional `sequence<ClassifierDecision>` array, or returning a wrapper dictionary (`{ decisions }`) if top-level call metadata is needed.
   - *Additive Confidence & Sampling Controls:* The baseline API shape intentionally starts minimal, returning `probabilities`, `label`, `expectedScore`, and `confidence` on each `ClassifierDecision`. Because both input options and per-question decisions are extensible WebIDL dictionaries, richer uncertainty controls and diagnostics can be layered on as **strictly additive, non-breaking enhancements** if warranted. Should implementation-specific diffusion sampling controls and metrics (such as `maxSamples` / `samples`, `canvasWidth`, `drawsExecuted`, `agreement`, or `standardError`) be exposed as optional dictionary members for advanced callers, or kept internal in favor of model-agnostic effort hints (`classify(input, { effort })`) and uncertainty fields?
2. **Batching States (`classifyBatch`):** Evaluating *many questions over one input* is naturally parallelized in a single pass. For ranking or filtering *many inputs against one schema* (e.g., scoring 50 feed items or tabs), should `Classifier` expose a first-class `classifyBatch(inputs)` method?
3. **Multi-Modal `expectedInputs` vs. Simpler `expectedInputLanguages` / `expectedLanguage` Shorthand:** Adopting the Prompt API's `expectedInputs: [{ type: "text", languages: ["en"] }, { type: "image" }, { type: "audio" }]` shape prepares `Classifier` to accept `ImageBitmapSource` or `AudioBuffer` inputs alongside text without breaking session creation semantics. For common text-only classification tasks, should `Classifier.create()` and `Classifier.availability()` also accept a simpler `expectedInputLanguages: ["en"]` (as in `Summarizer` / `Writer`) or `expectedLanguage` shorthand option?
4. **Standardized vs. Client-Defined Taxonomies:** Alongside custom developer-defined `questions` schemas, should `Classifier.create()` also accept shorthand identifiers for standardized, interoperable web taxonomies (e.g., `taxonomy: "iab-v3.1"`) that return stable taxonomy IDs?
5. **Worker Context Support & Permissions Policy:** Gating `Classifier` behind a `classifier` Permissions Policy currently restricts the API to `Window` contexts, because Permissions Policy is not yet defined for workers. How should `Classifier` support background worker contexts (`DedicatedWorker`, `SharedWorker`, `ServiceWorker`) as proposals like [Permissions Policy for Workers](https://github.com/explainers-by-googlers/workers-permissions-policy) mature?

---

## Appendix A: Alternatives Considered

| Approach | Pros | Cons / Why Insufficient Alone |
| :--- | :--- | :--- |
| **Autoregressive Prompt API (`LanguageModel`)** | Highly flexible; well-suited for open-ended prose, summarization, and explanations. | **10–100x slower and heavier** (requires multi-GB models); sequential token generation wastes energy when only a discrete decision or probability is needed; lacks natively calibrated option probabilities. |
| **Server-Side Decision APIs** | Access to massive frontier models; zero client download size. | **Privacy & latency trade-offs:** Sensitive DOM or draft text leaves the device; network round-trips block real-time on-keystroke UI; incurs recurring server costs. |
| **Bring-Your-Own Model (WebGPU / WASM + ONNX)** | Works today via libraries like Transformers.js (`open-jev`, `laya-ts`); can pair with [Cross-Origin Storage (COS)](https://github.com/WICG/cross-origin-storage) to deduplicate identical weight files across origins. | **Fragmented downloads & storage overhead:** Even with [Cross-Origin Storage](https://github.com/WICG/cross-origin-storage), users still face redundant 300MB+ downloads and disk usage unless the web ecosystem converges on a tiny, standardized set of exact model checkpoints and quantizations; also lacks browser-managed OS/hardware scheduling. |
| **Fixed-Taxonomy-Only Classifier API** | Tiny footprint; stable numeric IDs for a single schema (e.g., IAB). | **Too narrow** for most app-specific routing, verification, scoring, and UI adaptation tasks where developers must define their own domain options. |

---

## Appendix B: Security, Privacy, Accessibility, i18n & Ecosystem Considerations

- **Privacy & Data Sovereignty:** All inference executes locally on the user's device. The API is strictly stateless: inputs are never transmitted over the network, stored on disk, or used to update model weights across calls. Access in `Window` contexts is gated by `SecureContext` and the `classifier` Permissions Policy (defaulting to `self`).
- **Security & Injection Resistance:** User-supplied `input` text is strictly separated from developer-supplied `questions` and `options` during schema compilation, preventing untrusted input from injecting fake options or corrupting adjacent decision slots. Hardware timing and floating-point precision variations must be bounded to prevent device fingerprinting.
- **Accessibility (a11y):** While the API has no built-in UI, websites can use fast local classification to make their content and controls much easier to navigate, such as matching everyday phrasing or voice commands to site actions (even when a user doesn't know the exact button label), surfacing the most likely next steps so keyboard or switch users don't have to tab through dozens of elements, and flagging dense text to offer simpler summaries or form guidance.
- **Internationalization (i18n):** Browsers must ensure equitable behavior across languages and scripts via `expectedInputs` (e.g., `[{ type: "text", languages: [...] }]`) in `Classifier.availability()` and `Classifier.create()`. If a requested language is unsupported by the active on-device checkpoint, the API must explicitly report `"unavailable"` or route to a multilingual checkpoint rather than returning uncalibrated guesses.
- **Ecosystem Effects & Device Equitability:** Non-autoregressive decision architectures bring valuable on-device AI capabilities to a much broader range of devices than language model counterparts, without straining memory, compute, or battery. Model sizes tend to be much smaller (e.g., 300–800 MB) and offer consistently fast performance (e.g., tens of milliseconds on modest consumer hardware).

---

## References & Prior Art

- **System One Models & Jev:** [Introducing System One Models & Jev (TypeSafe AI)](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
- **Open Implementations & Browser Runtimes:**
  - [Laya: Multilingual Non-Autoregressive System 1 Decision Engine](https://github.com/NandhaKishorM/laya)
  - [Open-Jev: Browser-Focused TypeScript Library for Typed Decisions](https://github.com/nico-martin/open-jev)
  - [Kev: Open Jev-like Decision Models on Qwen3/3.5](https://github.com/jaredpalmer/kev) & [kev-0.6b-ONNX on Hugging Face](https://huggingface.co/onnx-community/kev-0.6b-ONNX)
