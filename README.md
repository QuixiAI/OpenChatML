# **OpenChatML 2.2 — Specification**

*Release Candidate • 2025‑08‑08*

---

## 1 · Purpose & Scope

OpenChatML is a plain‑text envelope for LLM dialogue that:

*   cleanly separates **private reasoning** (`analysis`) from **user‑visible output** (`final`),
*   provides native support for **built‑in** and **user‑defined** tools via `commentary`,
*   enables safe, predictable **streaming** and cancellation,
*   remains readable by 1.x parsers by treating legacy messages as `final`,
*   and **interoperates with Harmony** (gpt‑oss) via HIP‑1.

---

## 2 · Document Header

Every transcript **must** begin with a YAML header.

```yaml
version: 2.2                       # required – spec major.minor
model:   gpt-oss-120b              # optional – model identifier
generation_settings:               # optional – settings are transcript-global
  temperature: 0.7
  reasoning_effort: "medium"       # low · medium · high
  builtin_tools: ["browser","python"]
capabilities:                      # optional negotiation block
  roles: ["system","developer","user","assistant","functions.*","tool"]
  channels: ["analysis","commentary","final"]
  tokens: ["<|start|>","<|channel|>","<|message|>","<|call|>","<|constrain|>","<|return|>","<|end|>","<|literal|>","<|endliteral|>"]
  streaming: ["response.delta","response.cancel","tool.cancel"]
profiles:
  harmony:
    enabled: true
    encoding: o200k_harmony        # informative; parsers MUST ignore unknown keys
    require_channels: ["analysis","commentary","final"]
    tool_calls_in: "commentary"    # see HIP‑1 §3
    builtins_in: ["analysis","commentary"]
```

*Parsers MUST ignore unknown keys.*
*`generation_settings` apply to the entire transcript unless a future revision specifies a mechanism for per-message overrides.*

---

## 3 · Control Tokens

*(Each SHOULD be a single BPE where possible; see portability clause.)*

| Token | Purpose |
| :--- | :--- |
| `<\|start\|>` | Open a message header |
| `<\|channel\|>` | Declare logical channel (`analysis` · `commentary` · `final`) |
| `<\|message\|>` | Start of body |
| `<\|call\|>` | Terminates JSON of an **outgoing** tool call (router stop) |
| `<\|constrain\|>` | Declares input data type for the subsequent body (e.g., `json`) |
| `<\|return\|>` | Sampling terminator for the **final** assistant message (stop token) |
| `<\|end\|>` | Close the message |
| `<\|literal\|>` | Begin a literal block; the body is treated as opaque bytes (no token scanning) |
| `<\|endliteral\|>` | End a literal block |

**Escaping:** Outside a literal block, a literal control token can be written by doubling the leading `<`: `<<|start|>` renders as `<|start|>`.

**Portability:** If single‑BPE registration is unavailable, parsers **MUST** recognize the literal ASCII sequences across token boundaries.

---

## 4 · Roles

| Role | Description |
| :--- | :--- |
| `system` | Ground‑truth constraints (highest authority) — **hidden by default** |
| `developer` | Implementation‑time instructions & tool defs — **hidden by default** |
| `user` | End‑user input |
| `assistant` | Model output |
| `functions.<tool>` | **Legacy/Compat role** for a tool’s reply (e.g., `functions.search`). |
| `tool` | **Canonical role** for a tool’s reply. Use with `name=` to carry the specific tool name. |

**Instruction hierarchy:** `system` > `developer` > `user` > `assistant` > `tool`.

**Canonical Usage:** For tool replies, producers **SHOULD** use `ROLE=tool name={tool_name}`. Parsers **MUST** accept `ROLE=functions.{tool_name}` for Harmony compatibility (see HIP-1).

**Renderer policy:** UIs **MUST** hide `system`, `developer`, `analysis`, and `commentary` from end users by default. Debug views MAY expose them with explicit opt‑in.

---

## 5 · Channels

| Channel | Shown to user? | Typical content |
| :--- | :--- | :--- |
| `analysis` | **No** | Chain‑of‑thought, self‑critique, internal tool use |
| `commentary` | **No** (see below) | Tool call JSON; optional **preambles** intended for the user when flagged `intent=preamble` |
| `final` | **Yes** | Polished answer |

**Rules**

1.  Each message has **exactly one** channel.
2.  If `<\|channel\|>` is absent, treat the message as `final` (back‑compat).
3.  Renderers **MUST NOT** display `analysis` content.
4.  `commentary` MAY include end‑user‑visible **preambles** (plans, action lists). To render, the assistant MUST set `intent=preamble` in the header (see §6) or copy content into a `final` message.

