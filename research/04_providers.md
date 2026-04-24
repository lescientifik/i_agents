# Research: LLM Provider Integration

**Statut: en cours**
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

## Recherches en cours (partie A, C, D)...

