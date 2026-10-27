# Case Study: Diagnosing Local LLM Tool-Calling Failures in OpenClaw

## Executive Summary

During development of the OpenClaw Opportunity Watcher, a local LLM repeatedly failed basic tool operations when orchestrated through OpenClaw.

The initial symptoms strongly suggested a model-level tool-calling problem:

- malformed tool arguments
- repeated tool-call failures
- unsuccessful file reads
- invalid tool-call structures
- retry loops
- hallucinated tool identifiers
- approximately 30 failed tool operations during one controlled test

Replacing the model would have been the easiest response.

Instead, the failure was treated as a systems problem.

The investigation progressively isolated the model, inference runtime, provider integration, agent runtime, tool schemas, and orchestration layer.

A direct test against Ollama demonstrated that the same local model could generate a valid native tool call when OpenClaw was removed from the execution path.

That shifted the fault boundary away from the model.

Further inspection showed that OpenClaw was automatically introducing a structured Tool Search abstraction between the model and the small set of tools assigned to the agent.

After disabling that abstraction, the model-facing tool surface became exactly the four tools intentionally granted to the agent.

A controlled file-read test then completed with:

```text
Tool calls: 1
Failures: 0
Result: Successful
```

The primary lesson was straightforward:

> An apparent LLM failure may actually be an orchestration failure.

---

## System Context

The Opportunity Watcher is a constrained autonomous research agent built around several independently controlled components:

```text
OpenClaw Agent
      ↓
Agent Tool Layer
      ↓
Provider Integration
      ↓
Private Network Path
      ↓
Ollama
      ↓
Local LLM
```

The agent was intentionally designed with a minimal operational tool surface:

```text
read
ls
web_search
web_fetch
```

This made the failure particularly interesting.

A model that should have needed only a simple `read` operation was producing repeated failures during an otherwise trivial file-access task.

---

## Problem Statement

The immediate question was:

**Why could the local model reason normally but fail to execute a basic OpenClaw tool operation reliably?**

Several explanations were plausible.

### Hypothesis 1 — Model Failure

The local model might not reliably support structured tool calling.

### Hypothesis 2 — Ollama Failure

Ollama might not be exposing or translating tool calls correctly.

### Hypothesis 3 — Provider / Adapter Failure

The OpenClaw-to-Ollama integration might be altering the model's native tool-call representation.

### Hypothesis 4 — Tool Schema Failure

The schemas presented to the model might be malformed, overly complex, or incompatible with the model.

### Hypothesis 5 — Context Failure

Large prompts, workspace instructions, or session history might be degrading structured output.

### Hypothesis 6 — Session Contamination

Previous failed calls might be influencing subsequent retries.

### Hypothesis 7 — Orchestration Failure

OpenClaw might be inserting additional abstractions between the model and the explicitly permitted tools.

The investigation was designed to reduce this hypothesis space rather than immediately tuning prompts or replacing models.

---

## Initial Symptoms

Testing across local models produced several inconsistent behaviors.

Observed failures included:

- tool arguments serialized incorrectly
- invalid parameter structures
- unsuccessful file reads
- repeated retries of the same operation
- model responses claiming progress without successful tool execution
- hallucinated tool identifiers
- malformed tool-call metadata

One Qwen-based OpenClaw test produced approximately 30 failed tool operations without completing the intended read.

At this point, it would have been reasonable to conclude:

> The local model is not capable of reliable tool use.

That conclusion would have been wrong.

---

## Isolation Strategy

The key diagnostic principle was to test the same capability while removing layers from the execution path.

The original path was conceptually:

```text
Model
  ↑
Ollama
  ↑
OpenClaw Provider Integration
  ↑
OpenClaw Agent Runtime
  ↑
OpenClaw Tool Abstraction
```

Rather than changing the prompt repeatedly, the test moved closer to the model.

The critical question became:

**Can the model produce a valid native tool call when OpenClaw is not involved?**

---

## Direct Ollama Test

A direct Ollama `/api/chat` request was constructed with a simple tool definition.

This bypassed the OpenClaw agent runtime and its tool orchestration while preserving the same underlying local model and Ollama inference runtime.

The result was decisive.

The model returned a valid native tool call with:

- a valid tool identifier
- the expected function
- correctly structured arguments

The response completed successfully in only a few seconds.

This established an important fact:

> The model was capable of structured tool calling through Ollama.

The fault boundary therefore moved upward.

The primary suspects were no longer the model itself or Ollama's basic native tool support.

Investigation shifted toward OpenClaw's provider and orchestration behavior.

---

## Inspecting the Model-Facing Tool Surface

The next step was to inspect what OpenClaw was actually presenting to the model.

The Opportunity Watcher had been deliberately granted only:

```text
ls
read
web_search
web_fetch
```

However, runtime inspection showed that the model-facing tool environment contained additional abstractions:

```text
ls
read
tool_search
tool_describe
tool_call
```

Instead of interacting directly with the intentionally small tool surface, the model was being asked to operate through an additional tool-discovery and invocation layer.

This changed the architecture from approximately:

```text
Model
  ↓
read
```

to:

```text
Model
  ↓
Tool Search
  ↓
Tool Description
  ↓
Tool Invocation
  ↓
read
```

For a large frontier model, this additional abstraction may be manageable.

For the local model being tested, it introduced unnecessary structured reasoning and additional failure opportunities.

---

## Root Cause

