# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

The loop runs inside a `for round_num in range(MAX_TOOL_ROUNDS)` block.

**Condition (a) — no tool calls:** After each LLM call, check `assistant_message.tool_calls`.
If it is falsy (None or empty list), the LLM has produced a final answer. Return
`assistant_message.content` immediately from inside the loop.

**Condition (b) — MAX_TOOL_ROUNDS reached:** When the for loop exhausts all rounds
without hitting a no-tool-calls response, execution falls through to code after the loop.
At that point, make one final LLM call *without* the `tools` parameter — this forces the
model to generate a text response from whatever context has been accumulated. Return
that response's content. This prevents the function from returning an empty string or
raising an exception when the safety limit is hit.

```
for round_num in range(MAX_TOOL_ROUNDS):
    call LLM
    if no tool_calls → return assistant_message.content   # exit condition (a)
    append assistant message + tool results to messages

# only reached if all MAX_TOOL_ROUNDS were consumed
final_response = call LLM without tools
return final_response.choices[0].message.content          # exit condition (b)
```

---

### Extracting the final text response

The text content lives at `response.choices[0].message.content`. Specifically:

- `response.choices` — list of completion choices (always index 0 for non-streaming)
- `.message` — the assistant message object
- `.content` — the string text of the response (is `None` when the message only contains
  tool_calls, so only read it after confirming tool_calls is falsy)

```python
return response.choices[0].message.content or ""
```

The `or ""` guards against a `None` content field, ensuring the function always returns
a string as required by the output contract.

---

## Implementation Notes

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: lookup_plant({"plant_name": "calathea"})
  → returns found=True with full calathea care dict
Round 2 tool call: get_seasonal_conditions({})
  → returns summer season data with detected_season=True
Final response: Combines calathea-specific care requirements with summer adjustments
  (e.g., increase humidity as indoor AC dries the air, water consistently, keep out
  of direct sun, maintain humidity above 50%).
```

**What happens when you ask about a plant that isn't in the database?**

```
lookup_plant returns {"found": False, "name": "string of pearls", "message": "..."}
The LLM reads the not-found message, which explicitly instructs it not to invent data.
The agent acknowledges the plant isn't in its database, then offers general guidance
based on plant type — string of pearls is a trailing succulent, so it gets general
succulent advice (infrequent deep watering, bright light, well-draining soil) clearly
framed as general knowledge rather than database-sourced data.
```

**One thing about the tool call API that surprised you:**

```
The assistant message containing tool_calls must be appended to the messages list as
a message object (not just its content string) before any tool results. This is because
the tool result messages reference the tool_call_id fields that live inside that assistant
message — the API needs both present and in order to reconstruct the call/result linkage.
Trying to pass tool results without the preceding assistant message causes an API error.
```
