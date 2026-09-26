# Models

How subagents pick models, and how to change that.

Builtin agents inherit your current Pi default model. This keeps new installs from depending on a provider you may not have configured. From there you can layer defaults and overrides:

- `subagents.defaultModel` — a default for every subagent that does not set its own model.
- `subagents.defaultProvider` — a provider preference for bare model ids, such as `llama-3`, when multiple providers expose the same id.
- `subagents.agentOverrides.<name>.model` — pin one role.
- `subagents.agentOverrides.<name>.defaultProvider` — choose or clear the provider preference for one role.
- `subagents.agentOverridesByProvider.<provider>.<name>` — layer role fields for the active parent provider.
- Per-run overrides — for one launch only.

Precedence, strongest first: per-run override → provider-scoped role override → `agentOverrides.<name>.model` → agent frontmatter `model` → `subagents.defaultModel` → the parent session model. A provider preference does not replace this order; it only resolves bare model ids when the active registry has more than one match. Fully qualified `provider/model` strings still win exactly.

Each launch resolves one model. Provider errors, including HTTP 429 responses, are returned from that model rather than selecting another one. Separately, a verified compaction abort after useful progress may continue the retained child session once on the same resolved model; this lifecycle recovery preserves work and is not model fallback.

Use `model: "inherit"` in agent frontmatter or `agentOverrides.<name>.model` to select the current parent session model explicitly.

## Local GLM-5.3-Flash on vLLM

