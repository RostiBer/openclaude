# OpenRouter Provider Profile Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `open-router` as a dedicated named provider profile with its own `/provider` wizard flow.

**Architecture:** `open-router` reuses the OpenAI-compatible transport (`CLAUDE_CODE_USE_OPENAI=1`) with a hardcoded base URL of `https://openrouter.ai/api/v1`. It follows the same pattern as `atomic-chat`: thin wrapper that pre-fills the URL and collects API key + model from the user. Four files change: `providerProfile.ts` (types + logic), `provider.tsx` (wizard UI), `provider-launch.ts` (CLI dev script), `provider-bootstrap.ts` (CLI profile init script).

**Tech Stack:** TypeScript, Node.js built-in `test` module (providerProfile tests), Bun test + React/Ink (provider wizard tests)

---

## File Map

| File | Change |
|---|---|
| `src/utils/providerProfile.ts` | Add type, constant, builder, launch branch |
| `src/utils/providerProfile.test.ts` | Add `open-router` launch + builder tests |
| `src/commands/provider/provider.tsx` | Add wizard steps + chooser option + summary case |
| `scripts/provider-launch.ts` | Add `open-router` to arg parsing + launch validation |
| `scripts/provider-bootstrap.ts` | Add `open-router` to `parseProviderArg` |
| `docs/advanced-setup.md` | Promote OpenRouter to standalone section |
| `package.json` | Add `dev:open-router` script |

---

## Task 1: Types, constant, and builder in `providerProfile.ts`

**Files:**
- Modify: `src/utils/providerProfile.ts`

- [ ] **Step 1: Add `'open-router'` to the `ProviderProfile` union and add the URL constant**

  In `src/utils/providerProfile.ts`, find the `ProviderProfile` type and the existing `DEFAULT_GEMINI_BASE_URL` constant, and update them:

  ```typescript
  // Change this:
  export type ProviderProfile = 'openai' | 'ollama' | 'codex' | 'gemini' | 'atomic-chat'

  // To this:
  export type ProviderProfile = 'openai' | 'ollama' | 'codex' | 'gemini' | 'atomic-chat' | 'open-router'
  ```

  After the `DEFAULT_GEMINI_BASE_URL` constant, add:

  ```typescript
  export const DEFAULT_OPENROUTER_BASE_URL = 'https://openrouter.ai/api/v1'
  ```

- [ ] **Step 2: Update `isProviderProfile()` to accept `'open-router'`**

  ```typescript
  // Change this:
  export function isProviderProfile(value: unknown): value is ProviderProfile {
    return (
      value === 'openai' ||
      value === 'ollama' ||
      value === 'codex' ||
      value === 'gemini' ||
      value === 'atomic-chat'
    )
  }

  // To this:
  export function isProviderProfile(value: unknown): value is ProviderProfile {
    return (
      value === 'openai' ||
      value === 'ollama' ||
      value === 'codex' ||
      value === 'gemini' ||
      value === 'atomic-chat' ||
      value === 'open-router'
    )
  }
  ```

- [ ] **Step 3: Add `buildOpenRouterProfileEnv()` after `buildOpenAIProfileEnv()`**

  ```typescript
  export function buildOpenRouterProfileEnv(options: {
    model?: string | null
    apiKey?: string | null
    processEnv?: NodeJS.ProcessEnv
  }): ProfileEnv | null {
    const processEnv = options.processEnv ?? process.env
    const key = sanitizeApiKey(options.apiKey ?? processEnv.OPENAI_API_KEY)
    if (!key) {
      return null
    }

    return {
      OPENAI_BASE_URL: DEFAULT_OPENROUTER_BASE_URL,
      OPENAI_MODEL: sanitizeProviderConfigValue(options.model, { OPENAI_API_KEY: key }, processEnv) || 'google/gemini-2.0-flash-001',
      OPENAI_API_KEY: key,
    }
  }
  ```

- [ ] **Step 4: Add `open-router` branch in `buildLaunchEnv()`**

  Inside `buildLaunchEnv()`, right before the `if (options.profile === 'codex')` block, add:

  ```typescript
  if (options.profile === 'open-router') {
    env.OPENAI_BASE_URL = persistedOpenAIBaseUrl || DEFAULT_OPENROUTER_BASE_URL
    env.OPENAI_MODEL = persistedOpenAIModel || 'google/gemini-2.0-flash-001'
    env.OPENAI_API_KEY = processEnv.OPENAI_API_KEY || persistedEnv.OPENAI_API_KEY
    delete env.CODEX_API_KEY
    delete env.CHATGPT_ACCOUNT_ID
    delete env.CODEX_ACCOUNT_ID
    return env
  }
  ```