---

## 6 · Message Grammar

```
<|start|>{ROLE}[ SP to={DEST}][ SP call_id={ID}][ SP name={ALIAS}][ SP intent={INTENT}][ SP content_type={CTYPE}]
[<|channel|>{CHANNEL}][ SP intent={INTENT}][ SP content_type={CTYPE}]
[<|constrain|>{CTYPE}]
<|message|>{BODY}<|end|>
```

*   `{DEST}` — Required when routing to a tool. **Canonical position is in the `<|start|>` header.** Format: `functions.{tool}` or `browser.search`. For HIP-1 compatibility, parsers **MUST** also accept `to=` after the channel tag.
*   `{ID}` — Required, unique identifier for a tool call. It **MUST** be present in the `<|start|>` header of the outgoing call and the corresponding tool reply.
*   `{ALIAS}` — Optional display name. When `ROLE=tool`, this **MUST** be set to the tool's name (e.g., `name=functions.get_current_weather`).
*   `{INTENT}` — Optional, values: `preamble` (user‑visible commentary), `status`, `debug` (non‑display).
*   `{CTYPE}` — Optional content type hint (e.g., `json`, `text`, `markdown`). When present in `<|constrain|>`, it applies to the body. If the body fails to parse as the specified type, runtimes **MUST** raise an `E-BODY-CONSTRAINT-VIOLATION` error.
*   `{BODY}` — UTF‑8 text or JSON. For an **outgoing** tool call, end the message with `<\|call\|>`.

**Canonical tool call envelope:** Use the tokenized text path above. A turn MUST NOT mix `<\|call\|>` and any alternate JSON `tool_calls` field.

---

## 7 · Chain‑of‑Thought Markers

*(inside `analysis` only — never shown to end users)*

*   `<|start_reflect|>` — self‑critique
*   `<|start_introspect|>` — latent probes
*   `<|start_reason|>` — step‑by‑step logic

**Semantics:** These markers are intended primarily for human-readable debugging and offline model analysis. Automated tools MAY be designed to parse them, but they are not intended to trigger specific runtime behaviors.

**Safety:** CoT is not moderated to the same standard as `final`. Do not display. When the previous assistant turn ended in `final`, subsequent sampling SHOULD drop prior CoT. Exception: retain `analysis` leading to unresolved tool calls.

---

## 8 · Tool Framework

### 8.1 Canonical invocation sequence

```
1) assistant   (analysis)     – rationale (optional)
2) assistant to=functions.X call_id=xyz789 (commentary)<|constrain|>json … <|call|>
3) tool name=functions.X call_id=xyz789 to=assistant (commentary, JSON result)
4) assistant   (analysis)     – post‑processing (optional)
5) assistant   (final)        – user answer <|return|>
```

### 8.2 Concurrency, ids, timeouts, cancellation

*   An assistant turn MAY request **N ≥ 1** tool calls.
*   Each call **MUST** carry a unique `call_id` in its `<|start|>` header. This attribute is the canonical identifier for the call.
*   Tool replies **MUST** echo the `call_id` in their header to correlate request and response.
*   Runtimes MAY execute calls concurrently; the assistant MUST NOT assume arrival order.
*   Optional `deadline_ms` per call can be specified in the call's JSON body.
*   New event: `tool.cancel {call_id, reason}`; tools SHOULD be idempotent.

**Reply envelope (recommended):**

```json
{ "ok": true, "content": { ... }, "error": null, "provenance": { "source": "...", "timestamp": "..." } }
```
*Note: The `call_id` is now handled in the message header, not the JSON body.*

### 8.3 Built‑in tools

*   `browser` and `python` MAY be invoked from `analysis` **or** `commentary` (Harmony‑style).
*   Function tools (developer‑defined) **must** be routed on `commentary`.
*   Tool inputs MUST be limited to the `commentary` body; tools MUST NOT receive `analysis`.

---

## 9 · Streaming & Cancellation

**Events**

| Event | Contains | UI policy |
| :--- | :--- | :--- |
| `response.delta` | `final` token slices | display |
| `response.cancel` | cancellation notice | stop rendering |
| `response.delta.flush` | paragraph/segment boundary hint | display boundary |
| `tool.cancel` | `{call_id, reason}` | internal routing |

*   Do not interleave unrelated streams unless each is tagged by a `message_id`.
*   If streaming ends without `…done` (i.e., without `<|return|>` or `<|call|>`), the message is incomplete.

