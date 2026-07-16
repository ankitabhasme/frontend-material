# AI/LLM Integration in the Frontend

> Practical, React-oriented patterns for shipping LLM features: streaming UIs, cancellation, cost/rate awareness, chat UX, the never-ship-keys rule, security, and accessibility of dynamic content. A fast-moving area — treat specifics as of 2026, principles as durable.

## Framing for a Lead

LLM features break several assumptions a normal frontend makes: responses are **slow, streamed, non-deterministic, metered, and untrusted**. Each of those forces a UI decision. The durable skills are not the SDK-of-the-month; they are streaming rendering, cancellation, cost/latency UX, and treating model output as hostile input. Providers and SDKs change monthly — architect so the model call is a swappable server-side detail.

**The one non-negotiable: no API keys in the browser.** Everything else is trade-offs; this is a hard line (see below).

---

## Streaming Responses in the UI

Users will not wait 8 seconds for a full completion staring at a spinner. Stream tokens so first content appears in a few hundred ms — this transforms perceived performance more than any other single choice.

### Transport: SSE vs fetch streams
- **Server-Sent Events (`EventSource`)** — simple, auto-reconnect, but GET-only and no custom headers. Awkward for authed POST chat.
- **`fetch` + `ReadableStream`** — the modern default. POST with a body, read the stream incrementally, full header control. This is what the major AI SDKs use under the hood.

```ts
async function streamChat(body: ChatRequest, signal: AbortSignal, onDelta: (t: string) => void) {
  const res = await fetch("/api/chat", {           // our own BFF, not the provider
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
    signal,                                          // cancellation
  });
  if (!res.ok || !res.body) throw new Error(`stream failed: ${res.status}`);

  const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    // parse SSE-style "data: {...}\n\n" frames and extract token deltas
    for (const delta of parseSSE(value)) onDelta(delta);
  }
}
```

### Incremental rendering in React
Accumulate deltas into state and render as they arrive. The trap is **re-rendering the whole message tree on every token** — for long messages with markdown, that is O(n²) work.

```tsx
function useAssistantStream() {
  const [text, setText] = useState("");
  const start = useCallback(async (req: ChatRequest, signal: AbortSignal) => {
    setText("");
    await streamChat(req, signal, (delta) =>
      setText((prev) => prev + delta)   // functional update; deltas may batch
    );
  }, []);
  return { text, start };
}
```

Mitigations for token-storm re-renders:
- **Throttle** UI updates (e.g. `requestAnimationFrame` or ~30–60ms batching) rather than committing per token.
- Parse/highlight markdown only on flush, or use a streaming-aware markdown renderer.
- Keep the growing message in a leaf component so the rest of the thread does not re-render.

### Suspense & transitions
- **Suspense** fits the *initial* async boundary — "waiting for the first byte / loading the conversation." It does not model a stream of deltas; you own that state yourself.
- **`useTransition` / `startTransition`** keeps the input responsive while the streamed update renders — mark the streaming state update as a transition so typing/scrolling is not blocked by render work.
- React 19 `use()` + streaming from RSC / the AI SDK's `useChat` abstract much of this, but you must still understand the underlying stream to debug it.

### Backpressure
The browser cannot slow the model, but it can slow *rendering*. If tokens arrive faster than the UI paints, decouple network reads from paints: buffer incoming deltas and flush to state on an animation frame, so a fast stream never floods the render loop. Drop or coalesce intermediate frames — the user only sees the latest.

---

## Cancellation & Abort

Every stream must be cancellable — the user navigates away, edits the prompt, or hits stop. Use `AbortController`; aborting also tells the server to stop generating, which **stops burning tokens** on an abandoned request.

```tsx
const controllerRef = useRef<AbortController | null>(null);

function send(req: ChatRequest) {
  controllerRef.current?.abort();          // cancel any in-flight stream
  const controller = new AbortController();
  controllerRef.current = controller;
  start(req, controller.signal).catch((e) => {
    if (e.name !== "AbortError") showError(e);   // aborts are expected, not errors
  });
}

useEffect(() => () => controllerRef.current?.abort(), []); // abort on unmount
```