- [ ] **Step 5: Run typecheck**

  ```bash
  bun run typecheck
  ```

  Expected: no errors related to `ProviderProfile` or `open-router`.

---

## Task 2: Tests for `providerProfile.ts`

**Files:**
- Modify: `src/utils/providerProfile.test.ts`

- [ ] **Step 1: Add import for new exports**

  At the top of `src/utils/providerProfile.test.ts`, add `buildOpenRouterProfileEnv` and `DEFAULT_OPENROUTER_BASE_URL` to the import:

  ```typescript
  import {
    buildStartupEnvFromProfile,
    buildAtomicChatProfileEnv,
    buildCodexProfileEnv,
    buildGeminiProfileEnv,
    buildLaunchEnv,
    buildOllamaProfileEnv,
    buildOpenAIProfileEnv,
    buildOpenRouterProfileEnv,
    createProfileFile,
    DEFAULT_OPENROUTER_BASE_URL,
    maskSecretForDisplay,
    loadProfileFile,
    PROFILE_FILE_NAME,
    redactSecretValueForDisplay,
    saveProfileFile,
    sanitizeProviderConfigValue,
    selectAutoProfile,
    type ProfileFile,
  } from './providerProfile.ts'
  ```

- [ ] **Step 2: Write the failing tests**

  Append to `src/utils/providerProfile.test.ts`:

  ```typescript
  test('buildOpenRouterProfileEnv returns null when no api key', () => {
    const env = buildOpenRouterProfileEnv({ processEnv: {} })
    assert.equal(env, null)
  })

  test('buildOpenRouterProfileEnv sets hardcoded base url and default model', () => {
    const env = buildOpenRouterProfileEnv({
      apiKey: 'sk-or-test',
      processEnv: {},
    })
    assert.ok(env)
    assert.equal(env.OPENAI_BASE_URL, DEFAULT_OPENROUTER_BASE_URL)
    assert.equal(env.OPENAI_MODEL, 'google/gemini-2.0-flash-001')
    assert.equal(env.OPENAI_API_KEY, 'sk-or-test')
  })

  test('buildOpenRouterProfileEnv uses provided model', () => {
    const env = buildOpenRouterProfileEnv({
      apiKey: 'sk-or-test',
      model: 'anthropic/claude-sonnet-4-5',
      processEnv: {},
    })
    assert.ok(env)
    assert.equal(env.OPENAI_MODEL, 'anthropic/claude-sonnet-4-5')
  })

  test('open-router launch sets CLAUDE_CODE_USE_OPENAI and OpenRouter base url', async () => {
    const env = await buildLaunchEnv({
      profile: 'open-router',
      persisted: profile('open-router', {
        OPENAI_BASE_URL: DEFAULT_OPENROUTER_BASE_URL,
        OPENAI_MODEL: 'anthropic/claude-sonnet-4-5',
        OPENAI_API_KEY: 'sk-or-persisted',
      }),
      goal: 'balanced',
      processEnv: {},
    })

    assert.equal(env.CLAUDE_CODE_USE_OPENAI, '1')
    assert.equal(env.OPENAI_BASE_URL, DEFAULT_OPENROUTER_BASE_URL)
    assert.equal(env.OPENAI_MODEL, 'anthropic/claude-sonnet-4-5')
    assert.equal(env.OPENAI_API_KEY, 'sk-or-persisted')
    assert.equal(env.CODEX_API_KEY, undefined)
    assert.equal(env.CHATGPT_ACCOUNT_ID, undefined)
    assert.equal(env.CLAUDE_CODE_USE_GEMINI, undefined)
  })

  test('open-router launch falls back to default model when no persisted model', async () => {
    const env = await buildLaunchEnv({
      profile: 'open-router',
      persisted: null,
      goal: 'balanced',
      processEnv: {
        OPENAI_API_KEY: 'sk-or-live',
      },
    })

    assert.equal(env.CLAUDE_CODE_USE_OPENAI, '1')
    assert.equal(env.OPENAI_BASE_URL, DEFAULT_OPENROUTER_BASE_URL)
    assert.equal(env.OPENAI_MODEL, 'google/gemini-2.0-flash-001')
    assert.equal(env.OPENAI_API_KEY, 'sk-or-live')
  })

  test('open-router launch ignores mismatched persisted gemini env', async () => {
    const env = await buildLaunchEnv({
      profile: 'open-router',
      persisted: profile('gemini', {
        GEMINI_MODEL: 'gemini-2.0-flash',
        GEMINI_API_KEY: 'gem-persisted',
      }),
      goal: 'balanced',
      processEnv: {
        OPENAI_API_KEY: 'sk-or-live',
      },
    })

    assert.equal(env.CLAUDE_CODE_USE_OPENAI, '1')
    assert.equal(env.OPENAI_BASE_URL, DEFAULT_OPENROUTER_BASE_URL)
    assert.equal(env.OPENAI_MODEL, 'google/gemini-2.0-flash-001')
    assert.equal(env.OPENAI_API_KEY, 'sk-or-live')
    assert.equal(env.GEMINI_API_KEY, undefined)
    assert.equal(env.GEMINI_MODEL, undefined)
  })
  ```