---

## 10 · JSON Interchange (informative)

For API transports that prefer a JSON envelope mirroring the text transcript:

```jsonc
{
  "role": "assistant",
  "channel": "final",
  "content": "Visible reply",
  "tool_call": {                   // absent unless <|call|> was emitted
    "id": "abc123",                 // from call_id= header attribute
    "recipient": "functions.lookup_weather", // from to= header attribute
    "content_type": "json",
    "arguments": "{\"location\":\"SF\"}"
  },
  "thinking": "Private reasoning (analysis)",   // optional; never deliver to end users
  "intent": "preamble"                          // optional; see §5
}
```

> **Rule:** The plain‑text envelope is canonical. JSON is a faithful projection; do not invent states the text cannot represent.

---

## 11 · Security & Provenance

*   Enforce channel boundaries: `analysis`/`commentary` MUST NOT be shown to end users unless explicitly copied to `final` or flagged `intent=preamble`.
*   Tools MUST return machine‑readable errors and SHOULD include `provenance` (source URL, checksum, timestamp) when applicable.
*   Runtimes SHOULD sanitize/escape user‑supplied strings inside message headers.

---

## 12 · Harmony Interop Profile (HIP‑1)

The following constraints ensure clean round‑trips with **Harmony / gpt‑oss**:

1.  **Tokens:** Include `<|constrain|>` and honor `<|return|>` as a valid stop token. Harmony’s `o200k_harmony` encoding is permitted but not required.
2.  **Channels:** Always include one of `analysis`, `commentary`, `final`. Harmony commonly emits `analysis` messages before a `final` or a `<|call|>`.
3.  **Roles & recipients:**
    *   Tool calls use `assistant … to=functions.{tool}` in `commentary`. For compatibility, parsers MUST accept `to=` after the `<|channel|>` tag, though the `<|start|>` header is canonical for OCM 2.2.
    *   Tool replies MAY use `ROLE=functions.{tool}`. OCM 2.2 producers **SHOULD** use the canonical `ROLE=tool name={functions.{tool}}`, but parsers **MUST** accept both for compatibility.
4.  **Constrain token:** For JSON tool arguments, emit `<|constrain|>json` before `<|message|>`.
5.  **Built‑ins:** Allow `browser.*` and `python` from `analysis` **or** `commentary`. (Harmony may prefer `analysis` for built‑ins.)
6.  **Preambles:** If the model emits a plan for the user on `commentary`, set `intent=preamble` so renderers may display it (Harmony calls this a “preamble”).
7.  **CoT handling:** When a turn ends with `final` (`<|return|>`), drop prior `analysis` on subsequent sampling. When a turn ends with `<|call|>`, include prior `analysis` and the call when resuming.
8.  **Stop conditions:** Treat `<|return|>` and `<|call|>` as hard stops for inference.

---

## 13 · Back‑Compatibility

*   Any 1.x transcript (no channels, no `developer`, etc.) is valid under 2.x and is read as all‑`final`.
*   A 1.x parser will not understand 2.x features and SHOULD fail loudly.

---

## 14 · Error Taxonomy

*   `E-PARSE-HEADER`: malformed header
*   `E-PARSE-CHANNEL-MISSING`: channel absent where required by profile
*   `E-BODY-CONSTRAINT-VIOLATION`: message body failed to parse against `<|constrain|>` type
*   `E-CALL-SCHEMA`: tool args missing/invalid
*   `E-TOOL-TIMEOUT`: tool exceeded `deadline_ms`
*   `E-TOOL-CANCELLED`: tool cancelled by runtime
*   `E-STREAM-TRUNCATED`: streaming ended without a stop token
*   `E-PERM-VISIBILITY`: attempt to display hidden channels

Runtimes SHOULD log these and provide user‑safe summaries when appropriate.

---

## 15 · ABNF (informative)

```abnf
WSP        = SP / HTAB
ROLE       = "system" / "developer" / "user" / "assistant" / "tool" / "functions." 1*(ALPHA / DIGIT / "_" / "-")
CHANNEL    = "analysis" / "commentary" / "final"
TOKEN      = "<|start|>" / "<|channel|>" / "<|message|>" / "<|call|>" / "<|constrain|>" / "<|return|>" / "<|end|>" / "<|literal|>" / "<|endliteral|>"
HEADER     = "<|start|>" ROLE [WSP "to=" 1*VCHAR] [WSP "call_id=" 1*VCHAR] [WSP "name=" 1*VCHAR] [WSP "intent=" 1*VCHAR] [WSP "content_type=" 1*VCHAR]
CHANNELTAG = [ "<|channel|>" CHANNEL [WSP "intent=" 1*VCHAR] [WSP "content_type=" 1*VCHAR] ]
CONSTRAIN  = [ "<|constrain|>" 1*VCHAR ]
MESSAGE    = "<|message|>" *OCTET
FOOTER     = "<|end|>"
FRAME      = HEADER [WSP] CHANNELTAG [WSP] CONSTRAIN MESSAGE FOOTER
```