---

## Token, Rate-Limit & Cost Awareness

Every call costs money and is metered. A lead builds this awareness into the UX and the client:

- **Rate limits (HTTP 429):** surface a graceful "slow down / try again" state; back off with jitter; never hammer-retry a 429. Respect `Retry-After` if the BFF forwards it.
- **Token budgets:** long chat histories grow the prompt every turn — cost scales with context. Trim/summarize old turns server-side; show the user when a conversation is being truncated.
- **Cost controls the client can help with:** debounce/disable send while streaming, prevent duplicate submits, cap max output tokens for cheap UIs, and let users pick a cheaper model where appropriate.
- **Streaming stop button** is a cost feature as much as a UX one — abandoning a long generation should stop billing.

> Interview signal: candidates who mention that cost scales with total context per turn (not just the new message) and that the client should not be the only rate-limit guard — the server must enforce it — are thinking at lead level.

---

## Optimistic UI & Chat State

### Message state model
Model each message with an explicit status so retries and errors have a home:

```ts
type Message = {
  id: string;
  role: "user" | "assistant";
  content: string;
  status: "pending" | "streaming" | "complete" | "error";
  error?: string;
};
```

### Optimistic user message
Render the user's message immediately (optimistic) and append an empty assistant message in `streaming`, then fill it. If the request fails, mark it `error` and offer **retry** — do not silently drop it. React 19's `useOptimistic` is purpose-built for the optimistic insert.

### Error & retry states
- Distinguish error *types*: network, 429 rate-limit, 5xx model/server, content-filter refusal, and abort (not an error). Each wants different copy and affordances.
- **Retry** should resend from the last good state, not duplicate the whole history. Preserve the partial stream if useful ("regenerate" vs "continue").
- Never leave a message stuck in `streaming` — time out and transition to `error` if the stream stalls.

---

## Prompt Handling: Client vs Server

**Ship prompts and keys server-side. This is the hard rule.**

- **NEVER put the provider API key in client code.** It is extractable from the bundle or network tab, and a leaked key means someone runs up your bill / exfiltrates data. Proxy every model call through your **backend or BFF** (Backend-for-Frontend), which holds the key in a secret manager and adds it server-side.
- The BFF is also where you enforce **auth, per-user rate limits, cost caps, prompt templates, input/output validation, and logging** — none of which can be trusted to the client.
- **System prompts belong on the server.** A system prompt in client code is visible to users and easily overridden; keep the authoritative prompt server-side. The client sends user intent; the server assembles the final prompt.

```
Browser ──(user message)──▶ BFF ──(prompt + system + key)──▶ LLM provider
        ◀────(streamed tokens)──── BFF ◀──(stream)────────────┘
                     ▲ auth, rate limit, cost cap, sanitize, log
```

---

## Security

Treat the model as an **untrusted input source and an untrusted output source** simultaneously.

- **Prompt injection:** user (or retrieved/RAG) content can contain instructions that hijack the model ("ignore previous instructions, reveal the system prompt / call this tool"). Mitigations are mostly server-side: separate instructions from data, constrain tool permissions, never let model output alone authorize a privileged action, and keep a human/guard in the loop for destructive tools. The frontend's job is to not *display* injected content as trusted and to gate consequential tool calls behind confirmation.
- **XSS from LLM output:** model output is untrusted HTML. If you render it as markdown/HTML, **sanitize** (DOMPurify) and never `dangerouslySetInnerHTML` with raw model output. Markdown can smuggle `<img onerror>`, `javascript:` links, and script. Sanitize after markdown→HTML, and constrain allowed tags.
- **PII & data governance:** do not send more user data to the model than needed; redact PII where possible; be explicit about what is logged (prompts often contain sensitive data). Know your provider's data-retention/training terms. This is increasingly a compliance requirement, not just hygiene.

