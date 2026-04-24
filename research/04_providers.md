# Research: LLM Provider Integration

**Statut: terminé**
**Date: 2026-04-24**

## Plan
1. Partie A - OpenRouter (API, quirks, tool calling, streaming, SDK)
2. Partie B - Codex via subscription ChatGPT (OAuth flow, endpoints, ToS, réutilisabilité)
3. Partie C - Abstractions unifiées (litellm, aisuite, instructor, aider, goose, openhands)
4. Partie D - Features cross-providers (tool calling, streaming, caching, thinking, tokens, retry, cost, images)
5. Livrable - Tableau comparatif, recommandation, design interface Python, snippets

---

## Partie B - Codex via abonnement ChatGPT (findings preliminaries)

### Flow OAuth (extrait du repo `openai/codex`, Rust login crate)

Codex CLI implemente un flow **OAuth 2.0 PKCE + authorization_code** classique contre `https://auth.openai.com`, puis fait un token exchange pour récupérer une API key style `sk-…` :

1. **Serveur local** sur `http://localhost:1455/auth/callback` (`tiny_http`).
2. **Redirect vers** `https://auth.openai.com/oauth/authorize?…` avec :
   - `response_type=code`
   - `client_id=app_EMoamEEZ73f0CkXaXp7hrann` (constant public dans le source)
   - `redirect_uri=http://localhost:1455/auth/callback`
   - `scope=openid profile email offline_access api.connectors.read api.connectors.invoke`
   - `code_challenge` (S256, PKCE)
   - `id_token_add_organizations=true`
   - `codex_cli_simplified_flow=true`
3. **Callback** reçoit `code`, le serveur POST `https://auth.openai.com/oauth/token` avec `grant_type=authorization_code` -> reçoit `{id_token, access_token, refresh_token}`.
4. **Token exchange** supplémentaire : même endpoint, `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`, `subject_token=<id_token>`, `requested_token=openai-api-key`, `subject_token_type=urn:ietf:params:oauth:token-type:id_token` -> retourne un `access_token` **qui est une API key au format OpenAI**.
5. **Persistance** dans `$CODEX_HOME/auth.json` (par défaut `~/.codex/auth.json`) avec la structure :
   ```json
   {
     "OPENAI_API_KEY": "sk-...",  // token-exchanged API key
     "tokens": {
       "id_token": {...claims...},
       "access_token": "<jwt>",
       "refresh_token": "<opaque>",
       "account_id": "<chatgpt_account_id>"
     },
     "last_refresh": "2026-04-24T...Z"
   }
   ```

### Endpoint LLM utilisé en mode ChatGPT

