---
name: freellmapi
description: Set up, connect and use a self-hosted FreeLLMAPI router, a single OpenAI- and Anthropic-compatible /v1 endpoint that routes requests across many free LLM provider tiers with automatic failover. Use when the user mentions FreeLLMAPI, wants to point Claude Code, Codex, an OpenAI SDK or another client at it, pick a routing strategy (auto, auto:fast, auto:smart, a profile), list usable models, check usage or free-tier budget, or debug which provider served a request.
---

# FreeLLMAPI

FreeLLMAPI runs on the user's own machine and exposes one endpoint, by default `http://localhost:3001/v1`. Provider keys are added in its dashboard and stored encrypted. Clients only ever use one **unified key** (`freellmapi-…`) from the dashboard's Keys page.

## Keys and safety

- Read the unified key from an environment variable such as `FREELLMAPI_KEY`. Never print it, commit it, or put it in a URL or a config file the user didn't ask for.
- Provider keys belong in the dashboard, not in prompts or files.
- Free tiers have usage rules. FreeLLMAPI describes itself as for personal experimentation, so don't suggest using it to break a provider's terms.

## 1. Install or start it

Check whether it's already running: open `http://localhost:3001`, or call `GET /v1/models` with the key.

If it isn't installed, the project offers:
- **Desktop app:** the Windows `.exe` or macOS `.dmg` from the project's Releases page. The tray popover shows the unified key.
- **Docker one-liner:** `curl -fsSL https://freellmapi.co/install.sh | bash`. Suggest the user read the script first; don't pipe it to a shell yourself without their yes.
- **From source:** see `docs/en/install/01-install.md` in the repository.

Then the user adds provider keys on the **Keys** page and orders the **Fallback Chain**.

## 2. Connect a client

**OpenAI-compatible clients:** base URL `http://localhost:3001/v1`, API key = the unified key.

```python
from openai import OpenAI
import os
client = OpenAI(base_url="http://localhost:3001/v1", api_key=os.environ["FREELLMAPI_KEY"])
r = client.chat.completions.create(model="auto", messages=[{"role": "user", "content": "Hello"}])
print(r.choices[0].message.content)
```

**Claude Code and Anthropic SDKs:** use the server's origin, without `/v1`. Both `x-api-key` and `Authorization: Bearer` work.

```powershell
$env:ANTHROPIC_BASE_URL = "http://localhost:3001"
```

The project's helper sets up a client and backs up the old config first. `--dry-run` shows what it would change:

```bash
npx freellmapi setup-claude --url http://localhost:3001 --api-key <unified-key> --dry-run
```

`npx freellmapi launch` starts Claude Code against the router without writing the key to a config file. Prefer it when the user wants the key kept out of files.

## 3. Choose a model

| `model` value | Behaviour |
|---|---|
| `auto` | Uses the dashboard's active fallback chain |
| `auto:fast` | Favours measured speed |
| `auto:smart` | Favours the most capable models |
| `auto:reliable` | Favours recent success rate |
| `auto:balanced` | Default blend |
| `auto:<profile>` | Uses a named profile's chain, for example `auto:coding` |
| `fusion` | Asks several models in parallel; a judge merges one answer |
| a model id | That model only |

List the models that can actually be called right now:

```bash
curl "http://localhost:3001/v1/models?execution_status=ready" -H "Authorization: Bearer $FREELLMAPI_KEY"
```

## 4. Other endpoints

Chat, `/v1/responses`, `/v1/completions`, `/v1/embeddings`, `/v1/images/generations`, `/v1/videos/generations`, `/v1/audio/speech`, `/v1/audio/transcriptions`, `/v1/messages` (Anthropic format) and `/v1beta` (Gemini format). Streaming and tool calling work across providers.

## 5. Debug and monitor

- Every response has an `X-Routed-Via: <platform>/<model>` header showing which provider served it. Report it when the user asks why a reply looks different.
- `GET /v1/usage` (with the unified key) shows usage.
- Rate-limited providers are skipped automatically. If every request fails, check the Keys page for keys that need re-testing (`npx freellmapi keys test <platform>`), and the fallback chain for disabled models.
- The interactive API reference is at `GET /v1/docs`, and the spec at `GET /v1/openapi.json`. Check them before guessing parameters.

Never claim a request succeeded without seeing the response, and never invent a model id: list the models first.