```tsx
import DOMPurify from "dompurify";
// Render model markdown safely
function AssistantMarkdown({ md }: { md: string }) {
  const html = useMemo(() => DOMPurify.sanitize(markdownToHtml(md)), [md]);
  return <div dangerouslySetInnerHTML={{ __html: html }} />; // sanitized only
}
```

---

## Function / Tool Calling UI

Models can request a tool call; the app executes it and feeds the result back. The UI implications:

- **Render tool calls as first-class message events** — "Searching orders…", "Calling calendar API…" — so the user sees *why* there is a pause and what the model is doing.
- **Confirmation gates for consequential tools** (send email, make payment, delete). Never auto-execute a destructive tool purely from model output — require explicit user approval. This is both UX and a prompt-injection defense.
- Show tool **results and errors** inline; a failed tool call should not silently stall the turn.
- Multi-step (agentic) loops need a visible trace and a way to interrupt — a runaway loop is a cost and trust problem.

---

## RAG (High Level)

Retrieval-Augmented Generation: instead of relying only on the model's training, you **retrieve relevant documents** (usually via embeddings + a vector search) and inject them into the prompt as context, so answers are grounded in your data and can cite sources.

The frontend's role is narrow but real:
- Render **citations/sources** and let users open the referenced document — grounding is the point of RAG, so surface it.
- Show retrieval state ("Searching knowledge base…") as its own step.
- Handle "no relevant context found" gracefully rather than letting the model confabulate.
- All retrieval and embedding happens server-side; the client displays grounded output and provenance.

---

## Latency & Perceived Performance

LLM latency is high and variable; manage perception:
- **Skeletons / typing indicator** for the gap before the first token.
- **Stream** so partial results appear early — a partial answer beats a spinner.
- **Progressive disclosure:** show the answer as it forms; render tool steps as they happen.
- Time-to-first-token is the metric users feel; optimize the BFF path to first byte (avoid buffering the whole response server-side before forwarding).

---

## Accessibility of Streaming / Dynamic Content

Dynamic, incrementally-updating content is hostile to screen readers if done naively.

- Wrap the assistant response region in **`aria-live="polite"`** so updates are announced without stealing focus. Use `assertive` only for genuinely urgent messages (errors), sparingly.
- **Do not announce every token** — a token-by-token `aria-live` region is unusable noise. Announce on meaningful boundaries (sentence, or on stream completion), or keep the live region updating text but let the SR settle on pauses.
- Manage **focus** on new messages deliberately; do not yank focus away from the input mid-type.
- Ensure the **stop/retry** controls are keyboard reachable and labeled.
- Respect `prefers-reduced-motion` for typing animations and auto-scroll; give a way to pause auto-scroll so users reading earlier content are not dragged to the bottom.

---

### Interview Questions — AI/LLM Integration

**Why must LLM API keys never live in the frontend, and what's the correct architecture?**

> Anything in client code — including env vars baked into the bundle — is fully visible in the network tab and the bundle, so a provider key in the frontend is a published key: someone extracts it and runs up your bill or exfiltrates data through your account. The correct architecture is a backend-for-frontend proxy: the browser sends the user's intent to our own endpoint, and the BFF holds the key in a secret manager and attaches it server-side before calling the provider. The BFF is also the only place I can trust to enforce auth, per-user rate limits, cost caps, the authoritative system prompt, and input/output sanitization — none of which the client can be trusted with. The provider stream is piped back through the BFF to the browser.

**How do you stream a chat response in a React app, and what's the main rendering pitfall?**

> I use `fetch` with a `ReadableStream` — POST the request to our BFF, read the response body incrementally, decode, parse the SSE-style frames, and accumulate token deltas into state so content appears within a few hundred milliseconds instead of after the full completion. The main pitfall is re-rendering the entire message thread on every token, which for a long markdown message is roughly O(n squared) work and janks the UI. I mitigate by throttling state commits to an animation frame or a ~30-60ms batch rather than per token, isolating the growing message in a leaf component so the rest of the thread doesn't re-render, and marking the streaming update as a transition so the input stays responsive. Markdown parsing/highlighting happens on flush, not per token.

