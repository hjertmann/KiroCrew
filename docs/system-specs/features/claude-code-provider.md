# Standalone provider — removed

> **This provider no longer exists in KiroCrew.** The public fork drives a single
> backend — `kiro-cli` over the Agent Client Protocol (`agent.provider` is fixed
> to `acp`). The removed standalone provider, the removed Bedrock provider, the
> removed agent-renderer / mirror modules, their config fields, and the dashboard
> provider selector were all removed during de-Amazoning. There is no provider to
> choose.

## What remains (the dormant ACP seam)

`acp/client.py` keeps an inert protocol seam (`ACP_BACKEND_CLAUDE` / the
`_is_claude` branch) so an internal companion package can re-register an
alternate `claude-agent-acp` backend without forking the client. The public core
never selects it: `_resolve_kiro_bin` is the only backend the provider factory
wires, and the dashboard exposes no provider choice. **Do not re-add the
registration glue or a provider selector** — see the repo-root `CLAUDE.md`.

The seam's binary-resolution details (`_resolve_claude_acp_bin`, the per-session
`settings.local.json` permission routing, `CLAUDE_CONFIG_DIR` isolation) are
documented in [`acp-client.md`](../modules/acp-client.md) for the companion that
re-enables it — they are not user-facing in the public build.

## Model registry

`model_registry.json` (loaded by `model_registry.py`, mirrored to
`website/src/model_registry.json` and guarded by `test/test_model_registry_parity.py`)
remains the single source of truth for model names and context windows. Its
per-entry `providers.claude_code` value is what `claude-agent-acp` accepts for its
`model` config option: a **tier** name, not a versioned id — `sonnet`, `opus`,
`opus[1m]`, `haiku`, `claude-fable-5[1m]`, `default`. Because the vocabulary
carries no version, several canonicals share one wire value (`claude-sonnet-4.5`
→ `sonnet`) and a version-only entry carries `""`, which resolves to the backend's
own pick rather than relabelling a neighbouring version (Opus 4.7 is not
`opus[1m]`). The Bedrock-style `global.anthropic.*` spellings an internal
companion uses are kept as `aliases`, so anything persisted in that form still
folds onto a canonical key.

The accepted set is advertised per session in the `session/new` response
(`configOptions` with `id: "model"`). `_apply_startup_model` withholds any
configured model absent from it and stays on the backend default, because
`session/set_config_option` answers an unaccepted value with a `-32603` that
aborts session **init** — a config still holding a kiro-cli id after a provider
switch would otherwise make every session unstartable.

Canonical keys include `fable-5-1m` (leads the JSON but is **not** default —
Opus 4.8 `opus-4.8-1m` remains the sole `"default": true` entry), and
`available_models()` (`model_registry.py`) is default-first-ordered, so a
non-default entry sitting ahead of Opus in the file cannot change the
`auto`-path pick. Each entry's `aliases` also include the bare, prefix-stripped
provider-id spelling (e.g. `claude-fable-5`) so `from_provider_id` folds bare
ids onto the canonical key.

One collision is deliberate: `opus` is both an alias on `opus-4.8-1m` (for a
hand-typed `agent.model`) and the wire value of the 200K `opus-4.8`. A provider id
outranks an alias in `_build_indices`, so on `claude_code` the string resolves to
the 200K row — which is what the backend means by `opus`, `opus[1m]` being the 1M
one. On `acp` it still resolves to `opus-4.8-1m`.

**A tier without `[1m]` is a 200K session, and the row that carries it must say
200K.** The backend advertises no window, so this is the only number the context
budget has until the first `usage_update.size`. Two rules follow:

- `_registry_window` resolves a `claude_code` wire value (`_CC_PROVIDER_IDS`) in
  the `claude_code` index FIRST, ahead of its normal acp-first order. `opus` and
  `sonnet` are also bare aliases of the 1M canonicals on the kiro side, so the
  acp-first walk reported 1M for a 200K session and the budget over-assembled.
- Exactly ONE row may carry a given wire value: `_build_indices` **assigns**
  provider ids (aliases only `setdefault`), so a second claimer silently wins by
  declaration order. The 200K `sonnet-4.5` is therefore the sole claimer of
  `sonnet`, and it folds the Sonnet-class spellings so a `4.6` pin still lands
  on `sonnet` rather than on `""` — `""` would hand back the org default, which can
  be Opus, a cost surprise for someone who asked for the cheap tier.
  `test_one_row_per_non_empty_provider_id` enforces this rule rather than leaving
  it documented.
- **A fold belongs in `provider_aliases.<provider>`, never in the shared
  `aliases` list.** `aliases` is one namespace applied to EVERY provider in the
  row, so a spelling added to make one backend fold also re-points it on the
  other's index — and the winner is decided by declaration order. Putting the 1M
  Sonnet spellings (`claude-sonnet-4.6-1m`, `claude-sonnet-4-6[1m]`) in
  `sonnet-4.5`'s shared list shrank kiro's 1M Sonnet to 200K, because those two
  are not a provider id of any row and so had nothing to outrank the alias.
  Deleting them is not the fix either: that restores kiro but regresses
  `claude_code` to `""`. `_build_indices` unions `aliases` with only the current
  provider's `provider_aliases` entry, so a per-backend fold cannot mutate a
  sibling's canonical resolution.
  `test_every_sonnet_spelling_keeps_its_window_on_acp` pins every spelling on
  both backends — the dotted one alone passed while the `[1m]`/`-1m` forms were
  broken.

The row's own version label being stale for this backend is harmless, because the
picker takes display name and description from the live advertised list; only the
window is read from the registry.

Still inaccurate, deliberately: `auto` reports 1M, from the kiro `--list-models`
cache (kiro serves `auto` at 1M). On this backend `auto` resolves to the org
default, whose window is unknown until the first turn, and `model_window` has no
provider argument to scope that cache by. The live `usage_update.size` corrects it
per turn.
