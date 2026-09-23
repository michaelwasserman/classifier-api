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

We are exploring a declarative, **three-step developer pattern** on `window.Classifier` (exposed in `Window` contexts):
1. **Define a question schema** (`context` plus `questions` using `binary`/`boolean`, `categorical`/`choice`, or `ordinal`/`score` modalities) and initialize a session via `Classifier.create(schema)`.
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

To make on-device classification models dependable primitives for web software, the API and underlying browser runtime must address key functional and operational constraints:

- **Strict Output Conformance:** The API must never throw syntax parsing errors or return hallucinated out-of-vocabulary strings. Every `categorical`/`choice` decision `label` is strictly one of the caller's `options`; every `binary`/`boolean` or `ordinal`/`score` outputs bounded probabilities (`[0.0, 1.0]`) and well-defined expected values (`expectedScore`).
- **Calibrated Confidence:** Raw neural network logits are frequently overconfident. Runtimes must apply calibration (such as sequence-bucketed RLCD temperature scaling or multi-pass variance estimation) so that returned `probabilities` and `confidence` scores reliably reflect a useful certainty signal.
- **Single-Pass Parallelism & Question Isolation:** Multiple questions evaluated over the same input state share the encoded state representation while remaining isolated from each other. Packing multiple questions into a schema evaluates in a single pass rather than sequential model invocations.
- **Question & Option Limits (Taxonomy Design):**
  - *Number of Questions & Choices:* Because the model evaluates every question and option in a single pass, there are practical limits on how many questions fit in one schema and how many `options` each question can list (typically 2–16 options in compact models, or up to dozens in larger models). To pick from a much larger set (like hundreds of products), apps can narrow candidates down in stages or score items individually.
  - *Clear, Distinct Options:* When options overlap in meaning (e.g., `"fees"` vs. `"billing"`), lack helpful `description` text, or don't match the input, a well-calibrated model splits its probabilities across the plausible choices and reports a lower `confidence` score rather than guessing blindly.
  - *Order Independence:* Whether an option is listed first or last in the `options` array should not bias the model's scores.
- **Input Length & Context Bounds (`contextWindow`, `contextUsage`, `measureContextUsage`):** On-device encoders and diffusion backbones have bounded token windows (e.g., 512–1,024 tokens for compact checkpoints up to 8,192 tokens for larger backbones). Consistent with the Prompt API (`LanguageModel`) and other built-in AI APIs, a `Classifier` session can expose its total `contextWindow`, its `contextUsage`, and a `measureContextUsage(input, options)` method so clients can inspect capacity at runtime:
  - *Session Creation (`Classifier.create`):* Compiling the session's `context`, `questions`, and `options` consumes a fixed portion of the window, reflected by `classifier.contextUsage`. Because `Classifier` is stateless across calls (it does not accumulate chat history), `classifier.contextUsage` remains **static** for the lifetime of the session. If the schema itself exceeds `contextWindow`, `Classifier.create()` rejects with a `QuotaExceededError`.
  - *Per-Call Evaluation (`classifier.classify`):* Each `classify(input, options)` call must fit within the remaining capacity (`classifier.contextWindow - classifier.contextUsage`). Clients can pre-check an input's footprint via `await classifier.measureContextUsage(input, options)` to chunk or trim long content; if `contextUsage + inputUsage > contextWindow`, `classify()` rejects with a `QuotaExceededError` rather than silently truncating text.
