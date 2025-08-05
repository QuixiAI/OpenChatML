# **OpenChatML 2.0 — Specification**

*Release Candidate • 2025-08-05*

---

## 1 · Purpose & Scope

OpenChatML 2.0 is a plain-text envelope for Large Language Model (LLM) dialogue that

* cleanly separates **private reasoning** (`analysis`) from **user-visible output** (`final`),
* provides native support for **built-in** and **user-defined** tools via a `commentary` channel,
* enables safe, predictable **streaming**, and
* remains readable by 1.x parsers by treating legacy messages as `final`.

---

## 2 · Document Header

Every transcript **must** begin with a YAML header.

```yaml
version: 2.0                 # required – spec major version
model:   gpt-oss-120b        # optional – model identifier
generation_settings:         # optional block
  temperature: 0.7
  reasoning_effort: "medium" # low · medium · high
  builtin_tools: ["browser"] # enable predefined tools
```

*Parsers MUST ignore unknown keys.*

---

## 3 · Control Tokens

*(Each SHOULD be a single BPE token.)*

| Token           | Purpose                                                       |
| --------------- | ------------------------------------------------------------- |
| `<\|start\|>`   | Open a message header                                         |
| `<\|channel\|>` | Declare logical channel (`analysis` · `commentary` · `final`) |
| `<\|message\|>` | Start of body                                                 |
| `<\|call\|>`    | Terminates JSON of an **outgoing** tool call                  |
| `<\|return\|>`  | Generation terminator (training only)                         |
| `<\|end\|>`     | Close the message                                             |

---

## 4 · Roles

| Role               | Description                                        |
| ------------------ | -------------------------------------------------- |
| `system`           | Ground-truth constraints (highest authority)       |
| `developer`        | Implementation-time instructions & tool defs       |
| `user`             | End-user input                                     |
| `assistant`        | Model output                                       |
| `functions.<tool>` | Synthetic author of a tool’s reply                 |
| `tool`             | **Alias** for `functions.<tool>` (interchangeable) |

---

## 5 · Channels

| Channel      | Shown to user? | Typical content                 |
| ------------ | -------------- | ------------------------------- |
| `analysis`   | **No**         | Chain-of-thought, self-critique |
| `commentary` | **No**         | Raw JSON to/from tools          |
| `final`      | **Yes**        | Polished answer                 |

**Rules**

1. Each message has **exactly one** channel.
2. A message lacking `<\|channel\|>` **MUST** be treated as `final` *(back-compat)*.
3. Renderers **MUST** hide or redact `analysis` & `commentary`.

---

## 6 · Message Grammar

```
<|start|>{ROLE}[ to={DEST}][ name={ALIAS}]<|channel|>{CHANNEL}<|message|>{BODY}<|end|>
```

* `{DEST}` — required when routing to a tool or when a tool responds.
* `{ALIAS}` — optional display name (rare).
* `{BODY}` — UTF-8 text or JSON; if JSON is an **outgoing** call, append `<\|call\|>`.

---

## 7 · Chain-of-Thought Markers

*(inside `analysis` only)*

* `<|start_reflect|>` — self-critique
* `<|start_introspect|>` — latent probes
* `<|start_reason|>` — step-by-step logic

Raw CoT must **never** appear in `final`.

---

## 8 · Tool Framework

### 8.1 Built-in Tools

Names in `generation_settings.builtin_tools` (e.g. `browser`, `python`) are invocable without extra schema.

### 8.2 User-Defined Tools

Supply OpenAPI-style schemas inside a `developer` message; they are addressed as `functions.<tool_name>`.

### 8.3 Invocation Sequence

```
1) assistant   (analysis)     – rationale
2) assistant→functions.X (commentary + JSON)<|call|>
3) functions.X→assistant (commentary, JSON result)
4) assistant   (analysis)     – post-processing
5) assistant   (final)        – user answer
```

---

## 9 · Streaming Events (Optional API)

| Event                                   | Contains             | UI policy |
| --------------------------------------- | -------------------- | --------- |
| `response.delta`                        | `final` token slices | display   |
| `response.reasoning_text.delta`         | raw CoT              | hide      |
| `response.reasoning_summary_text.delta` | moderated summary    | optional  |
| `…done`                                 | channel completion   | internal  |

---

## 10 · JSON Interchange Schema

```jsonc
{
  "role":    "assistant",
  "thinking": "Private reasoning (analysis)",      // optional
  "content":  "Visible reply (final)",             // optional
  "tool_calls": [                                  // optional
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "browser.search",
        "arguments": "{\"query\":\"latest news\"}"
      }
    }
  ]
}
```

*An assistant turn supplies **either** `content` **or** `tool_calls`; `thinking` may accompany either.*

---

## 11 · Worked Example (Raw Transcript)

```text
<|start|>developer<|message|>
# Instructions
Use `browser` for news. When user orders, call `order_pizza`.
<|end|>

<|start|>user<|message|>
What's the latest Mars-rover news? Then order a large pepperoni pizza.
<|end|>

<|start|>assistant<|channel|>analysis<|message|>
Two tasks: (1) fetch rover news, (2) order pizza.
<|end|>

<|start|>assistant to=functions.browser.search<|channel|>commentary<|message|>
{"query":"latest Mars rover news"}<|call|>

<|start|>functions.browser.search to=assistant<|channel|>commentary<|message|>
{"results":[{"title":"Rover Finds Ancient Water Clues","url":"…"}]}
<|end|>

<|start|>assistant<|channel|>analysis<|message|>
Summarised news; next, call pizza function.
<|end|>

<|start|>assistant<|channel|>final<|message|>
**News:** Rover has found new evidence of ancient water on Mars!  
Placing your pizza order now…
<|end|>

<|start|>assistant to=functions.order_pizza<|channel|>commentary<|message|>
{"size":"large","toppings":["pepperoni"]}<|call|>
```

---

## 12 · Back-Compatibility

* Any 1.x transcript (no channels, no `developer`, etc.) is valid under 2.0 and is read as all-`final`.
* A 1.x parser will not understand 2.0 features and **should fail loudly**.

---

## 13 · Implementation Checklist

| Area            | Action                                                                                |
| --------------- | ------------------------------------------------------------------------------------- |
| **Tokenizer**   | Add the six control tokens.                                                           |
| **Parser**      | Enforce exactly-one-channel; default to `final` if absent.                            |
| **Renderer**    | Hide `analysis` & `commentary`.                                                       |
| **Tool Router** | Dispatch `assistant to=functions.*`; return results under `functions.* to=assistant`. |
| **Streaming**   | Optionally emit event names in §9.                                                    |
| **Training**    | Teach model to end with `<\|return\|>` in the `final` channel.                        |

---

### End of Specification