- [ ] **Step 3: Run the tests to verify they fail (new exports not yet implemented would fail, but since Task 1 is done first, they should pass)**

  ```bash
  bun test src/utils/providerProfile.test.ts
  ```

  Expected: all tests pass including the new ones.

- [ ] **Step 4: Commit**

  ```bash
  git add src/utils/providerProfile.ts src/utils/providerProfile.test.ts
  git commit -m "feat: add open-router provider profile type and builder"
  ```

---

## Task 3: Wizard UI in `provider.tsx`

**Files:**
- Modify: `src/commands/provider/provider.tsx`

- [ ] **Step 1: Add import for new exports**

  In `src/commands/provider/provider.tsx`, add `buildOpenRouterProfileEnv` and `DEFAULT_OPENROUTER_BASE_URL` to the import from `providerProfile.js`:

  ```typescript
  import {
    buildCodexProfileEnv,
    buildGeminiProfileEnv,
    buildOllamaProfileEnv,
    buildOpenAIProfileEnv,
    buildOpenRouterProfileEnv,
    createProfileFile,
    DEFAULT_GEMINI_BASE_URL,
    DEFAULT_GEMINI_MODEL,
    DEFAULT_OPENROUTER_BASE_URL,
    deleteProfileFile,
    loadProfileFile,
    maskSecretForDisplay,
    redactSecretValueForDisplay,
    sanitizeApiKey,
    sanitizeProviderConfigValue,
    saveProfileFile,
    type ProfileEnv,
    type ProfileFile,
    type ProviderProfile,
  } from '../../utils/providerProfile.js'
  ```

- [ ] **Step 2: Add new Step variants to the `Step` union type**

  Find the `Step` union type and add two new variants:

  ```typescript
  type Step =
    | { name: 'choose' }
    | { name: 'auto-goal' }
    | { name: 'auto-detect'; goal: RecommendationGoal }
    | { name: 'ollama-detect' }
    | { name: 'openai-key'; defaultModel: string }
    | { name: 'openai-base'; apiKey: string; defaultModel: string }
    | {
        name: 'openai-model'
        apiKey: string
        baseUrl: string | null
        defaultModel: string
      }
    | { name: 'gemini-key' }
    | { name: 'gemini-model'; apiKey: string }
    | { name: 'codex-check' }
    | { name: 'openrouter-key' }
    | { name: 'openrouter-model'; apiKey: string }
  ```

- [ ] **Step 3: Add `'open-router'` option to `ProviderChooser`**

  In the `ProviderChooser` function, find the `options` array and add the OpenRouter entry after Gemini:

  ```typescript
  const options: OptionWithDescription<ProviderChoice>[] = [
    {
      label: 'Auto',
      value: 'auto',
      description:
        'Prefer local Ollama when available, otherwise guide you into OpenAI-compatible setup',
    },
    {
      label: 'Ollama',
      value: 'ollama',
      description: 'Use a local Ollama model with no API key',
    },
    {
      label: 'OpenAI-compatible',
      value: 'openai',
      description:
        'GPT-4o, DeepSeek, OpenRouter, Groq, LM Studio, and similar APIs',
    },
    {
      label: 'Gemini',
      value: 'gemini',
      description: 'Use a Google Gemini API key',
    },
    {
      label: 'OpenRouter',
      value: 'open-router',
      description: 'Access Claude, Gemini, DeepSeek, and others via openrouter.ai',
    },
    {
      label: 'Codex',
      value: 'codex',
      description: 'Use existing ChatGPT Codex CLI auth or env credentials',
    },
  ]
  ```

- [ ] **Step 4: Add `open-router` case to `buildSavedProfileSummary()`**

  Inside the `switch (profile)` statement in `buildSavedProfileSummary()`, add before `default`:

  ```typescript
  case 'open-router':
    return {
      providerLabel: 'OpenRouter',
      modelLabel: getSafeDisplayValue(
        env.OPENAI_MODEL ?? 'google/gemini-2.0-flash-001',
        process.env,
        env,
      ),
      endpointLabel: getSafeDisplayValue(
        env.OPENAI_BASE_URL ?? DEFAULT_OPENROUTER_BASE_URL,
        process.env,
        env,
      ),
      credentialLabel:
        maskSecretForDisplay(env.OPENAI_API_KEY) !== undefined
          ? 'configured'
          : undefined,
    }
  ```