- **Multi-Language & Modality Support (`expectedInputs`):** Consistent with the [Prompt API (`LanguageModel`)](https://github.com/webmachinelearning/prompt-api), `Classifier.availability()` and `Classifier.create()` accept `expectedInputs` (e.g., `expectedInputs: [{ type: "text", languages: ["en", "es", "ja"] }]`). Because an English-only encoder can fail with overconfident guesses on non-Latin scripts, declaring expected input languages and modalities upfront allows the browser runtime to verify capability coverage, select or download a suitable multilingual/multimodal checkpoint, or return `"unavailable"` when a requested language or input type is not supported.

---

## 5. Open Questions & Future Explorations

We invite community feedback on several open design questions:

1. **Result Ergonomics, Additive Confidence Diagnostics & Uncertainty Controls:**
   - *Keyed Record vs. Positional Array / Wrapper:* Having `classify()` resolve to a `record<DOMString, ClassifierDecision>` keyed by question `id` allows developers to either hold all decisions in a single cohesive `result` object (`result.category.label`) or destructure by name (`const { command } = await classifier.classify(...)`). Alternative shapes under consideration include also accepting `questions` as a keyed object in `Classifier.create()`, returning a positional `sequence<ClassifierDecision>` array, or returning a wrapper dictionary (`{ decisions }`) if top-level call metadata is needed.
   - *Additive Confidence & Sampling Controls:* The baseline API shape intentionally starts minimal, returning `probabilities`, `label`, `expectedScore`, and `confidence` on each `ClassifierDecision`. Because both input options (`ClassifierCreateOptions`, `ClassifierClassifyOptions`) and per-question decisions (`ClassifierDecision`) are extensible WebIDL dictionaries, richer uncertainty controls and diagnostics can be layered on as **strictly additive, non-breaking enhancements** if warranted. Should implementation-specific diffusion sampling controls and metrics (such as `maxSamples` / `samples`, `canvasWidth`, `drawsExecuted`, `agreement`, or `standardError`) be exposed as optional dictionary members for advanced callers, or kept internal in favor of model-agnostic effort hints (`classify(input, { effort })`) and uncertainty fields?
2. **Batching States (`classifyBatch`):** Evaluating *many questions over one input* is naturally parallelized in a single pass. For ranking or filtering *many inputs against one schema* (e.g., scoring 50 feed items or tabs), should `Classifier` expose a first-class `classifyBatch(inputs)` method?
3. **Multi-Modal `expectedInputs` vs. Simpler `expectedInputLanguages` / `expectedLanguage` Shorthand:** Adopting the Prompt API's `expectedInputs: [{ type: "text", languages: ["en"] }, { type: "image" }, { type: "audio" }]` shape prepares `Classifier` to accept `ImageBitmapSource` or `AudioBuffer` inputs alongside text without breaking session creation semantics. For common text-only classification tasks, should `Classifier.create()` and `Classifier.availability()` also accept a simpler `expectedInputLanguages: ["en"]` (as in `Summarizer` / `Writer`) or `expectedLanguage` shorthand option, or exclusively use `expectedInputs`?
4. **Standardized vs. Client-Defined Taxonomies:** Alongside custom developer-defined `questions` schemas, should `Classifier.create()` also accept shorthand identifiers for standardized, interoperable web taxonomies (e.g., `taxonomy: "iab-v3.1"`) that return stable taxonomy IDs?
5. **Worker Context Support & Permissions Policy:** Gating `Classifier` behind a `classifier` Permissions Policy (`PermissionsPolicyFeature::kClassifier`) currently restricts the API to `Window` contexts, because Permissions Policy is not yet defined for workers. How should `Classifier` support background worker contexts (e.g., `DedicatedWorker`, `SharedWorker`, `ServiceWorker`) as proposals like [Permissions Policy for Workers](https://github.com/explainers-by-googlers/workers-permissions-policy) mature?

---

## Appendix A: Alternatives Considered

| Approach | Pros | Cons / Why Insufficient Alone |
| :--- | :--- | :--- |
| **Autoregressive Prompt API (`LanguageModel`)** | Highly flexible; can generate arbitrary prose and explanations. | **10–100x slower and heavier** (requires multi-GB models); sequential token generation wastes energy when only a discrete decision or probability is needed; prone to uncalibrated probabilities. |
| **Server-Side Decision APIs** | Access to massive frontier models; zero client download size. | **Privacy & data sovereignty risk** (sensitive DOM/user input leaves the device); network latency blocks real-time UI; recurring API costs for developers. |
| **Bring-Your-Own Model (WebGPU / WASM + ONNX)** | Works today via libraries like Transformers.js (`open-jev`, `laya-ts`); can pair with the [Cross-Origin Storage (COS)](https://github.com/WICG/cross-origin-storage) proposal to deduplicate identical weight files across origins. | **Fragmented downloads & storage overhead:** Even with [Cross-Origin Storage](https://github.com/WICG/cross-origin-storage), users still face redundant 300MB+ downloads and disk usage unless the web ecosystem converges on a tiny, standardized set of exact model checkpoints and quantizations; also lacks browser-managed hardware/OS scheduling (LiteRT / XNNPACK / Accelerate). |
| **Fixed-Taxonomy-Only Classifier API** | Tiny footprint; stable numeric IDs for a single schema (e.g., IAB). | **Too narrow** for most app-specific routing, verification, scoring, and UI adaptation tasks where developers must define their own domain options. |

---

## Appendix B: Security, Privacy, Accessibility, i18n & Ecosystem Considerations

- **Privacy & Data Sovereignty:** All inference executes locally on the user's device. The API is strictly stateless: inputs are never transmitted over the network, stored on disk, or used to update model weights across calls. Access in `Window` contexts is gated by `SecureContext` and the `classifier` Permissions Policy (`PermissionsPolicyFeature::kClassifier`, defaulting to `Self`).
- **Security & Injection Resistance:** User-supplied `input` text is strictly separated from developer-supplied `questions` and `options` during schema compilation, preventing untrusted input from injecting fake options or corrupting adjacent decision slots. Hardware timing and floating-point precision variations must be bounded to prevent device fingerprinting.
- **Accessibility (a11y):** While the API has no built-in UI, websites can use fast local classification to make their content and controls much easier to navigate, such as matching everyday phrasing or voice commands to site actions (even when a user doesn't know the exact button label), surfacing the most likely next steps so keyboard or switch users don't have to tab through dozens of elements, and flagging dense text to offer simpler summaries or form guidance.
- **Internationalization (i18n):** Browsers must ensure equitable behavior across languages and scripts via `expectedInputs` (e.g., `[{ type: "text", languages: [...] }]`) in `Classifier.availability()` and `Classifier.create()`. If a requested language is unsupported by the active on-device checkpoint, the API must explicitly report `"unavailable"` or route to a multilingual checkpoint rather than returning uncalibrated guesses.
- **Ecosystem Effects & Device Equitability:** Non-autoregressive decision architectures bring valuable on-device AI capabilities to a much broader range of devices than language model counterparts, without straining memory, compute, or battery. Model sizes tend to be much smaller (e.g. 300MB–800MB) and offer consistently fast performance (e.g. tens of milliseconds on modest consumer hardware).

---

## References & Prior Art

- **System One Models & Jev:** [Introducing System One Models & Jev (TypeSafe AI)](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
- **Open Implementations & Browser Runtimes:**
  - [Laya: Multilingual Non-Autoregressive System 1 Decision Engine](https://github.com/NandhaKishorM/laya)
  - [Open-Jev: Browser-Focused TypeScript Library for Typed Decisions](https://github.com/nico-martin/open-jev)
  - [Kev: Open Jev-like Decision Models on Qwen3/3.5](https://github.com/jaredpalmer/kev) & [kev-0.6b-ONNX on Hugging Face](https://huggingface.co/onnx-community/kev-0.6b-ONNX)