*`MESSAGE` terminates at the next `<|end|>` matching the current frame.*

---

## 16 · Worked Examples

### 16.1 Minimal chat (no tools)

```
<|start|>user<|message|>What is 2 + 2?<|end|>
<|start|>assistant<|channel|>analysis<|message|>Simple arithmetic; answer directly.<|end|>
<|start|>assistant<|channel|>final<|message|>4.<|return|>
```

### 16.2 Canonical function call with JSON constraint

```
<|start|>system<|message|>You are a helpful AI assistant.
Knowledge cutoff: 2024-06
Current date: 2025-08-08

Reasoning: high
# Valid channels: analysis, commentary, final. Channel must be included for every message.
Calls to these tools must go to the commentary channel: 'functions'.<|end|>

<|start|>developer<|message|># Tools

## functions
namespace functions {
// Gets weather for a city.
type get_current_weather = (_: {
  location: string,
  format?: "celsius" | "fahrenheit", // default: celsius
}) => any;
} // namespace functions<|end|>

<|start|>user<|message|>What's the weather in Tokyo?<|end|>
<|start|>assistant<|channel|>analysis<|message|>Call functions.get_current_weather with location Tokyo.<|end|>
<|start|>assistant to=functions.get_current_weather call_id=wx1<|channel|>commentary<|constrain|>json<|message|>{"location":"Tokyo","format":"celsius"}<|call|>
<|start|>tool name=functions.get_current_weather call_id=wx1 to=assistant<|channel|>commentary<|message|>{"ok":true,"content":{"temperature":20,"sunny":true}}<|end|>
<|start|>assistant<|channel|>final<|message|>It’s 20 °C and sunny in Tokyo right now.<|return|>
```

### 16.3 Commentary preamble intended for the user

```
<|start|>assistant intent=preamble<|channel|>commentary<|message|>**Plan:** 1) Search docs 2) Extract figures 3) Summarize.<|end|>
```

Renderers MAY show this preamble to the user because `intent=preamble` is set.

### 16.4 Literal block (embedding control tokens safely)

```
<|start|>user<|message|>Please print these markers exactly:
<|literal|>
<|start|><|channel|><|message|><|end|>
<|endliteral|><|end|>
```

---

## 17 · Conformance Fixtures (suggested)

1.  1.x transcript (no channels) → parsed as all‑`final`.
2.  Fully‑channeled 2.2 with `<|return|>`.
3.  Multi‑tool turn with two concurrent calls and echoed `call_id`s in headers.
4.  Tool error path (`ok:false` in body, `E-TOOL-TIMEOUT` as error code).
5.  Literal block containing `<|start|>`.
6.  Message body that violates a `<|constrain|>json` directive → `E-BODY-CONSTRAINT-VIOLATION`.
7.  Harmony preamble on `commentary` with `intent=preamble`.
8.  Tool reply using the legacy `ROLE=functions.x` → parsed successfully.

---

## 18 · Implementation Checklist

| Area | Action |
| :--- | :--- |
| **Tokenizer** | Add the nine control tokens; support multi‑token fallback. |
| **Parser** | Channel optionality; `final` default; `to=` accepted in start header (canonical) or after `<|channel|>`. |
| **Renderer** | Hide `system`/`developer` by default; hide `analysis`/`commentary` unless `intent=preamble`. |
| **Tool Router** | Dispatch based on `assistant to=...`; use `call_id` from header for tracking; support concurrency + cancellation. |
| **Streaming** | Implement `response.delta`, `response.cancel`, `response.delta.flush`. |
| **Safety** | Enforce channel boundaries; never surface CoT; handle constraint violations by raising `E-BODY-CONSTRAINT-VIOLATION`. |
| **Harmony** | Enable HIP‑1; honor `<|constrain|>` and built‑ins from `analysis` or `commentary`; accept legacy role/attribute positions. |
| **Training** | Teach model to end `final` with `<|return|>`; use `<|call|>` for tool calls with `to=` and `call_id=` in the header. |
