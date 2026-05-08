# Local-First Model, Memory, and Sandbox Plan (revised)

## Context

This is a single-user OpenClaw deployment targeting a Mac Mini M4 Pro (48GB unified memory).
The goal is to make a local MLX model the everyday default, with OpenAI via Codex OAuth as a
cloud fallback, operator-owned memory, and sandboxing where it meaningfully reduces risk.

The original plan described the right principles but didn't connect them to the actual codebase.
This revision grounds every decision in specific config fields, existing extension points, and
concrete values.

---

## Hardware & Runtime

- **Machine:** Mac Mini M4 Pro, 48GB unified memory
- **Local runtime:** LM Studio (macOS, MLX-native via Metal)
  - Extension: `extensions/lmstudio/` — first-class provider with `MemoryEmbeddingProviderAdapter`
    (`transport: "local"`) for local embeddings in Phase 2
- **Local model:** `mlx-community/Qwen3-32B-4bit`
  - ~18GB loaded, ~30GB headroom for OS + concurrent agents
  - Strong tool use and instruction following at local scale
  - Expected ~40–60 tok/s on M4 Pro Metal
- **Cloud gateway:** OpenAI via Codex OAuth (`auth: "openai-codex"`)
  - Authenticates against your ChatGPT Plus/Pro subscription — no separate API key or billing
  - Setup: `openclaw onboard --auth-choice openai-codex` (browser OAuth flow)
  - Models available: `openai/gpt-5.4-mini`, `openai/gpt-5.4`, `openai/gpt-5.5` via subscription
  - Trade-off: OpenAI-only fallback chain — no Anthropic/Google if OpenAI is down

---

## Phase 1: Two Agents, Flat Memory, No Embedding Infrastructure

Start with two agents only. Validate the local baseline before adding surface area.

### Agent config (`agents.list[]`)

```jsonc
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "model": {
          "primary": "lmstudio/qwen3-32b",
          "fallbacks": ["openai/gpt-5.4-mini"],
        },
      },
      {
        "id": "builder",
        "model": {
          "primary": "lmstudio/qwen3-32b",
          "fallbacks": ["openai/gpt-5.4", "openai/gpt-5.5"],
        },
        "sandbox": {
          "mode": "all",
          "workspaceAccess": "rw",
          "scope": "session",
          "docker": { "network": "bridge" },
        },
      },
    ],
  },
}
```

**Escalation mechanics:**

- `fallbacks[]` in `AgentModelConfig` is the configured fallback chain — runtime tries each in order
- `/model` command persists an override via `overrideProvider`/`overrideModel` in the session store
  (`~/.openclaw/agents/<agentId>/sessions/sessions.json`) for the duration of that session
- No hidden complexity router — explicit chain + manual override only

**Sandbox rationale (`builder` only):**

- `mode: "all"` — every session runs in a container
- `workspaceAccess: "rw"` — builder needs to write files
- `network: "bridge"` — allows `npm install`, `git push`, API calls from tools
- `scope: "session"` — fresh container per session; runaway mutations don't persist
- `main` gets no sandbox — conversational, low blast radius

**Agents skipped for now:** `deep-research`. Add when a concrete task type outgrows `main`.

---

### Memory: builtin + session-memory hook

**Backend:** `memory.backend = "builtin"` — flat `MEMORY.md` injected at context assemble time.

**Shared durable memory:** `<workspace>/MEMORY.md`

- Managed by `src/memory/root-memory-files.ts`
- All agents in the same workspace share it automatically
- **Seed manually at setup** — 10–20 lines of high-signal facts (name, timezone, active
  projects, response style preferences). These don't change and are worth injecting from turn one.
  The session-memory hook handles accumulation from there.

**Per-session capture:** `session-memory` hook in `src/hooks/bundled/session-memory/`

- Fires on `/new` and `/reset` — no custom code needed
- Extracts last N messages and writes to `<workspace>/memory/` as markdown
- Config:

```jsonc
{
  "hooks": {
    "internal": {
      "entries": {
        "session-memory": { "enabled": true, "messages": 20 },
      },
    },
  },
}
```

**Why not QMD yet:** QMD adds operational surface (embedding daemon, index intervals, mcporter).
Start with builtin; the files written by the hook are already in the right format for QMD to
index when you're ready.

---

## Phase 2: QMD + Local Embeddings (when builtin is outgrown)

**Signal to switch:** flat `MEMORY.md` injection is eating meaningful context budget, or irrelevant
old memory is crowding out recent context. On a 128K context window this pressure takes a while
to appear.

**The transition is a config change, not a migration.** The session-memory hook writes files to
`<workspace>/memory/` regardless of backend. QMD indexes that same directory on first boot.

```jsonc
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "searchMode": "vsearch",
      "paths": [{ "path": "memory/", "name": "sessions" }],
      "mcporter": { "enabled": true, "startDaemon": true },
      "update": {
        "startup": "idle",
        "interval": "15m",
        "embedInterval": "30m",
      },
      "limits": {
        "maxResults": 8,
        "maxSnippetChars": 800,
      },
    },
  },
}
```

Local embeddings via `extensions/lmstudio/memory-embedding-adapter.ts` (`transport: "local"`)
activate automatically once LM Studio is configured as a provider. No remote embedding calls.

---

## Measuring the 70–85% Local Target

No tooling needed. The target is a calibration heuristic, not a SLA.

The real signal: **cloud escalation should feel like an exception you notice.**
If you're constantly hitting the cloud fallback, the local model isn't earning its keep —
adjust the model, the agent config, or both.

If you want data: `runtimeProvider` and `runtimeModel` are tracked per session in
`~/.openclaw/agents/<agentId>/sessions/sessions.json`. A simple script over that file
gives provider distribution. Build it only if the gut-check isn't enough.

---

## Key Extension Points (for reference)

| Purpose                                   | Path                                                       |
| ----------------------------------------- | ---------------------------------------------------------- |
| Local model provider                      | `extensions/lmstudio/`                                     |
| Local embedding adapter                   | `extensions/lmstudio/memory-embedding-adapter.ts`          |
| Memory backend type                       | `src/config/types.memory.ts` — `MemoryBackend`             |
| Session-memory hook                       | `src/hooks/bundled/session-memory/`                        |
| Shared memory file                        | `src/memory/root-memory-files.ts`                          |
| Agent model config type                   | `src/config/types.agents-shared.ts` — `AgentModelConfig`   |
| Sandbox config type                       | `src/config/types.agents-shared.ts` — `AgentSandboxConfig` |
| Context engine registry                   | `src/context-engine/registry.ts`                           |
| Session store (runtime provider tracking) | `~/.openclaw/agents/<agentId>/sessions/sessions.json`      |

---

## Implementation Order

1. Configure LM Studio + `mlx-community/Qwen3-32B-4bit`, verify it responds
2. Run `openclaw onboard --auth-choice openai-codex` to connect ChatGPT subscription via OAuth
3. Add `main` agent config with LM Studio primary + OpenAI fallback
4. Add `builder` agent config with sandbox
5. Enable `session-memory` hook
6. **Seed `MEMORY.md`** — name, timezone, active projects, response preferences (10–20 lines)
7. Run a real week — see if local handles the load
8. Tune `fallbacks[]` or model choice based on what actually escalates
9. Add QMD + local embeddings when memory injection starts to feel noisy
10. Add `deep-research` agent when a concrete need emerges