OpenClaw's structured Tool Search functionality was being automatically applied to the local Ollama configuration.

The agent itself had not been intentionally designed to require dynamic tool discovery.

Its permitted tool set was already extremely small and explicitly known.

Structured Tool Search therefore provided little value while increasing orchestration complexity.

The practical root cause was:

> **An unnecessary orchestration abstraction was being inserted between the local model and an already constrained tool surface.**

The model was not simply failing to call `read`.

It was struggling with the additional meta-tool workflow used to discover, describe, and invoke `read`.

---

## Remediation

Structured Tool Search was explicitly disabled for this implementation.

The effective configuration change was:

```text
tools.toolSearch = false
```

This was intentionally preferred over falsely declaring that the model did not support tools.

Native tool support had already been demonstrated directly through Ollama.

After the change, the model-facing tools became exactly:

```text
ls
read
web_search
web_fetch
```

This restored the simpler architecture originally intended for the agent.

---

## Validation

A fresh controlled test was executed after the configuration change.

The agent was instructed to perform a basic workspace file read.

Before remediation, comparable tests had generated repeated failures and retry loops.

After remediation:

```text
Tool calls: 1
Failures: 0
Result: Successful
```

The model successfully invoked `read` directly and completed the requested operation.

This provided a clean before-and-after validation of the hypothesis.

---

## Why the Diagnosis Matters

The easiest response to repeated local-model failures would have been to replace the model.

That would potentially have hidden the underlying systems issue.

A larger model might have been capable of navigating the additional abstraction more successfully, creating the appearance that the problem had been solved.

But the unnecessary orchestration complexity would still exist.

The diagnostic process instead separated:

**model capability**

from:

**system behavior**

That distinction matters when engineering agentic systems.

An agent's observable behavior is produced by more than the LLM.

It is the result of interactions among:

- model capability
- inference runtime
- API/provider integration
- system prompts
- context construction
- tool schemas
- orchestration abstractions
- permissions
- session state
- external services

Treating every failure as a prompt or model problem can lead to expensive and fragile fixes.

---

## Security Implications

The issue also had security implications.

### Smaller Tool Surfaces Are Easier to Reason About

The Opportunity Watcher required four tools.

Adding meta-tools expanded the model's effective interaction surface without providing a necessary business capability.

Removing unnecessary abstraction reduced both complexity and potential failure modes.

### Capability Should Be Explicit

The agent follows a least-privilege model.

Its available capabilities should therefore reflect deliberate design decisions rather than implicit runtime behavior wherever practical.

### Behavioral Reliability Supports Security

A tool permission is useful only if the model interacts with that tool predictably.

Repeated malformed calls, retry loops, and confused tool selection increase operational uncertainty.

Reducing orchestration complexity improved both reliability and the ability to reason about the agent's behavior.

### More Capable Models Are Not a Substitute for Architecture

A sufficiently powerful model may compensate for unnecessarily complicated orchestration.

That does not make the architecture simpler or safer.

The goal is not to determine how much complexity the model can tolerate.

The goal is to expose only the complexity required to perform the mission.

---

## Broader Engineering Lessons

### 1. Isolate Layers Before Replacing Components

When an agent fails, identify which layer owns the failure.

A useful diagnostic sequence is:

```text
Model
↓
Inference Runtime
↓
Provider / Adapter
↓
Agent Runtime
↓
Tool Layer
↓
External Dependency
```

Test components independently where possible.

### 2. Direct API Tests Are Powerful Diagnostic Controls

Bypassing the orchestration framework created a known-good comparison point.

Once the model successfully produced a native Ollama tool call, the investigation had evidence to move the fault boundary upward.

### 3. Inspect What the Model Actually Receives

Configured tools and model-facing tools are not necessarily identical.

Runtime inspection matters.

### 4. Reduce Indirection When It Provides No Value

Dynamic tool discovery can be useful when an agent has access to a large and changing capability set.

It can be unnecessary when the agent has four explicitly assigned tools.

Architecture should reflect the use case.

### 5. Do Not Confuse Compensation With Remediation

Replacing an 8B model with a much larger model might have reduced visible failures.

It would not have demonstrated that the orchestration defect was understood.

### 6. Validate the Fix With a Controlled Before/After Test

A configuration change is not validated because the system "seems better."

The same capability should be retested under controlled conditions.

In this case, the relevant result was:

```text
Before:
Repeated malformed operations
Approximately 30 failures in one test
Task not completed reliably

After:
1 tool call
0 failures
Task completed successfully
```

---

## Design Principle Reinforced

This incident reinforced one of the broader principles behind the Opportunity Watcher:

> **The LLM is a reasoning component inside the system — not the system itself.**

Reliable autonomous systems require controls around the model.

Those controls include:

- deterministic policy
- constrained tools
- explicit trust boundaries
- evidence provenance
- failure handling
- least privilege
- observable execution
- human authority over consequential actions

The quality of the model matters.

The quality of the system surrounding the model matters just as much.

---

## Outcome

The investigation produced three concrete outcomes.

**Reliability improved.**

The local model could interact directly with the required OpenClaw tools without the previous retry behavior.

**The architecture became simpler.**

An unnecessary tool-discovery abstraction was removed from a four-tool agent.

**The diagnostic model improved.**

Future failures are now approached as distributed systems problems rather than automatically attributed to the LLM.

The final lesson is the same one that changed the direction of the investigation:

> **An apparent LLM failure may actually be an orchestration failure.**