- Default base URL : **`https://chatgpt.com/backend-api/codex`** (et non `api.openai.com`)
- Wire API : **Responses API** (`POST /responses`, endpoint compact `/responses/compact`) - le même format JSON que `api.openai.com/v1/responses`, mais routé via le backend ChatGPT.
- Headers requis :
  - `Authorization: Bearer <access_token>` (le JWT, pas l'API key exchangée)
  - `ChatGPT-Account-ID: <chatgpt_account_id>` (obligatoire, sinon 401)
  - Optionnel : `X-OpenAI-Fedramp: true` (workspaces fedramp only)
  - Identity headers Codex : `X-Codex-Installation-Id`, `X-Codex-Window-Id`, `x-responsesapi-include-timing-metrics`
  - `originator: codex_cli_rs` (Codex identifie son client ; si manquant, risque de blocage côté backend)

### Refresh
- `POST https://auth.openai.com/oauth/token` avec `grant_type=refresh_token`, `client_id=app_EMoamEEZ73f0CkXaXp7hrann`, `refresh_token=...`
- Response: `{id_token, access_token, refresh_token}` (rotation)
- Codex refresh tous les 8 jours (`TOKEN_REFRESH_INTERVAL: i64 = 8`) ou sur 401.

### SDK Python officiel (sdk/python)
OpenAI fournit un SDK Python expérimental `codex-app-server-sdk` qui **spawn le binaire `codex` en subprocess** et communique via **JSON-RPC 2.0 sur stdio** :
```python
from codex_app_server import Codex
with Codex() as codex:
    thread = codex.thread_start(model="gpt-5")
    result = thread.run("Say hello")
    print(result.final_response)
```
- Command : `codex app-server --listen stdio://`
- Notifications asynchrones : `agent_message_delta`, `turn_completed`, etc.
- **Pas d'exposition directe d'un client HTTP** - le binaire gère auth, tokens, tool calling.
- Dépendance runtime : `openai-codex-cli-bin` (wheel avec binaire par plateforme).

### ToS / légalité
- OpenAI Usage Policies (2025) : usage programmatique via Codex CLI est **explicitement autorisé** si on utilise le CLI officiel. Les projets tiers (reimplementation pure Python de l'OAuth) sont dans une zone grise similaire aux projets historiques `revChatGPT` :
  - **OK** : spawner le `codex` binaire officiel comme subprocess (c'est exactement ce que fait le SDK Python OpenAI).
  - **Risqué** : reimplementer le flow OAuth avec le `client_id` codé en dur hors du CLI officiel. Même si techniquement c'est du code public MIT, l'utilisation du endpoint `chatgpt.com/backend-api/codex` n'est pas documentée pour les tiers. OpenAI peut révoquer le `client_id` ou ajouter un `X-Client-Verification` à tout moment.
  - **Interdit** : abuser le compte (partage, scraping, fine-tuning via exfiltration).
- Historiquement : `revChatGPT`, `chatgpt-api`, `PyChatGPT` ont tous cassé quand OpenAI a ajouté Cloudflare + Arkose + signatures. Leur durée de vie moyenne était 2-6 mois.

### Conclusion partie B (provisoire)
**Recommandation** : pour un agent Python, **subprocess le `codex` CLI officiel** via le SDK `codex-app-server-sdk` (ou reimplementer le JSON-RPC stdio) est **la seule approche pérenne**. Reimplementer l'OAuth flow + hitting `chatgpt.com/backend-api/codex` directement fonctionnera peut-être 6 mois puis cassera. Si on veut du contrôle fin (e.g. notre propre boucle d'agent, pas celle de codex), on perd alors l'avantage du subscription.

---

## Partie A - OpenRouter

### Endpoint & auth
- **Base URL** : `https://openrouter.ai/api/v1`
- **Endpoint chat** : `POST /chat/completions` (OpenAI-compatible payload)
- **Endpoint Responses API (beta)** : `POST /responses` (surface OpenAI-compatible, forward vers providers qui supportent la Responses API)
- **Endpoints utilitaires** : `GET /models`, `GET /models/{author}/{slug}/endpoints`, `GET /generation?id=<gen_id>` (récupération coût/tokens a posteriori), `GET /credits`
- **Auth** : `Authorization: Bearer <OPENROUTER_API_KEY>`
- **Headers OR-spécifiques (optionnels)** :
  - `HTTP-Referer: <your-site-url>` - attribution leaderboard
  - `X-Title: <your-app-name>` (ancien `X-OpenRouter-Title`) - attribution leaderboard
  - `OR-Organization: <org_id>` - multi-org billing

### OpenAI SDK compat
Le wire-format est **strictement compatible OpenAI chat/completions**. On peut réutiliser `openai-python` tel quel :
```python
from openai import OpenAI
client = OpenAI(base_url="https://openrouter.ai/api/v1", api_key=os.environ["OPENROUTER_API_KEY"])
resp = client.chat.completions.create(model="anthropic/claude-sonnet-4.6", messages=[...])
```
Il **n'y a pas de SDK Python officiel OpenRouter** - ils documentent explicitement l'usage du SDK OpenAI avec `base_url` override (JS et Python). La communauté maintient des wrappers minces (e.g. `openrouter-py`) mais rien de canonique.

### Tool calling
- **Schema : OpenAI format** (`{"type": "function", "function": {"name", "description", "parameters"}}`). OpenRouter normalise et traduit vers chaque provider downstream (Anthropic `tools`/`tool_use`/`tool_result`, Gemini `functionDeclarations`) côté serveur. **On envoie du OpenAI, on reçoit du OpenAI**.
- `tool_choice` : `"auto"` | `"none"` | `"required"` | `{"type": "function", "function": {"name": "foo"}}`.
- `parallel_tool_calls` : supporté (true par défaut quand le model le permet).
- **Filtrer les modèles tool-capable** : `GET /models?supported_parameters=tools`. Tous les modèles ne supportent pas (e.g. certains base models).
- **Quirk** : re-envoyer le `tools` array à *chaque* tour de la boucle tool_use, pas seulement au premier.
- **Quirk Anthropic via OR** : le format retour reste OpenAI (`tool_calls` array) mais les IDs générés (`call_xxx`) doivent être renvoyés tels quels dans le tour suivant. OR gère la translation des IDs côté Anthropic.

### Prompt caching (crucial pour un agent)
OpenRouter fait du **passthrough** : chaque provider garde son modèle économique, OR forward et expose les metrics.

| Provider | Type | Discount read | Min tokens | TTL | Notes |
|---|---|---|---|---|---|
| OpenAI | implicite | 0.25-0.5x | 1024 | ~5 min | zéro write cost |
| Anthropic | explicite (`cache_control`) | 0.1x (read) | 1024 (Sonnet) / 4096 (Opus, Haiku 4.5) | 5min ou 1h | write = 1.25x (5m) ou 2x (1h) |
| Gemini 2.5 | implicite | ~0.25x | 4096 | 3-5 min | last breakpoint only |
| DeepSeek | implicite | ~0.1x | N/A | auto | writes facturés normalement |
| Grok / Moonshot / Kimi (Groq) | implicite | varie | N/A | auto | |

**Cache_control Anthropic via OR** : même syntaxe que l'API native Anthropic :
```json
{"type": "text", "text": "<system prompt long>", "cache_control": {"type": "ephemeral"}}
```
ou top-level `{"cache_control": {"type": "ephemeral", "ttl": "1h"}}` (force routing vers Anthropic direct, exclut Bedrock/Vertex).

**Métriques de retour** dans `usage.prompt_tokens_details` :
```json
{"cached_tokens": 10318, "cache_write_tokens": 0}
```

**Sticky routing** : OR verrouille automatiquement le provider endpoint après un cache write pour maximiser les hits du cache (important : ne pas changer de provider entre tours d'agent).

### Routing / fallbacks (autres champs OR-spécifiques)
Champs non-OpenAI utilisables dans le body :
- `provider: {order: ["Anthropic", "Bedrock"], allow_fallbacks: true, require_parameters: true, data_collection: "deny"}` - préférence + filtres
- `models: ["anthropic/claude-sonnet-4.6", "openai/gpt-5"]` - fallback list (si le premier refuse, cascade)
- `transforms: ["middle-out"]` - compression auto si context overflow
- `route: "fallback"` - même chose, API historique

### Modèles coding-friendly (avril 2026, prix indicatifs /M tokens input/output)
- `anthropic/claude-sonnet-4.6` : ~$3 / $15 - reference pour agentic coding, tool-use excellent
- `anthropic/claude-opus-4.7` : ~$15 / $75 - raisonnement long
- `openai/gpt-5` : ~$5 / $15 - bon tool calling, context 400k
- `openai/gpt-5-codex` : ~$3 / $12 - fine-tuné code
- `google/gemini-2.5-pro` : ~$2.5 / $10 - 1M context, cache implicite agressif
- `qwen/qwen3-coder-480b` : ~$0.3 / $1.2 - open weights, excellent rapport qualité/prix
- `deepseek/deepseek-v3.2` : ~$0.3 / $1.2 - caching auto 90% discount, strong coder
- `moonshotai/kimi-k2` : ~$0.6 / $2.5 - 200k context, bon sur debugging
- `x-ai/grok-4` : ~$5 / $15 - tool-use fiable

---

## Partie C - Abstractions unifiées existantes

### LiteLLM (BerriAI/litellm)
**Deux modes** : (1) **SDK Python** `litellm.completion(...)` en lib, (2) **Proxy gateway** FastAPI (virtual keys, spend tracking, admin UI).
- **Providers** : 100+ (OpenAI, Anthropic, Bedrock, Vertex, Gemini, Cohere, Mistral, Groq, Together, OpenRouter, Ollama, vLLM, HF, etc.).
- **Normalisation** : tout est exposé au **format OpenAI chat/completions** (`messages`, `tools`, `tool_calls`, `usage`). Request et response sont traduits.
- **Features couvertes** : streaming (SSE unifié), tool calling (schema OpenAI, traduit vers Anthropic/Gemini), caching (wrapper Redis + support caching natif Anthropic via `cache_control` passthrough), fallbacks, retries, cost tracking (`response._hidden_params["response_cost"]`), token counting, structured outputs via `response_format`.
- **Points faibles** :
  - Surface API énorme, beaucoup d'edge cases, beaucoup de bugs historiques sur tool calling Anthropic (parsing tool_result).
  - Réécrit son propre modèle de messages → parfois perd des infos provider-spécifiques (thinking blocks Claude, reasoning summaries OpenAI).
  - Dépendances lourdes (FastAPI, sqlalchemy, redis, posthog, sentry…) même en mode SDK.
  - Logique complexe pour le cost tracking → prix parfois obsolètes dans `litellm/model_prices_and_context_window.json`.

### aisuite (andrewyng/aisuite)
- **Philosophie** : minimalisme. Un wrapper ~1000 LoC total autour des SDK officiels.
- Interface unique : `client.chat.completions.create(model="anthropic:claude-...", ...)`.
- **Pas** de proxy, pas de caching, pas de retry built-in, pas de streaming unifié robuste.
- **Bon** pour un prototype multi-provider quand on fait du texte simple. **Insuffisant pour un agent coder** (tool calling hétérogène, pas de cache).

### instructor
- Orthogonal : **structured output** (Pydantic + retry validation) par dessus *n'importe quel* client (OpenAI, Anthropic, litellm, Gemini…).
- Utile pour parser les outputs modèle, **pas** pour abstraire les providers. Peut se composer avec un provider wrapper.

### aider (llm.py / legacy.py)
`aider` (Paul Gauthier) **utilise litellm** comme backend unique depuis 2024. Son `Model` class wrappe `litellm.completion` avec :
- Détection de features (supports_tool_calling, supports_vision, supports_reasoning) par modèle.
- Gestion custom du streaming et du tool calling (reconstruit les tool_calls partiels).
- Tarif + token counting via litellm.
- Fallback manuel entre providers en cas d'erreur 529/overloaded.

Leçon aider : litellm **vaut le coup** comme base mais il faut une couche de correction par-dessus (feature flags, tool call reassembly, retry logic adaptée).

### Goose (block/goose) et OpenHands
- **Goose** : Rust, utilise `mcp-client` + providers custom (Anthropic native, OpenAI native, Databricks, OpenRouter via OpenAI-compat). Pas de litellm, choix de direct-SDK pour chaque provider majeur. Justification : contrôle fin du caching Anthropic.
- **OpenHands** (all-hands-ai) : Python, **utilise litellm** avec wrapper `llm.py` qui gère retries, metrics, caching Anthropic, thinking tokens, token counting.

### Comparaison brève litellm vs aisuite vs instructor

| | litellm | aisuite | instructor |
|---|---|---|---|
| Scope | tout multi-provider | multi-provider minimal | structured output |
| Lignes de code | ~100k | ~1k | ~10k |
| Tool calling unifié | oui (imparfait) | basique | N/A |
| Prompt caching | oui (passthrough) | non | N/A |
| Streaming | oui | partiel | oui (via backend) |
| Cost tracking | oui | non | non |
| Dépendances | lourdes | minimales | moyennes |
| Maintenance | très active | modérée | très active |

### Question clef : wrapper litellm ou écrire le notre ?

**Arguments pour wrapper litellm** :
- Gagne l'intégration de 100+ providers immédiatement.
- Normalisation OpenAI-schema déjà faite (tool_calls, streaming chunks, usage).
- Cost tracking, token counting, fallbacks gratuits.
- Precédent : aider + OpenHands le font et ça marche.

**Arguments pour écrire le notre** :
- Contrôle fin du caching Anthropic (critique pour un agent long-running) - litellm a historiquement des bugs sur le cache_control placement.
- Pas de dépendance lourde (litellm tire ~50 packages).
- On ne supporte que 3 providers (OpenRouter OpenAI-compat, Anthropic direct, Codex subscription via subprocess) - litellm est overkill.
- Thinking blocks / reasoning tokens / extended thinking : on veut passer la donnée brute, pas une normalisation qui les perd.
- Meilleur debugging (stack traces courtes, pas de magie).

**Tradeoff retenu** : **écrire notre propre abstraction**, fine et minimale, avec 3 providers directs :
1. `AnthropicProvider` (SDK `anthropic` officiel, pour caching + thinking full fidelity)
2. `OpenRouterProvider` (SDK `openai` avec `base_url` override, pour tout le reste : GPT-5, Gemini, Qwen, DeepSeek, Kimi, Grok)
3. `CodexSubscriptionProvider` (subprocess `codex app-server`, JSON-RPC stdio - cf Partie B)

Pour un 4ème provider (Bedrock, Vertex, Ollama), ajouter un `LiteLLMFallbackProvider` en backup - litellm en dépendance optionnelle (extra `[litellm]`).

---

## Partie D - Features cross-providers

Pour chaque feature : support natif par provider + stratégie d'abstraction.

### Tool calling
| Provider | Schema | Parallel | Forced tool | Notes |
|---|---|---|---|---|
| OpenAI (GPT-5, o3, etc.) | `tools: [{type:"function", function:{...}}]` | oui | `tool_choice:{type:"function", function:{name:"x"}}` | standard de fait |
| Anthropic | `tools: [{name, description, input_schema}]`, contenu `tool_use`/`tool_result` en blocks | oui | `tool_choice:{type:"tool", name:"x"}` | input/output **blocks** dans `content` array, pas `tool_calls` à côté |
| Gemini | `functionDeclarations` dans `tools` | oui | `toolConfig.functionCallingConfig.mode="ANY"` | schema OpenAPI subset |
| OpenRouter | schema OpenAI, normalisation côté serveur | oui | oui | traduit vers Anthropic/Gemini |
| DeepSeek / Qwen / Kimi | OpenAI-compat | oui | oui | robuste depuis fin 2025 |

**Stratégie** : canonical interne = **schema OpenAI**. `AnthropicProvider` fait la traduction bidirectionnelle (blocks ↔ tool_calls). Les autres passent tels quels via OpenAI-compat.

### Streaming
- **SSE** avec `data: <json>\n\n` et `data: [DONE]` pour OpenAI-compat.
- **Anthropic** : SSE avec event types nommés (`message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`) - besoin de réassembler les deltas par index.
- **Quirk tool_calls streaming** : OpenAI envoie des deltas partiels sur `function.arguments` (JSON string qui s'accumule). Anthropic envoie des deltas `input_json_delta` avec `partial_json`. Il faut accumuler string puis parser à la fin.
- **Stratégie** : type interne `StreamChunk` = union (`TextDelta`, `ToolCallDelta`, `ThinkingDelta`, `UsageDelta`, `StopReason`). Chaque provider yield cette union.

### Prompt caching
| Provider | Mécanisme | Explicit ? |
|---|---|---|
| Anthropic | `cache_control: {type: "ephemeral", ttl?: "1h"}` sur block | oui |
| OpenAI | auto si prompt ≥ 1024 tokens | non |
| Gemini 2.5 | implicite, context cache API pour explicit | mixte |
| DeepSeek | auto, très agressif | non |
| OpenRouter | passthrough de ce que le provider expose | suit le provider |

**Stratégie** : exposer `cache_breakpoints: list[int] | None` dans l'API interne (indices de messages après lesquels poser un `cache_control`). Pour Anthropic : traduit en `cache_control` blocks. Pour OpenAI/Gemini/DeepSeek : ignoré (caching est auto). `usage.cached_tokens` normalisé partout.

### Thinking / reasoning tokens
| Provider | Expose thinking ? | Contrôle budget |
|---|---|---|
| OpenAI o1/o3/o4 | non (masqués), usage.reasoning_tokens seulement | `reasoning_effort: "low"/"medium"/"high"` |
| Anthropic (extended thinking) | oui, `thinking` blocks dans content | `thinking: {type:"enabled", budget_tokens: N}` |
| DeepSeek R1 | oui, `<think>...</think>` inline ou `reasoning_content` field | non |
| Gemini 2.5 thinking | partiel (summary seulement) | `thinking_config.thinking_budget` |

**Stratégie** : champ interne `thinking: ThinkingConfig(effort: "low"|"med"|"high", budget_tokens: int | None)`. Chaque provider le traduit. Output : `CompletionResponse.thinking_text: str | None` (best-effort, None quand OpenAI masque). **Les thinking blocks Anthropic doivent être renvoyés tels quels dans le prochain tour** (sinon le modèle refuse) → on garde le message assistant complet (pas juste le texte).

### Image input
- OpenAI / OpenRouter / Gemini / Anthropic : tous supportent `image_url` (URL ou `data:image/png;base64,...`).
- Anthropic format : `{"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}` **ou** `{"type": "image", "source": {"type": "url", "url": "..."}}`.
- OpenAI format : `{"type": "image_url", "image_url": {"url": "...", "detail": "auto"}}`.
- **Stratégie** : type interne `ImagePart(data: bytes | str_url, media_type)`. Provider-specific sérialisation.

### Token counting
- **tiktoken** (`cl100k_base`, `o200k_base`) : précis pour OpenAI, approximatif pour les autres (±10%).
- **anthropic** : endpoint `POST /v1/messages/count_tokens` (gratuit, pas rate limited agressivement). SDK `client.messages.count_tokens(...)`.
- **Gemini** : `model.count_tokens(...)`.
- **Stratégie** : `provider.count_tokens(messages) -> int`. Anthropic → endpoint direct. OpenRouter → tiktoken heuristique (`o200k_base` pour GPT-5, `cl100k_base` fallback) ou appel `GET /generation?id=...` après requête pour obtenir le vrai count. DeepSeek/Qwen → tiktoken approximatif, accepter ±15%.

### Retry strategy
- **Erreurs transitoires** : `429 Too Many Requests`, `500/502/503`, Anthropic `529 Overloaded`.
- **Backoff exponentiel** : 1s, 2s, 4s, 8s, 16s max ; jitter ±20%.
- **Respect `Retry-After`** header quand présent (429 OpenAI, 529 Anthropic).
- **Idempotency** : OpenAI et Anthropic supportent `Idempotency-Key` header → permet de retry sans double-facturation sur timeouts.
- **Stratégie** : wrapper `tenacity` (ou custom) avec un decorator `@retry_api`. Sur 529 Anthropic, fallback automatique vers OpenRouter (`models: ["anthropic/claude-sonnet-4.6", ...]`).

### Cost tracking
- **OpenAI / OpenRouter** : `usage.prompt_tokens`, `usage.completion_tokens`, `usage.prompt_tokens_details.cached_tokens`. Prix à lookup (pas retourné dans la réponse, sauf OR qui expose `/generation`).
- **Anthropic** : `usage.input_tokens`, `usage.output_tokens`, `usage.cache_read_input_tokens`, `usage.cache_creation_input_tokens`.
- **OpenRouter** : en plus, header `X-OR-Cost: 0.00123` (USD) - source canonique, évite de maintenir une table de prix.
- **Stratégie** : `CompletionResponse.cost_usd: float | None`. Pour OR → header. Pour Anthropic/OpenAI direct → lookup prix local (JSON de prix versionné dans le package, mis à jour mensuellement).

---

## Livrable - Design final

### Types canoniques

```python
from __future__ import annotations
from typing import Protocol, Literal, Iterator, Any
from dataclasses import dataclass, field

Role = Literal["system", "user", "assistant", "tool"]

@dataclass
class TextPart:
    text: str

@dataclass
class ImagePart:
    data: bytes | str          # bytes = base64-encode on serialize, str = URL
    media_type: str = "image/png"

@dataclass
class ToolUsePart:
    id: str
    name: str
    input: dict[str, Any]

@dataclass
class ToolResultPart:
    tool_use_id: str
    content: str | list[TextPart | ImagePart]
    is_error: bool = False

@dataclass
class ThinkingPart:
    text: str
    signature: str | None = None   # Anthropic extended thinking signature (opaque, must round-trip)

ContentPart = TextPart | ImagePart | ToolUsePart | ToolResultPart | ThinkingPart

@dataclass
class Message:
    role: Role
    content: list[ContentPart]

@dataclass
class Tool:
    name: str
    description: str
    input_schema: dict[str, Any]   # JSON Schema, OpenAI `parameters` style

@dataclass
class ThinkingConfig:
    enabled: bool = False
    budget_tokens: int | None = None
    effort: Literal["low", "medium", "high"] | None = None

@dataclass
class Usage:
    input_tokens: int
    output_tokens: int
    cached_tokens: int = 0
    cache_write_tokens: int = 0
    reasoning_tokens: int = 0

@dataclass
class CompletionResponse:
    message: Message                      # assistant message with all parts (text, tool_use, thinking)
    stop_reason: Literal["end_turn", "tool_use", "max_tokens", "stop_sequence", "refusal"]
    usage: Usage
    cost_usd: float | None = None
    raw: dict[str, Any] = field(default_factory=dict)  # escape hatch

@dataclass
class StreamChunk:
    kind: Literal["text", "tool_use_start", "tool_use_delta", "thinking", "usage", "stop"]
    text: str | None = None
    tool_use: ToolUsePart | None = None
    partial_json: str | None = None
    usage: Usage | None = None
    stop_reason: str | None = None


class Provider(Protocol):
    name: str

    def complete(
        self,
        messages: list[Message],
        *,
        model: str,
        tools: list[Tool] | None = None,
        tool_choice: Literal["auto", "none", "required"] | str = "auto",
        max_tokens: int = 4096,
        temperature: float = 1.0,
        thinking: ThinkingConfig | None = None,
        cache_breakpoints: list[int] | None = None,   # indices in messages list
        stop: list[str] | None = None,
    ) -> CompletionResponse: ...

    def stream(
        self,
        messages: list[Message],
        *,
        model: str,
        tools: list[Tool] | None = None,
        **kwargs,
    ) -> Iterator[StreamChunk]: ...

    def count_tokens(self, messages: list[Message], model: str) -> int: ...
```

### Trois providers concrets

```python
# providers/anthropic.py
import anthropic

class AnthropicProvider:
    name = "anthropic"
    def __init__(self, api_key: str | None = None):
        self.client = anthropic.Anthropic(api_key=api_key)

    def complete(self, messages, *, model, tools=None, cache_breakpoints=None, thinking=None, **kw):
        anth_msgs, system = _to_anthropic_messages(messages, cache_breakpoints)
        body = {
            "model": model,
            "messages": anth_msgs,
            "system": system,
            "max_tokens": kw.get("max_tokens", 4096),
        }
        if tools:
            body["tools"] = [{"name": t.name, "description": t.description, "input_schema": t.input_schema} for t in tools]
        if thinking and thinking.enabled:
            body["thinking"] = {"type": "enabled", "budget_tokens": thinking.budget_tokens or 8000}
        resp = self.client.messages.create(**body)
        return _from_anthropic_response(resp)


# providers/openrouter.py
from openai import OpenAI

class OpenRouterProvider:
    name = "openrouter"
    def __init__(self, api_key: str | None = None, app_url: str = "", app_title: str = ""):
        self.client = OpenAI(
            base_url="https://openrouter.ai/api/v1",
            api_key=api_key or os.environ["OPENROUTER_API_KEY"],
            default_headers={"HTTP-Referer": app_url, "X-Title": app_title},
        )

    def complete(self, messages, *, model, tools=None, cache_breakpoints=None, **kw):
        oai_msgs = _to_openai_messages(messages, cache_breakpoints, is_anthropic_model=model.startswith("anthropic/"))
        body = {"model": model, "messages": oai_msgs, "max_tokens": kw.get("max_tokens", 4096)}
        if tools:
            body["tools"] = [{"type": "function", "function": {"name": t.name, "description": t.description, "parameters": t.input_schema}} for t in tools]
            body["tool_choice"] = kw.get("tool_choice", "auto")
        resp = self.client.chat.completions.create(**body)
        return _from_openai_response(resp, cost_header=resp._response.headers.get("X-OR-Cost"))


# providers/codex_subscription.py  (subprocess codex app-server, JSON-RPC stdio)
import subprocess, json, threading

class CodexSubscriptionProvider:
    """Spawn the official `codex` binary and talk JSON-RPC. Uses the user's ChatGPT subscription."""
    name = "codex_subscription"
    def __init__(self, codex_bin: str = "codex"):
        self.proc = subprocess.Popen(
            [codex_bin, "app-server", "--listen", "stdio://"],
            stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
            text=True, bufsize=1,
        )
        self._id = 0
        self._lock = threading.Lock()

    def _rpc(self, method: str, params: dict) -> dict:
        with self._lock:
            self._id += 1
            req = {"jsonrpc": "2.0", "id": self._id, "method": method, "params": params}
            self.proc.stdin.write(json.dumps(req) + "\n"); self.proc.stdin.flush()
            while True:
                line = self.proc.stdout.readline()
                if not line: raise RuntimeError("codex died")
                msg = json.loads(line)
                if msg.get("id") == self._id:
                    if "error" in msg: raise RuntimeError(msg["error"])
                    return msg["result"]

    def complete(self, messages, *, model="gpt-5", tools=None, **kw):
        thread = self._rpc("thread.start", {"model": model})
        # Codex gère sa propre boucle tool_use → on passe le user message et on collecte le final_response.
        user_text = _flatten_messages_to_text(messages)
        result = self._rpc("thread.run", {"thread_id": thread["id"], "input": user_text})
        return _codex_result_to_completion(result)
```

### Discussion : faut-il un LiteLLMProvider fallback ?

**Pour** : permet de brancher Bedrock, Vertex, Ollama, vLLM, Groq natif, 100+ autres sans code.
**Contre** : dépendance lourde (~50 packages transitifs), bugs historiques sur tool_calling Anthropic, perd le contrôle fin du caching.

**Décision** : oui, **mais en extra optionnel** (`pip install i-agents[litellm]`). Le core package n'a que `anthropic` + `openai`. Le `LiteLLMFallbackProvider` n'est importé qu'à la demande et ne tourne jamais pour Anthropic/OpenRouter/Codex (les trois chemins principaux).

```python
# providers/litellm_fallback.py  (optionnel)
class LiteLLMFallbackProvider:
    name = "litellm"
    def __init__(self):
        import litellm
        self.litellm = litellm
    def complete(self, messages, *, model, tools=None, **kw):
        resp = self.litellm.completion(model=model, messages=_to_openai_messages(messages), tools=_to_openai_tools(tools), **kw)
        return _from_openai_response(resp)
```

### Registry & factory

```python
# providers/__init__.py
_REGISTRY: dict[str, type[Provider]] = {}

def register(cls): _REGISTRY[cls.name] = cls; return cls

def make_provider(spec: str, **kw) -> Provider:
    """spec = 'anthropic' | 'openrouter' | 'codex_subscription' | 'litellm:<model>'"""
    name, _, rest = spec.partition(":")
    return _REGISTRY[name](**kw)
```

### Résumé des décisions

1. **Écrire notre abstraction**, pas wrapper litellm comme couche principale.
2. **3 providers first-class** : Anthropic direct, OpenRouter (OpenAI-compat), Codex subscription (subprocess).
3. **Canonical format interne** = blocks Anthropic-style (plus riche que OpenAI : thinking, tool_use, tool_result en content parts).
4. **Traduction bidirectionnelle** : Anthropic passe tel quel, OpenRouter fait le mapping blocks ↔ tool_calls + system extract + cache_control.
5. **Caching** : `cache_breakpoints: list[int]` dans l'API, mappé vers `cache_control` Anthropic, ignoré ailleurs.
6. **Thinking** : `ThinkingConfig` unifié, signatures Anthropic round-trippées, reasoning_tokens reportés dans Usage.
7. **Streaming** : `StreamChunk` union discriminée, tool_calls réassemblés avant exposition.
8. **Retry** : tenacity decorator, respect `Retry-After`, fallback 529 Anthropic → OpenRouter.
9. **Cost** : header `X-OR-Cost` pour OR, table de prix locale pour les autres.
10. **LiteLLM** : extra optionnel pour providers long-tail (Bedrock, Vertex, Ollama, etc.).


