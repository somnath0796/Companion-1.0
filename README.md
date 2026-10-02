# Companion 1.0
Problem Summary

I have an agentic chatbot using:

- Google ADK "2.7.0" previously
- LiteLLM previously working with Gemini 2.5 Flash
- Gemini "2.5 Flash" previously worked correctly
- Architecture uses a Supervisor → Sub-agent pattern
- Sub-agents dynamically use available tools, including RAG/MCP/custom tools
- Model is accessed through an OpenAI-compatible completions interface
- Streaming is enabled
- Frontend receives agent events through AG UI/SSE

I upgraded the stack because Gemini 2.x models are being deprecated:

Previously working

Google ADK 2.7.0
LiteLLM
Gemini 2.5 Flash
OpenAI-compatible model interface
stream=True

Current failing stack

Google ADK 2.9.2
LiteLLM 1.102.x
Gemini 3.7
OpenAI-compatible model interface
stream=True

The normal chatbot interaction works, but tool calls fail during agent execution.

GitHub Copilot suggested that tool calls may be getting excluded, but I need to determine whether the tools are actually missing from the request or whether they are being lost while processing the streamed model response.

---

Agent Architecture

The architecture is approximately:

User
  |
  v
Supervisor Agent
  |
  | decides which sub-agent should handle request
  v
Sub-agent
  |
  | model determines whether a tool is required
  v
LiteLLM
  |
  | OpenAI-compatible API
  v
Model Provider / Gemini 3.7
  |
  | streamed tool call
  v
LiteLLM response translation
  |
  v
ADK event processing
  |
  v
Tool dispatcher
  |
  v
Tool result
  |
  v
Model continuation

The important part is that the model is not being called directly through ADK's native Gemini model interface. It is being accessed through LiteLLM and an OpenAI-compatible API.

---

Main Question

Why does the same agent/tool architecture work with Gemini 2.5 Flash but break after moving to Gemini 3.7 + ADK 2.9.2 + newer LiteLLM?

I want to identify whether the regression is caused by:

1. Gemini 3.x streaming behavior
2. OpenAI-compatible Gemini response translation
3. LiteLLM streaming/tool-call aggregation
4. ADK's conversion of LiteLLM responses into ADK events
5. Gemini 3 "thought_signature" handling
6. Parallel tool-call handling
7. Tool history / function-call ordering
8. ADK model capability detection
9. Some incompatibility between these versions

---

Primary Hypothesis: Streaming Tool Calls

One particularly suspicious behavior is Gemini 3.x OpenAI-compatible streaming.

A streamed response may contain a tool call in a delta:

{
  "choices": [
    {
      "delta": {
        "tool_calls": [
          {
            "id": "call_123",
            "type": "function",
            "function": {
              "name": "some_tool",
              "arguments": "{\"foo\":\"bar\"}"
            }
          }
        ]
      }
    }
  ]
}

but the final chunk may contain:

{
  "choices": [
    {
      "delta": {},
      "finish_reason": "stop"
    }
  ]
}

instead of:

"finish_reason": "tool_calls"

If this occurs, an agent runtime could potentially interpret the model turn as complete even though a tool call was emitted earlier in the stream.

This would produce:

Gemini
  |
  +-- tool_call emitted
  |
  +-- finish_reason = stop
          |
          v
       adapter
          |
          v
   model considered finished
          |
          X
   tool never dispatched

rather than:

tool_call
   |
   v
tool dispatcher
   |
   v
tool result
   |
   v
model continuation

Please verify whether this behavior exists in the exact Gemini 3.7/provider path being used.

---

Second Hypothesis: Gemini 3 Thought Signatures

Gemini 3 tool calls may contain Gemini-specific reasoning metadata such as:

"extra_content": {
  "google": {
    "thought_signature": "..."
  }
}

The critical question is whether LiteLLM and ADK preserve this information.

The expected flow is approximately:

Gemini tool call
      |
      +-- tool_call
      |
      +-- thought_signature
             |
             v
        tool execution
             |
             v
      tool result
             |
             v
next model request

If the "thought_signature" is dropped during conversion from:

Gemini → OpenAI format → LiteLLM → ADK

the next Gemini request may fail or behave incorrectly.

Please verify whether ADK 2.9.2 + LiteLLM 1.102.x preserve Gemini 3 thought signatures when using an OpenAI-compatible model interface.

---

Third Hypothesis: LiteLLM Streaming Aggregation

Please inspect LiteLLM's handling of multiple streamed "tool_calls".

The sub-agent can expose multiple tools, so Gemini may generate:

tool_call #1
tool_call #2
tool_call #3

within one streamed model response.

I want to verify that LiteLLM's streaming accumulator correctly merges all tool-call deltas rather than:

delta 1 → tool call A
delta 2 → tool call B
delta 3 → tool call C

becoming only:

final tool call = C

or otherwise losing IDs, arguments, or metadata.

This is particularly important because the supervisor/sub-agent architecture can produce more complex tool-selection behavior than a simple single-agent test.

---

Fourth Hypothesis: ADK Tool/Model Capability Detection

ADK 2.7+ changed model capability handling.

Please inspect how ADK 2.9.2 determines capabilities for a "LiteLlm" model using an OpenAI-compatible Gemini 3.x model.