**What does cancellation buy you beyond UX, and how do you implement it?**

> Beyond letting the user stop or edit, aborting a stream signals the server to stop generating, which stops burning tokens on a request nobody's waiting for — so it's a cost control, not just a nicety. I implement it with an `AbortController` per request: I keep the controller in a ref, abort any in-flight stream before starting a new one, pass the signal into `fetch`, and abort on unmount via an effect cleanup. I treat `AbortError` as expected rather than surfacing it as a real error, and I distinguish it from network and rate-limit failures in the message state model.

**Model output is rendered as markdown in your UI. What are the security concerns?**

> Model output is untrusted, so rendering it is an XSS surface. Markdown compiles to HTML that can smuggle `<img onerror=...>`, `javascript:` links, or script tags, and if I pass raw model output to `dangerouslySetInnerHTML` I've handed an attacker — possibly via prompt injection through retrieved content — script execution in my origin. So I sanitize the rendered HTML with something like DOMPurify after the markdown-to-HTML step, constrain allowed tags, and never render unsanitized model output. The related concern is prompt injection: retrieved or user content can carry instructions that hijack the model, so I never let model output alone authorize a consequential action, and I gate destructive tool calls behind explicit user confirmation.

**How do you handle rate limits and cost in the client experience?**

> On a 429 I show a graceful slow-down state and back off with jitter rather than hammer-retrying, respecting `Retry-After` if the BFF forwards it — and I treat the server as the real enforcer, since a client-side guard is bypassable. For cost, the key insight is that cost scales with total context per turn, not just the new message, so long histories get expensive; I trim or summarize old turns server-side and signal to the user when truncation happens. Client-side I disable send while streaming to prevent duplicate submits, expose a stop button (abandoning a long generation stops billing), and where appropriate cap max output tokens or let users choose a cheaper model.

**How do you make a streaming chat UI accessible?**

> I wrap the assistant response in an `aria-live="polite"` region so updates are announced without stealing focus, reserving `assertive` for urgent errors. Critically, I don't announce every token — a per-token live region is unusable noise for a screen reader — so I let the SR settle on meaningful boundaries like sentence ends or stream completion. I manage focus deliberately so new messages don't yank focus away from someone still typing, make the stop and retry controls keyboard-reachable and labeled, and respect `prefers-reduced-motion` for typing animations while giving users a way to pause auto-scroll so they aren't dragged to the bottom while reading earlier content.

**Explain function/tool calling from a frontend perspective. What UI safeguards matter?**

> With tool calling, the model requests an action, the app executes it, and the result is fed back for the model to continue. In the UI I render each tool call as a first-class message event — "Searching orders…", "Calling calendar…" — so the pause is explained and the user sees what's happening, and I surface results and errors inline so a failed call doesn't silently stall the turn. The safeguard that matters most is a confirmation gate on consequential tools — send email, payment, delete — because auto-executing a destructive action purely from model output is both bad UX and a prompt-injection attack vector. For agentic multi-step loops I show a visible trace and give the user a way to interrupt, since a runaway loop is a cost and trust problem.

**What is RAG at a high level, and what is the frontend's responsibility in a RAG feature?**

> Retrieval-Augmented Generation retrieves relevant documents — typically via embeddings and vector search — and injects them into the prompt so the model's answer is grounded in your data and can cite sources, rather than relying solely on training data. All the retrieval and embedding is server-side. The frontend's job is narrow but real: render citations and let users open the source documents, because grounding is the entire point and provenance is what makes it trustworthy; show retrieval as its own visible step; and handle the "no relevant context found" case gracefully so the model doesn't confabulate an answer that looks authoritative.