- [ ] **Step 5: Update `buildCurrentProviderSummary()` to detect OpenRouter URL**

  In `buildCurrentProviderSummary()`, inside the `if (isEnvTruthy(processEnv.CLAUDE_CODE_USE_OPENAI))` block, update the `providerLabel` detection:

  ```typescript
  let providerLabel = 'OpenAI-compatible'
  if (request.transport === 'codex_responses') {
    providerLabel = 'Codex'
  } else if (request.baseUrl.includes('localhost:11434')) {
    providerLabel = 'Ollama'
  } else if (request.baseUrl.includes('localhost:1234')) {
    providerLabel = 'LM Studio'
  } else if (request.baseUrl.includes('openrouter.ai')) {
    providerLabel = 'OpenRouter'
  }
  ```

- [ ] **Step 6: Add wizard routing for `open-router` choice and new steps**

  In `ProviderWizard`, inside the `switch (step.name)` block:

  First, add `'open-router'` to the `case 'choose'` handler inside `onChoose`:

  ```typescript
  } else if (value === 'open-router') {
    setStep({ name: 'openrouter-key' })
  } else if (value === 'clear') {
  ```

  Then add two new `case` blocks after `case 'codex-check'`:

  ```typescript
  case 'openrouter-key':
    return (
      <TextEntryDialog
        resetStateKey={step.name}
        title="OpenRouter setup"
        subtitle="Step 1 of 2"
        description={
          process.env.OPENAI_API_KEY
            ? 'Enter an OpenRouter API key (sk-or-...), or leave blank to reuse the current OPENAI_API_KEY from this session.'
            : 'Enter your OpenRouter API key. Get one at https://openrouter.ai/keys.'
        }
        initialValue=""
        placeholder="sk-or-..."
        mask="*"
        allowEmpty={Boolean(process.env.OPENAI_API_KEY)}
        validate={value => {
          const candidate = value.trim() || process.env.OPENAI_API_KEY || ''
          return sanitizeApiKey(candidate)
            ? null
            : 'Enter a real API key. Placeholder values are not valid.'
        }}
        onSubmit={value => {
          const apiKey = value.trim() || process.env.OPENAI_API_KEY || ''
          setStep({ name: 'openrouter-model', apiKey })
        }}
        onCancel={() => setStep({ name: 'choose' })}
      />
    )

  case 'openrouter-model':
    return (
      <TextEntryDialog
        resetStateKey={step.name}
        title="OpenRouter setup"
        subtitle="Step 2 of 2"
        description="Enter a model name. Leave blank for google/gemini-2.0-flash-001. Browse models at https://openrouter.ai/models."
        initialValue=""
        placeholder="google/gemini-2.0-flash-001"
        allowEmpty
        onSubmit={value => {
          const env = buildOpenRouterProfileEnv({
            apiKey: step.apiKey,
            model: value.trim() || 'google/gemini-2.0-flash-001',
            processEnv: {},
          })
          if (env) {
            finishProfileSave(onDone, 'open-router', env)
          }
        }}
        onCancel={() => setStep({ name: 'openrouter-key' })}
      />
    )
  ```

- [ ] **Step 7: Run typecheck**

  ```bash
  bun run typecheck
  ```

  Expected: no errors.

- [ ] **Step 8: Run wizard tests**

  ```bash
  bun test src/commands/provider/provider.test.tsx
  ```

  Expected: all existing tests pass.

- [ ] **Step 9: Commit**

  ```bash
  git add src/commands/provider/provider.tsx
  git commit -m "feat: add OpenRouter wizard steps to /provider command"
  ```

---

## Task 4: Dev scripts — `provider-launch.ts` and `provider-bootstrap.ts`

**Files:**
- Modify: `scripts/provider-launch.ts`
- Modify: `scripts/provider-bootstrap.ts`

- [ ] **Step 1: Add `'open-router'` to `parseLaunchOptions()` in `provider-launch.ts`**

  Find the condition that checks valid profile strings and add `'open-router'`:

  ```typescript
  if ((lower === 'auto' || lower === 'openai' || lower === 'ollama' || lower === 'codex' || lower === 'gemini' || lower === 'atomic-chat' || lower === 'open-router') && requestedProfile === 'auto') {
    requestedProfile = lower as ProviderProfile | 'auto'
    continue
  }
  ```