Pi owns the OpenAI-compatible connection; pi-subagents inherits the **selected parent model** for its builtins. No extension-specific provider or model fork is needed. On the vLLM host, enable automatic tool calls and reasoning parsing for GLM-5.3-Flash (the [vLLM recipe](https://recipes.vllm.ai/zai-org/GLM-5.3-Flash) uses `--enable-auto-tool-choice --tool-call-parser glm47 --reasoning-parser glm47`). Check that your NVFP4 checkpoint, vLLM build, and GPU support those flags; the recipe's NVFP4 variant requires Blackwell. The client and every detached child must be able to reach the same server URL.

In `~/.pi/agent/models.json`, register the **exact served model ID** returned by that server's `/v1/models` (or set `--served-model-name` to the ID you choose). Replace the URL, ID, and token limit below with your deployment's actual values; 130K here is an example, not the hosted Go model's advertised 1M context. Pi's `contextWindow` must not exceed vLLM's configured `--max-model-len`:

```json
{
  "providers": {
    "local-glm": {
      "baseUrl": "http://YOUR_VLLM_HOST:8000/v1",
      "api": "openai-completions",
      "apiKey": "local",
      "models": [{
        "id": "YOUR_SERVED_MODEL_ID",
        "name": "GLM-5.3-Flash NVFP4 (local)",
        "reasoning": true,
        "input": ["text"],
        "contextWindow": 130000,
        "maxTokens": 16384,
        "compat": {
          "supportsDeveloperRole": false,
          "supportsStore": false,
          "maxTokensField": "max_tokens"
        },
        "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
      }]
    }
  }
}
```

If vLLM requires an API token, use `"apiKey": "$VLLM_API_KEY"` and provide that environment variable to **both** the parent and detached runner; never put the token in the JSON file. Test a normal tool call and a response after the tool returns before trusting long sessions. If your server rejects `reasoning_effort`, add `"supportsReasoningEffort": false` under `compat`; this disables Pi's effort field, not the checkpoint's own reasoning. Avoid pinning a Go model ID in subagent settings: `opencode-go/glm-5.3-flash` was useful for an external smoke test, but the local server can expose a different ID.

The [official GLM-5 guidance](https://github.com/zai-org/GLM-5#note) says Flash accepts `reasoning_effort` values `low`, `high`, and `max` (default `max`); other values fall back to `max`. The bundled `scout` asks for low and `worker` for high; check the **resolved** child model/effort in run status before assuming the local server honored them. For chat sessions the model authors advise `clear_thinking=true`; if your vLLM build supports `chat_template_kwargs`, test `"samplingParams": {"chat_template_kwargs": {"clear_thinking": true}}` as a model setting against multi-turn tool use before enabling it for a long job. The [official ZCode implementation](https://github.com/zai-org/ZCode) lists Flash as a model and pins child selection on resume, but its subagent routing is generic—not a model-specific mode to copy into Pi.

For long local jobs, start with these **optional** values, then tune them against measured queue and prefill latency. Set `httpIdleTimeoutMs` in Pi's `~/.pi/agent/settings.json` (or project `.pi/settings.json`), and the other values in `~/.pi/agent/extensions/subagent/config.json`:

Pi settings:

```json
{ "httpIdleTimeoutMs": 900000 }
```

pi-subagents config:

```json
{ "timeoutMs": 14400000, "checkpointBeforeDeadlineMs": 600000, "globalConcurrencyLimit": 2 }
```

The four-hour `timeoutMs` replaces the 30-minute default for a plain child; `checkpointBeforeDeadlineMs` asks an async single child to report state before the deadline, but is best-effort and cannot interrupt an active tool. A scripted async workflow has no default top-level deadline; put a suitable deadline on each child or an explicit workflow deadline when needed. Keep concurrency within your vLLM KV-cache capacity (two is a starting point, not a benchmark). Avoid hard tool-call caps for writers. Give each worker a bounded, verifiable slice, persist progress at stage boundaries, use mission/run artifacts to recover after parent compaction, and inspect or resume the **same** run rather than launching a second writer when it is slow. See [timeouts](configuration.md#timeoutms), [workflows](workflows.md), and [missions](missions.md).

Check `pi --list-models YOUR_SERVED_MODEL_ID`, then launch Pi with `--provider local-glm --model YOUR_SERVED_MODEL_ID`. Ask it to run one read-only scout child, then a two-child `runs.all` workflow; verify each child reports the local provider/model in `/subagents-fleet` or `subagent({action:"status",id:"..."})`, returns an actual file read, and reaches a terminal receipt. The Go smoke test can validate the Pi tool protocol, but cannot measure local NVFP4 latency, long decode stability, or a 130K-context overflow.

### Rehearse a 130K window with GLM-5.3-Flash on OpenCode Go

No context-limiting extension is needed. Pi's `models.json` can override the built-in Go model's advertised window for **both** a parent and children using the same Pi agent directory. Use an isolated profile so the rehearsal does not change your normal models:

```sh
export PI_CODING_AGENT_DIR="$(mktemp -d)"
cat > "$PI_CODING_AGENT_DIR/models.json" <<'JSON'
{"providers":{"opencode-go":{"modelOverrides":{"glm-5.3-flash":{"contextWindow":130000,"maxTokens":16384}}}}}
JSON
export OPENCODE_API_KEY="$OPENCODE_GO_API_KEY"
pi --list-models glm-5.3-flash  # opencode-go row: 130K context, 16.4K output
```

From the repository, run Pi with `--provider opencode-go --model glm-5.3-flash --no-extensions --extension "$PWD/index.ts" --no-context-files --no-session --mode json` and save the JSON event stream. Give the parent a broad repo question and authorize a **fresh scout** with the absolute repo cwd, then repeat the same question in a fresh direct-only session (`--no-extensions`, without this extension). For a controlled comparison, give the delegated parent `--no-builtin-tools --tools subagents_enable,subagent,bg_wait,grep` and the direct parent `--tools read,grep,find,ls`; allow the delegated parent a **file-scoped grep** to verify the scout's main citation and its exact line number. Check that the parent did not run bulk searches, the scout reached a completed receipt, and the cited line actually contains the claimed code. Compare the peak parent prompt per assistant response in each JSON stream:

```sh
jq -s '[.[] | select(.type=="message_end" and .message.role=="assistant") | .message.usage | .input + .cacheRead + .cacheWrite] | max' delegated.jsonl
```

The same expression applies to `direct.jsonl`. Also compare parent tool-result sizes, run count, and the child receipt's `totalTokens`/`windowPeak` in its async `status.json`. On one read-only model-resolution task in this orb, parent peak prompt was **8,999 tokens delegated vs 11,934 direct**; child `windowPeak` was **10,542**. A follow-up with citation verification peaked at **22,008 parent tokens** after GLM omitted the required `agent` field and fetched a long tool guide before retrying. These are variable single-run measurements, not an efficiency guarantee: children have their own context and total work can increase. The original scout also cited the right code at the wrong line; verify citations rather than trusting a handoff blindly. This setting controls Pi's context accounting and compaction threshold (default reserve 16,384, so compaction starts above about 113,616 tokens); it **does not cap the Go server's actual 1M window** or prove the 130K boundary was exercised. Test that boundary and your NVFP4 deployment on the local vLLM endpoint separately.

## Setting defaults and overrides

In `~/.pi/agent/settings.json` (user) or the project config settings file (`.pi/settings.json` in standard Pi; project wins):

```json
{
  "defaultModel": "deepseek-v4-pro",
  "subagents": {
    "defaultModel": "deepseek-v4-flash",
    "defaultProvider": "gpu-a",
    "agentOverrides": {
      "oracle": {
        "model": "deepseek-v4-pro"
      },
      "worker": {
        "defaultProvider": "gpu-b"
      }
    }
  }
}
```

To keep one role definition but configure it differently for work and personal providers, add the unambiguous provider map beside `agentOverrides`:

```json
{
  "subagents": {
    "agentOverrides": {
      "worker": { "thinking": "medium" }
    },
    "agentOverridesByProvider": {
      "github-copilot": {
        "worker": { "model": "github-copilot/gpt-5-mini" }
      },
      "openrouter": {
        "worker": { "model": "openrouter/openai/gpt-5-mini" }
      }
    }
  }
}
```

The provider key comes from the active parent session model (or an explicit host `preferredProvider`). Provider-scoped fields layer over the ordinary override in the same settings file; project settings still win over user settings.

For one run, put the override in the command:

```text
/run reviewer[model=anthropic/claude-sonnet-4:high] "Review this diff"
```

For a persistent role override:

```json
{
  "subagents": {
    "agentOverrides": {
      "reviewer": {
        "model": "anthropic/claude-sonnet-4",
        "thinking": "high"
      }
    }
  }
}
```

`subagents.defaultModel` and `subagents.defaultProvider` apply to builtin, package, user, project, and runtime-registered agents. `defaultModel` fills only agents that do not set `model` in frontmatter or in their runtime definition. `defaultProvider` is also applied to frontmatter and override models so bare ids resolve against the intended provider. Per-run model overrides and `agentOverrides.<name>.model` win over frontmatter and the global default. The same `agentOverrides` block can change `tools`, `skills`, inherited context, prompt text, or disable an agent (see [agents.md](agents.md)); matching custom-agent frontmatter is replaced for any field set by the override. Runtime-registered agents take only `model`, `defaultProvider`, `fast`, and `thinking` from `agentOverrides.<name>`; their other definition fields stay owned by the registering extension.

## Fast mode

Set `fast: true` on a run, in agent frontmatter, or in `subagents.agentOverrides.<name>.fast` to request the OpenAI priority service tier for supported native OpenAI-Codex children. This can use a higher quota tier or cost more. It is off by default.

Fast mode fails before launch unless the resolved model is a native `openai-codex/*` model. External runners, Anthropic models, and other providers do not use fast mode.

## Recommended model tiering (optional)

A setup that works well in practice: route agents by task shape instead of running everything on one model. Four tiers:

1. **Fast workhorse** — the cheapest capable model at low thinking, for recon, lookups, and mechanical edits. Example: `openai-codex/gpt-5.6-luna:low` on `scout`.
2. **Standard well-scoped** — a mid-tier model at medium thinking, for most delegations: routine multi-file edits, focused reviews, straightforward implementation. Example: `openai-codex/gpt-5.6-luna:max` on `worker`, `reviewer`, and a lightweight `delegate` agent.
3. **Deep but bounded** — a top reasoning model at high thinking, only for hard tasks that arrive with explicit goals and completion criteria. These models tend to loop on vague goals, so keep them off open-ended work. Example: `openai-codex/gpt-5.6-sol:high` on oracle-style agents.
4. **Taste and intent** — a model that reads human intent well and makes judgment calls without looping, for ambiguous work: UX and design decisions, product tradeoffs, planning from vague requirements, writing quality. Example: `anthropic/claude-fable-5` at `low` for lighter passes and `medium` for harder ones.

The routing rule: use the capability tiers (1–3) when the task is well-scoped, and the intent tier (4) when scoping or judging is the task itself.

Each launch resolves one model and starts the child once. Provider, authentication, quota, rate-limit, stream, empty-response, context-overflow, and provisioning failures are returned from that attempt. To try another model, the parent or operator must issue a later explicit launch.

## Thinking level defaults

Set `subagents.defaultThinking` to give builtin, package, user, and project agents without a `thinking` value a shared thinking level, independent of the parent session's default. Project settings win over user settings. Matching `agentOverrides.<name>.thinking` and per-run thinking overrides replace frontmatter; otherwise explicit frontmatter remains in effect. `thinking: false` remains an explicit opt-out:

```json
{
  "subagents": {
    "defaultThinking": "medium",
    "agentOverrides": {
      "reviewer": { "thinking": "high" }
    }
  }
}
```

If your provider rejects model IDs with thinking suffixes, set `subagents.disableThinking: true` in user or project settings. That clears bundled builtin thinking defaults in one place. An explicit higher-precedence `agentOverrides.<name>.thinking` value can opt a role back in or replace custom-agent frontmatter thinking.

### Thinking ceiling

Set `subagents.maxThinking` to enforce a hard maximum for every native Pi child. The supported levels, from least to most thinking, are `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, and `max`:

```json
{
  "subagents": {
    "defaultThinking": "medium",
    "maxThinking": "xhigh"
  }
}
```

Requests above the ceiling fail before child startup; the setting covers frontmatter, `agentOverrides`, per-run overrides, parallel/chain children, nested launches, and resumed children. Project settings take precedence over user settings. External runners retain their existing behavior.

## Extension defaults

Set `subagents.defaultExtensions` to give builtin, package, user, and project agents without an `extensions` field a shared extension allowlist:

- Absent: preserves Pi's normal ambient extension discovery.
- Empty array: sets `extensions: []` for agents that do not explicitly define it, disabling ambient extension loading.
- Non-empty array: supplies that allowlist to agents that do not explicitly define one.

Project settings win over user settings. Use `agentOverrides.<name>.extensions` for per-agent settings; a matching override replaces custom-agent frontmatter for that field.

```json
{
  "subagents": {
    "defaultExtensions": [],
    "agentOverrides": {
      "researcher": {
        "extensions": ["./tools/research.ts"]
      }
    }
  }
}
```

Set `subagents.defaultSubagentOnlyExtensions` to give agents without a `subagentOnlyExtensions` field a shared child-only extension list while preserving ambient extension discovery. An empty array is an explicit empty default but, unlike `defaultExtensions: []`, does not disable ambient extensions. An agent's frontmatter list (including `[]`) suppresses the default; user and then project `agentOverrides.<name>.subagentOnlyExtensions` replace it or clear it with `false`. Lists are not combined.

The two defaults resolve independently, with an explicitly present project value winning over the user value. If both are set, `defaultExtensions` still disables ambient discovery and the child-only paths are loaded alongside its allowlist. Both reject non-arrays, non-string or blank entries with an error naming the setting and source file. Extension paths execute trusted code, so use project defaults only for trusted repositories and extensions.

## Inspecting the live mapping

To see what `pi-subagents` has actually loaded right now:

```text
/subagents-models
/subagents-models reviewer
```

That reports the live runtime mapping, which can differ from settings on disk until you reload Pi.

## Fuzzy model matching

You do not have to spell a model exactly. Model ids are matched fuzzily against the registry, so these all resolve to the same model:

- Provider separator variations: `anthropic/claude-sonnet-4`, `anthropic:claude-sonnet-4`, `anthropic.claude-sonnet-4`
- Id separator variations: `claude-haiku-4.5` vs `claude-haiku-4-5`
- Case differences: `Claude-Sonnet-4` vs `claude-sonnet-4`
- Optional trailing date stamps: `claude-haiku-4-5-20251001` or `claude-haiku-4-5-2025-10-01` vs `claude-haiku-4-5`

Exact `provider/id` matches still win, and a qualified provider query never silently switches providers — it only matches within the named provider. Ambiguous bare ids that exist under multiple providers still require a provider prefix or the current session's provider to disambiguate.

Registry ids that themselves contain `/` (Hugging Face `owner/name`) resolve the same way as Pi's main agent: `thinkingmachines/Inkling` becomes `huggingface/thinkingmachines/Inkling` when that id is unique or offered by the current session provider. A first path segment that matches a registered provider still means `provider/id`.

## Model scope enforcement

To keep subagents inside a budget or compliance profile, enforce a model scope. Put `subagents.modelScope` in user or project settings (project overrides user):

```json
{
  "subagents": {
    "modelScope": {
      "enforce": true,
      "strict": true,
      "allow": ["inherit", "openai/gpt-5-*", "openai-codex/gpt-5.6-*"],
      "agents": {
        "worker": { "allow": ["openai-codex/gpt-5.6-luna"] },
        "reviewer": { "allow": ["inherit"] }
      }
    }
  }
}
```

- `allow` is a list of glob patterns matched against the resolved `provider/id` (only `*` is special, case-insensitive). The literal `inherit` means the current parent session model.
- `agents.<name>` adds a second allow-list for that agent. The model must pass both the global list and the matching agent list, so an agent rule cannot weaken the global rule. Agent rules inherit `enforce` and `strict` when those fields are absent.
- A top-level `enforce: true` with only agent allow-lists restricts only those named agents. Unknown names are allowed so settings can be shared across projects and machines.
- Models you pass explicitly — the tool-call `model`, `--model`, or a clarify pick — error and abort the run.
- By default, models from agent frontmatter, `subagents.defaultModel`, or the inherited parent session model only warn and remain available, so existing configurations keep working while you tighten the scope.
- Set `strict: true` with `enforce: true` to reject every resolved out-of-scope model, including inherited models.
- `enforce: true` requires at least one non-empty global or agent `allow` list; otherwise the config is rejected at load time.

Model scope is policy only. It rejects or warns; it does not select a cheaper model. Set `agentOverrides.worker.model` to choose a worker model and use `modelScope.agents.worker` to prevent a per-run override from escaping that restriction.

`inherit` expands in the parent process at each launch. It is never sent to the child as a model id. A nested child therefore inherits its immediate parent's current model, not the original top-level model. If no parent model is available, an enforced `inherit` entry does not match and fails closed.

Project `modelScope` settings replace the complete user `modelScope`, as with the existing project-over-user settings precedence. Project settings are trusted and can therefore replace user restrictions.

## Profiles and provider model catalogs

Profiles let you generate and save role-to-model assignments from a provider's live catalog.

Profiles are stored under:

```text
~/.pi/agent/profiles/pi-subagents/
```

Provider model catalogs are cached under:

```text
~/.pi/agent/profiles/pi-subagents/providers/
```

The workflow:

```text
/subagents-refresh-provider-models openai-codex
/subagents-generate-profiles openai-codex
/subagents-load-profile openai-codex.quota
```

- `/subagents-refresh-provider-models` writes a serialized provider model catalog with observed registry data, simple role-oriented classification, and live probe results from tiny one-shot `pi -p --model ... --no-tools` checks. The cache refreshes when missing or stale; use `--force` to ignore freshness and probe again immediately.
- `/subagents-generate-profiles` uses the provider catalog to produce quota and quality profiles.
- `/subagents-check-profile` re-checks each assigned model in a saved profile against the current registry and a live probe, so you can detect model removals, auth problems, or stale assignments.
