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
history, then add the new user message. Gradio 6.x `ChatInterface` uses the
**messages** format: `history` is a list of `{"role", "content", ...}` dicts. Replay
each, keeping only `role`/`content` (Gradio may attach extra keys like `metadata`
or `options` that the LLM API rejects):

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for turn in history:
    messages.append({"role": turn["role"], "content": turn["content"]})

messages.append({"role": "user", "content": user_message})
```

> Note: older Gradio versions passed history as `[user, assistant]` tuple pairs.
> This project runs Gradio 6.x, which uses the dict/messages format above.

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

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
The loop runs at most MAX_TOOL_ROUNDS iterations, counting ROUNDS (LLM turns),
not individual tool calls — a single response may request several tool calls but
still counts as one round. Each iteration makes a fresh LLM call, then inspects
response.choices[0].message:

(a) No tool calls — message.tool_calls is falsy. The LLM has a final answer, so I
    exit and return message.content (see the next field).

(b) Tool calls present — append the assistant message and the tool results, then
    let the loop continue to the next iteration (which makes the next LLM call).
    If the loop completes all MAX_TOOL_ROUNDS iterations without ever hitting case
    (a), the budget is exhausted. At that point the most recent assistant message
    requested tools, so its .content is empty — there is no usable final answer.
    I return a user-readable fallback string instead:
    "I wasn't able to finish looking that up. Could you try rephrasing your question?"

Structure (round-based, so the boundary is unambiguous):

    for _ in range(MAX_TOOL_ROUNDS):
        response = client.chat.completions.create(...)   # fresh call each round
        message = response.choices[0].message
        if not message.tool_calls:
            return message.content                        # case (a)
        messages.append(message)
        for tool_call in message.tool_calls:              # may be >1 — one round
            ...                                           # append tool result
    return FALLBACK_MESSAGE                                # case (b): budget hit
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
When the loop exits via case (a), the final text lives on the assistant message's
content field: response.choices[0].message.content (a str). I return that.

Note this is only populated when the message has no tool_calls — an assistant
message that requested tools carries its intent in .tool_calls and leaves .content
empty, which is exactly why the MAX_TOOL_ROUNDS exit (case b) needs its own
fallback string rather than returning .content.
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: [tool name, args]
Round 2 tool call: [tool name, args] (if any)
Final response: [brief description]
```

**What happens when you ask about a plant that isn't in the database?**

```
[describe the behavior you observed]
```

**One thing about the tool call API that surprised you:**

```
[your answer here]
```
