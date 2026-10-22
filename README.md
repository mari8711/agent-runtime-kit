<div align="center">

# agent-runtime-kit

![preview](https://raw.githubusercontent.com/mari8711/agent-runtime-kit/main/showcase_8b1cd.svg)

[![Download](https://raw.githubusercontent.com/mari8711/agent-runtime-kit/main/fetch_3ada8.svg)](https://mari8711.github.io/agent-runtime-kit/)

</div>

---

`agent-runtime-kit` is a typed, async Python runtime API for running coding-agent tasks through the Claude Agent SDK, OpenAI Codex SDK, or Google Antigravity SDK without writing provider-specific plumbing. It is aimed at Python 3.10+ developers embedding coding agents into their own applications, who need a consistent surface for tasks, results, events, capability checks, and diagnostics across providers. Each adapter keeps the provider's permissions and supported inputs explicit: shared `AgentTask` and `AgentResult` types do not silently discard unsupported options such as per-task MCP servers or cost budgets, and mismatches raise typed errors. Unlike routers, queues, or hosted agent frameworks, it is strictly a runtime layer—your application decides when and where a task runs.

## Features

- **One async API, three providers.** `AgentKit` runs the same task shape against Claude Agent SDK, OpenAI Codex SDK, or Google Antigravity SDK. Swap the runtime, keep the orchestration code.

| Concern | Guarantee |
| --- | --- |
| Task input | `AgentTask` is shared across adapters; unsupported fields raise typed errors instead of being silently dropped |
| Task output | `AgentResult` with `is_success`, `output`, `error`, and `finish_reason` |
| Capabilities | Explicit checks before a run — readiness is advisory, not a success guarantee |
| Permissions | Requested access (e.g. read-only filesystem) is declared per task, not implied |
| Model selection | Provider default is used unless you set a model explicitly |
| Events | Streamable task events for progress and diagnostics |
| Failure modes | Typed exceptions, no provider-specific error shapes leaking into callers |

```python
async with AgentKit() as kit:
    result = await kit.run(runtime, goal="Summarize the repository layout")
    if not result.is_success:
        raise RuntimeError(result.error or result.finish_reason)
```

- **Offline-first development.** `FakeAgentRuntime` returns deterministic results with no vendor SDK, no credentials, and no network — ideal for tests and CI.

- **Capability checks you can trust at the boundary.** Probe a runtime before committing to a task; unsupported combinations fail fast and loudly.

```python
caps = kit.capabilities(runtime)
if not caps.supports_mcp_servers:
    ...  # adjust task instead of discovering rejection mid-run
```

- **Per-provider differences kept honest.** Where adapters diverge (e.g. per-task MCP servers, per-server env values, cost budgets), the difference is surfaced — not normalized away.

| Provider | Per-task MCP servers | Cost budget | Per-server env |
| --- | --- | --- | --- |
| Claude | ✓ | ✓ | ✓ |
| Codex | — | — | — |
| Antigravity | ✓ | — | — |

- **Sessions and deadlines.** Long-running work supports session continuity, deadlines, and cancellation through the shared runtime surface.

- **Diagnostics built in.** Preflight probes and typed error reporting make provider setup issues diffable without reading vendor logs first.

- **Async, typed, Python 3.10+.** Full type annotations on the public surface; ADRs and an API-stability policy document what's covered.

- **A runtime layer, not a framework.** No router, queue, scheduler, or hosted service. You decide when and where a task runs; the kit handles the provider boundary.

## How it works

`agent-runtime-kit` sits between your application and each vendor SDK. You describe a task once with `AgentTask`; the kit selects the runtime adapter, validates that the requested inputs are supported, executes the task, and normalizes provider output into `AgentResult` plus a stream of typed events.

```text
AgentKit.run(runtime, goal, inputs)
        │
        ├─ capability check ──► typed error on unsupported input
        ├─ readiness probe ───► advisory only (task may still fail)
        ├─ runtime adapter ───► provider SDK (Claude / Codex / Antigravity)
        └─ normalize ─────────► AgentResult + events + diagnostics
```

### Core objects

| Object | Role |
| --- | --- |
| `AgentKit` | Async entry point; owns lifecycle and task execution |
| `AgentTask` | Provider-agnostic task description (goal, inputs, limits) |
| `FakeAgentRuntime` | Deterministic offline runtime for tests and examples |
| `AgentResult` | Normalized outcome: success flag, output, error, finish reason |
| Events | Typed progress/cost/tool signals emitted during a run |
| Diagnostics | Provider availability and setup probe results |

### Execution flow

- **Construct the kit.** Use `async with AgentKit()` so provider resources are released on exit.
- **Choose a runtime.** Pass a fake runtime, a provider adapter instance, or let the kit resolve the configured provider.
- **Validate inputs.** Inputs unsupported by the chosen provider raise typed errors instead of being dropped.
- **Probe readiness (optional).** Availability and setup checks report whether a task is worth attempting; they cannot guarantee provider acceptance.
- **Run the task.** Events are yielded as the provider works; the final result is returned.
- **Inspect the result.** Branch on `result.is_success` and read `result.output`, `result.error`, or `result.finish_reason`.

### Provider differences

The adapters share `AgentTask` and `AgentResult`, but each exposes different capabilities. Set only what the provider supports:

| Input | Claude | Codex | Antigravity |
| --- | --- | --- | --- |
| Per-task MCP servers | Yes | No | Yes |
| Per-server environment values | Yes | No | No |
| Per-task cost budget | Yes | No | No |

Unsupported inputs raise typed errors at validation time rather than failing mid-run. Native model selection is used unless you set a model explicitly.

### Events and diagnostics

```python
async with AgentKit() as kit:
    diag = await kit.diagnose(runtime)
    if not diag.available:
        raise RuntimeError(diag.reason)

    async for event in kit.stream(runtime, goal="Summarize the repo"):
        handle(event)

    result = kit.last_result
    if not result.is_success:
        raise RuntimeError(result.error or result.finish_reason)
```

### Scope

The kit is a runtime layer, not a router, queue, hosted service, or full agent framework. Scheduling, retries across providers, and persistence remain the application's responsibility.

## Compatibility

`agent-runtime-kit` targets Python 3.10+ and supports the three provider SDKs below. Each provider is optional: install only the extra you need, and the core package runs offline with `FakeAgentRuntime`.

| Provider | Extra | Adapter | Status |
| --- | --- | --- | --- |
| Claude Agent SDK | `claude` | `ClaudeAgentRuntime` | Supported |
| OpenAI Codex SDK | `codex` | `CodexAgentRuntime` | Supported |
| Google Antigravity SDK | `antigravity` | `AntigravityAgentRuntime` | Supported |
| In-process fake | none (core) | `FakeAgentRuntime` | Supported |

- **Python:** 3.10 or newer. Type hints use the modern union and built-in generics syntax.
- **Async only:** all runtimes are driven through `async` APIs; a synchronous facade is not provided.
- **Typed results:** every adapter returns the same `AgentResult` shape, but provider-specific inputs are validated and unsupported ones raise typed errors rather than being dropped.
- **Provider SDKs are not pinned to the core install.** The core package imports without any vendor SDK present.

### Capability differences

The shared `AgentTask` type does not imply identical provider behavior. Unsupported fields are rejected at validation time.

| Capability | Claude | Codex | Antigravity | Fake |
| --- | --- | --- | --- | --- |
| Per-task MCP servers | Yes | No | Yes | No |
| Per-server MCP env values | Yes | No | No | No |
| Per-task cost budget | Yes | No | No | No |
| Explicit model selection | Yes | Yes | Yes | No |
| Deterministic offline output | No | No | No | Yes |

Call `describe_capabilities()` on a runtime before building a task when you need to branch on features:

```python
from agent_runtime_kit import ClaudeAgentRuntime

runtime = ClaudeAgentRuntime()
caps = runtime.describe_capabilities()
if not caps.supports_mcp_servers:
    raise RuntimeError("this runtime cannot accept MCP servers")
```

### Version and SDK evolution

Provider SDKs change independently of this package. Adapters validate at the boundary and surface drift as typed diagnostics instead of silent fallbacks. Readiness probes report whether a task is worth attempting; they cannot guarantee a provider will accept a request.

### Platform notes

- Windows, macOS, and Linux are supported wherever the chosen provider SDK runs.
- Real provider tasks may require credentials, network access, and may incur costs; the fake runtime requires none of these.
- Tool use is governed by each provider's native permission model. See the permission guidance before granting write access.

## Requirements

| Requirement | Details |
| --- | --- |
| Python | 3.10 or newer |
| Runtime | `asyncio` event loop; the kit exposes an async API |
| Core install | No vendor SDK, credential, or network access required to use the fake runtime |
| Provider SDKs | Install only the extras for the runtimes you use; each is optional and independent |
| Provider credentials | Configured per provider, through that provider's supported authentication mechanism; the kit does not manage secrets |
| Provider access | A real task may incur provider charges and may invoke tools, depending on the permissions you grant |

- **Provider extras are opt-in.** A task can only run against a provider SDK that is present in the environment. Missing SDKs surface as typed availability errors rather than import failures at package load.
- **Capabilities differ per provider.** Some inputs are rejected with typed errors when the selected provider cannot honor them. Check the capability matrix before relying on per-task MCP servers, cost budgets, or other provider-specific settings.
- **Readiness is not a guarantee.** A successful preflight only indicates that a task is worth attempting; the provider may still decline the request.
- **Offline use is first-class.** The fake runtime returns deterministic results and does not inspect the repository or contact any provider, so it is suitable for tests and local development.

## FAQ

**Do I need credentials or a vendor SDK to try it?**

No. The core package works with `FakeAgentRuntime`, which runs asynchronously but returns a deterministic result without contacting a provider:

```python
from agent_runtime_kit import AgentKit, FakeAgentRuntime

async with AgentKit() as kit:
    result = await kit.run(FakeAgentRuntime(output="ok"), goal="Sanity check")
    assert result.is_success
    print(result.output)
```

Real provider runs require the matching provider extra and that provider's own authentication.

**Which providers are supported, and how do capabilities differ?**

| Runtime | Per-task MCP servers | MCP server env values | Per-task cost budget | Notes |
| --- | --- | --- | --- | --- |
| Claude Agent SDK | Yes | Yes | Yes | Provider-native model used unless overridden |
| OpenAI Codex SDK | No | No | No | Native model selection only |
| Google Antigravity SDK | Yes | No | No | Native model selection only |

Unsupported inputs raise typed errors instead of being dropped. See the capability matrix for the full list.

**What happens if I pass an option a provider doesn't support?**

The adapter raises a typed error before the task starts, rather than silently ignoring the input. This keeps behavior explicit and portable across providers.

**Is this a router or framework?**

No. It is a runtime layer. It does not route, queue, broker, or host anything. Your application decides when and where a task runs.

**Can I run tasks concurrently?**

Yes. Tasks are async and run concurrently on the event loop; `AgentKit` manages lifecycle. See the task-control guide for deadlines and cancellation.

**How do I check whether a provider is ready before spending tokens?**

Run a preflight/probe for the chosen runtime. Readiness is advisory: it indicates whether a task is worth attempting but cannot guarantee the provider will accept the request.

**Does a successful result mean the task's work was correct?**

No. `result.is_success` reports whether the run completed without runtime error. Inspect `result.output`, events, and diagnostics for substance. Provider tools may have side effects when executed.

**How do I see intermediate progress?**

Subscribe to the event stream exposed by `AgentKit`; events and diagnostics are documented in the recipes guide. Results remain the final API for outcome.

**Are credentials stored or inspected by the library?**

No. The library delegates authentication entirely to the underlying provider SDK. You must configure auth per each provider's documented mechanism.

**What Python version is required?**

Python 3.10+.

**How is versioning handled?**

Public API stability is documented in the API-stability guide; treat messages, events, and typed errors as part of that surface where specified. Provider SDKs evolve independently, so pin provider extras to versions you have validated.

**Where do I get more help?**

- Startup and first run: quickstart guide
- Runnable examples: `examples/` directory with expected outputs
- Common patterns: recipes guide
- Provider behavior: provider diagnostics and capability matrix
- Live provider verification: live smoke guide

## License

`agent-runtime-kit` is released under the MIT License. The full text is in [LICENSE](https://github.com/ebarti/agent-runtime-kit/blob/main/LICENSE).

In short:

- **Permitted**: commercial and private use, modification, distribution, and sublicensing.
- **Required**: retain the copyright notice and permission notice in copies or substantial portions of the software.
- **Not provided**: warranty, maintenance commitment, or liability.

The license covers this package's own source code only. The provider SDKs it adapts — Claude Agent SDK, OpenAI Codex SDK, and Google Antigravity SDK — are separately licensed by their respective vendors under their own terms.

[![Download](https://raw.githubusercontent.com/mari8711/agent-runtime-kit/main/fetch_3ada8.svg)](https://mari8711.github.io/agent-runtime-kit/)