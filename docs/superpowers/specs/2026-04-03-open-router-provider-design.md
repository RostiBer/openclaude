# Design: OpenRouter Provider Profile

**Date:** 2026-04-03  
**Status:** Approved

## Summary

Add `open-router` as a dedicated named provider profile alongside the existing `openai`, `ollama`, `codex`, `gemini`, and `atomic-chat` profiles. OpenRouter is OpenAI-compatible, so it reuses the same transport and env-vars — the profile is a thin wrapper that pre-fills `OPENAI_BASE_URL=https://openrouter.ai/api/v1` and guides the user through entering an API key and model name.

## Architecture

### Type Changes (`src/utils/providerProfile.ts`)

- Add `'open-router'` to the `ProviderProfile` union type.
- Add constant `DEFAULT_OPENROUTER_BASE_URL = 'https://openrouter.ai/api/v1'`.
- Add `buildOpenRouterProfileEnv(options: { model?, apiKey?, processEnv? }): ProfileEnv | null` — identical shape to `buildOpenAIProfileEnv` but always sets `OPENAI_BASE_URL` to `DEFAULT_OPENROUTER_BASE_URL`.
- Update `isProviderProfile()` to accept `'open-router'`.
- Add `if (profile === 'open-router')` branch in `buildLaunchEnv()` that sets `CLAUDE_CODE_USE_OPENAI=1`, `OPENAI_BASE_URL=DEFAULT_OPENROUTER_BASE_URL`, `OPENAI_MODEL`, and `OPENAI_API_KEY`.

### Wizard Changes (`src/commands/provider/provider.tsx`)

- Add `'open-router'` to the `ProviderChoice` type and import `buildOpenRouterProfileEnv`.
- Add two new `Step` variants: `{ name: 'openrouter-key' }` and `{ name: 'openrouter-model'; apiKey: string }`.
- Add option to `ProviderChooser` Select:
  ```
  label: 'OpenRouter',
  value: 'open-router',
  description: 'Access Claude, Gemini, DeepSeek, and others via openrouter.ai'
  ```
- Add two `TextEntryDialog` wizard steps:
  1. **openrouter-key** — prompt for API key (`sk-or-...`), same validation as openai-key step.
  2. **openrouter-model** — prompt for model name, placeholder `google/gemini-2.0-flash-001`.
- Add `case 'open-router'` in `buildSavedProfileSummary()` showing label `'OpenRouter'`, model, endpoint from `DEFAULT_OPENROUTER_BASE_URL`.
- Update `buildCurrentProviderSummary()` to detect OpenRouter base URL and label it `'OpenRouter'` (similar to how Ollama is detected by `localhost:11434`).

### No Changes Needed

- `src/utils/model/providers.ts` — `getAPIProvider()` already returns `'openai'` for `CLAUDE_CODE_USE_OPENAI=1`, which covers OpenRouter transport.
- `src/utils/providerFlag.ts` — no change needed.

### Docs (`docs/advanced-setup.md`)

- Promote the existing "Google Gemini via OpenRouter" section to a standalone "OpenRouter" section.
- Add a note that any OpenRouter model can be used (not just Gemini).

## Data Flow

```
User runs /provider
  → chooses 'OpenRouter'
  → enters API key (sk-or-...)
  → enters model (e.g. anthropic/claude-sonnet-4-5)
  → buildOpenRouterProfileEnv() called
  → ProfileFile saved: { profile: 'open-router', env: { OPENAI_API_KEY, OPENAI_BASE_URL: 'https://openrouter.ai/api/v1', OPENAI_MODEL } }

OpenClaude starts
  → buildLaunchEnv() sees profile === 'open-router'
  → sets CLAUDE_CODE_USE_OPENAI=1, OPENAI_BASE_URL, OPENAI_MODEL, OPENAI_API_KEY
  → getAPIProvider() returns 'openai'
  → chat_completions transport used
```

## Error Handling

- If API key is blank/placeholder: same validation as `openai` profile — reject with error message.
- If model is blank: use placeholder `google/gemini-2.0-flash-001` as default.
- `buildOpenRouterProfileEnv()` returns `null` if no valid API key — wizard shows error.

## Testing

Existing tests in `src/utils/providerProfile.test.ts` and `src/commands/provider/provider.test.tsx` cover the profile save/load and wizard flow patterns. New cases should be added mirroring the `openai` profile tests but with `'open-router'` profile and `DEFAULT_OPENROUTER_BASE_URL`.