- [ ] **Step 2: Add `open-router` usage message and validation in `main()` of `provider-launch.ts`**

  Update the usage error message:

  ```typescript
  console.error('Usage: bun run scripts/provider-launch.ts [openai|ollama|codex|gemini|atomic-chat|open-router|auto] [--fast] [--goal <latency|balanced|coding>] [-- <cli args>]')
  ```

  Add API key validation after the `openai` check:

  ```typescript
  if (profile === 'open-router' && (!env.OPENAI_API_KEY || env.OPENAI_API_KEY === 'SUA_CHAVE')) {
    console.error('OPENAI_API_KEY is required for open-router profile. Run: bun run profile:init -- --provider open-router --api-key <key>')
    process.exit(1)
  }
  ```

- [ ] **Step 3: Add `open-router` to `parseProviderArg()` in `provider-bootstrap.ts`**

  ```typescript
  function parseProviderArg(): ProviderProfile | 'auto' {
    const p = parseArg('--provider')?.toLowerCase()
    if (p === 'openai' || p === 'ollama' || p === 'codex' || p === 'gemini' || p === 'atomic-chat' || p === 'open-router') return p
    return 'auto'
  }
  ```

- [ ] **Step 4: Add `open-router` branch to `main()` in `provider-bootstrap.ts`**

  Add `buildOpenRouterProfileEnv` to the import:

  ```typescript
  import {
    buildAtomicChatProfileEnv,
    buildCodexProfileEnv,
    buildGeminiProfileEnv,
    buildOllamaProfileEnv,
    buildOpenAIProfileEnv,
    buildOpenRouterProfileEnv,
    createProfileFile,
    saveProfileFile,
    selectAutoProfile,
    type ProfileFile,
    type ProviderProfile,
  } from '../src/utils/providerProfile.ts'
  ```

  Add `open-router` branch before the final `else` (openai):

  ```typescript
  } else if (selected === 'open-router') {
    const builtEnv = buildOpenRouterProfileEnv({
      model: argModel || null,
      apiKey: argApiKey || process.env.OPENAI_API_KEY || null,
      processEnv: process.env,
    })

    if (!builtEnv) {
      console.error('OpenRouter profile requires a real API key. Use --api-key or set OPENAI_API_KEY.')
      process.exit(1)
    }

    env = builtEnv
  } else {
  ```

- [ ] **Step 5: Add `dev:open-router` script to `package.json`**

  In `package.json`, after `"dev:atomic-chat"`, add:

  ```json
  "dev:open-router": "bun run scripts/provider-launch.ts open-router",
  ```

- [ ] **Step 6: Run typecheck**

  ```bash
  bun run typecheck
  ```

  Expected: no errors.

- [ ] **Step 7: Commit**

  ```bash
  git add scripts/provider-launch.ts scripts/provider-bootstrap.ts package.json
  git commit -m "feat: add open-router support to provider dev scripts"
  ```

---

## Task 5: Update docs

**Files:**
- Modify: `docs/advanced-setup.md`

- [ ] **Step 1: Promote OpenRouter to its own section**

  Replace the existing "### Google Gemini via OpenRouter" section with:

  ```markdown
  ### OpenRouter

  [OpenRouter](https://openrouter.ai) provides access to Claude, Gemini, DeepSeek, Llama, and many other models via a single OpenAI-compatible API.

  ```bash
  export CLAUDE_CODE_USE_OPENAI=1
  export OPENAI_API_KEY=sk-or-...
  export OPENAI_BASE_URL=https://openrouter.ai/api/v1
  export OPENAI_MODEL=google/gemini-2.0-flash-001
  ```

  You can use any model from [openrouter.ai/models](https://openrouter.ai/models). Model availability changes over time — if a model stops working, check the models page for current options.
  ```

- [ ] **Step 2: Commit**

  ```bash
  git add docs/advanced-setup.md
  git commit -m "docs: expand OpenRouter section in advanced-setup guide"
  ```

---

## Task 6: Final verification

- [ ] **Step 1: Run all provider-related tests**

  ```bash
  bun test src/utils/providerRecommendation.test.ts src/utils/providerProfile.test.ts
  ```

  Expected: all tests pass.

- [ ] **Step 2: Run wizard tests**

  ```bash
  bun test src/commands/provider/provider.test.tsx
  ```

  Expected: all tests pass.

- [ ] **Step 3: Run typecheck**

  ```bash
  bun run typecheck
  ```

  Expected: no errors.

- [ ] **Step 4: Smoke test**

  ```bash
  bun run smoke
  ```

  Expected: exits with 0.