Specifically verify:

supports function calling
supports tools
supports parallel function calling
supports structured output
supports streaming tool calls

I want to know whether the model/provider identifier causes ADK to make an incorrect capability assumption.

The model is not being accessed through ADK's native Gemini implementation, so model identification/capability inference may be important.

---

Fifth Hypothesis: Tool Call History Ordering

Gemini 3 may be stricter about function-call history.

The expected conversation sequence should be:

assistant
  tool_call

tool
  tool_result

assistant
  continuation

Please verify that the ADK → LiteLLM → provider pipeline does not accidentally generate something like:

assistant
  tool_call

assistant
  intermediate text

tool
  tool_result

or otherwise reorder tool calls/results.

This is especially important because the supervisor/sub-agent architecture introduces additional agent events around the tool execution.

---

Critical Debugging Experiment

The most useful isolation test is:

Test 1: Disable streaming

Run the exact same sub-agent + tool with:

stream=False

If:

stream=False → tool works
stream=True  → tool fails

then the likely problem is somewhere in:

Gemini 3.7
   ↓
streaming OpenAI compatibility
   ↓
LiteLLM stream aggregation
   ↓
ADK event reconstruction

rather than the tool schema itself.

---

Required Logging

Please help me determine where the tool call disappears.

I need to compare these four points.

1. ADK → LiteLLM request

Verify:

{
  "model": "...",
  "messages": [...],
  "tools": [...],
  "tool_choice": "...",
  "stream": true
}

Question:

Are the tools present here?

---

2. LiteLLM → Provider request

Verify that the outbound provider request still contains:

"tools": [...]

Question:

Are the same tools present after LiteLLM translation?

---

3. Raw provider → LiteLLM streaming response

Capture the raw SSE around the failure.

Specifically:

tool_calls
finish_reason
tool_call.id
function.name
function.arguments
extra_content
google.thought_signature

Question:

Does Gemini actually emit the tool call?

---

4. LiteLLM → ADK event

Inspect what ADK ultimately receives.

For example:

Raw response:
  tool_call = get_weather(...)
  finish_reason = stop

ADK event:
  function_call = None

If this occurs, the tool was not actually excluded by the model. It was lost during response/event translation.

---

Important Isolation Matrix

Please test the following:

Test| Model| Streaming| Supervisor| Tools| Purpose
A| Gemini 2.5| ON| ON| ON| Known-good baseline
B| Gemini 3.7| OFF| OFF| 1| Basic tool compatibility
C| Gemini 3.7| ON| OFF| 1| Isolate streaming
D| Gemini 3.7| ON| ON| 1| Supervisor interaction
E| Gemini 3.7| ON| OFF| Multiple| Parallel/multiple tools
F| Gemini 3.7| ON| ON| Multiple| Full production architecture

The most important comparison is:

Gemini 3.7
stream=False
vs
Gemini 3.7
stream=True

---

What I Need From the Investigation

Please inspect the relevant changes/issues in:

- Google ADK 2.7 → 2.9.2
- LiteLLM versions around 1.102.x
- Gemini 3.x OpenAI-compatible API
- Gemini 3.x function calling
- Gemini 3.x streaming
- Gemini 3 "thought_signature"
- ADK "LiteLlm" integration
- LiteLLM streaming tool-call aggregation
- parallel function calling
- tool-call history reconstruction

I specifically want to know:

1. Is Gemini 3.7 currently known to return tool_calls with
   finish_reason="stop" during streaming?

2. Does LiteLLM correctly normalize that into an OpenAI
   tool-call event?

3. Does ADK 2.9.2 correctly consume that LiteLLM event?

4. Is thought_signature preserved through LiteLLM and ADK?

5. Are multiple tool calls correctly accumulated?

6. Are tool calls being removed before the provider request,
   or are they being emitted by Gemini and subsequently lost?

7. Is there a known compatible combination of:
      ADK version
      LiteLLM version
      Gemini 3.x model
   for streaming function calling?

8. Is there a configuration/workaround that avoids the problem
   without abandoning streaming?

Most Suspicious Failure Chain

The current leading hypothesis is:

Gemini 3.7
    |
    | emits tool_call in streaming delta
    v
OpenAI-compatible response
    |
    | finish_reason incorrectly/ unexpectedly becomes "stop"
    v
LiteLLM stream aggregation
    |
    | tool call not represented correctly in final response
    v
ADK 2.9.2
    |
    | no usable FunctionCall event
    v
Tool dispatcher
    |
    X
tool never executes

A second possible failure is:

Gemini 3.7
    |
    | tool_call + thought_signature
    v
LiteLLM
    |
    X thought_signature lost
    v
ADK
    |
    v
tool executes
    |
    v
next Gemini request
    |
    X invalid/malformed Gemini 3 continuation

Please determine which of these is actually occurring rather than assuming the tools are being excluded.

Bottom Line

The Gemini 2.5 → Gemini 3.7 migration changed more than the model. The system now relies on Gemini 3-specific tool-call semantics while going through an OpenAI-compatible translation layer and ADK's event model.

The investigation should therefore focus on the complete tool-call round trip, not merely whether the model supports tools.