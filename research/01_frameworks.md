# Recherche: Frameworks agentiques pour le code

**Statut: en cours**
**Date: 2026-04-24**

## Objectif

Identifier les frameworks agentiques existants pour faciliter la construction d'une librairie Python modulaire permettant à un LLM de coder des projets. L'utilisateur veut:

1. Modules providers API (OpenRouter, Codex via subscription ChatGPT)
2. Loop agentique
3. Module de tools
4. Mécanisme original de "self-reset" (l'agent choisit quels fichiers relire pour repartir avec contexte frais mais pré-chargé)
5. Auto-modification du harness (ajouter tools à la volée, analyser succès/échecs pour ajuster prompts/tools)
6. Sandboxing Docker/podman

## Plan de recherche

- [ ] Claude Code (Anthropic) — architecture, /compact, tools
- [ ] OpenAI Codex CLI (2025, open source) — branchement via ChatGPT subscription
- [ ] aider (paul-gauthier/aider)
- [ ] OpenHands / OpenDevin (All-Hands-AI/OpenHands)
- [ ] SWE-agent (princeton-nlp/SWE-agent)
- [ ] Cline (cline/cline)
- [ ] Roo-Code (fork de Cline)
- [ ] Goose (block/goose)
- [ ] smol-developer, GPT-Engineer, GPT-Pilot, AutoCodeRover, MetaGPT
- [ ] Devin (Cognition) — closed source
- [ ] crewAI, AutoGen, LangGraph
- [ ] Cloner les 3-4 plus pertinents et analyser
- [ ] Synthèse finale

## Framework par framework

### 1. Claude Code (Anthropic) — officiel

- **URL**: https://github.com/anthropics/claude-code (propriétaire, non ouvert) + docs https://code.claude.com
- **Langage**: TypeScript/JavaScript minifié (harness), shippé comme binaire npm
- **Licence**: propriétaire (EULA Anthropic)
- **Modulaire**: oui (controller / tool layer / MCP / hooks / skills / subagents)
- **Mécanisme reset/compact**: OUI — pipeline 5 couches en cascade, auto à ~98% token budget
- **Auto-modification**: partielle — via `skills` (fichiers markdown déposés pour apprendre de nouveaux comportements) et `hooks` (settings.json)
- **Sandbox**: oui (modes permission, bash sandboxing optionnel sur macOS/Linux, Dockerfiles fournis)
- **Points clés**:
  - Core = "simple while-loop that calls the model, runs tools, and repeats" — le reste est autour
  - 7 modes de permission + classifier ML pour auto-approuver certains calls
  - Pipeline de compaction: strip images → group tool_use/tool_result → cap oversized tool results → summarize older messages → hard truncation
  - Extensibilité: MCP (remote tools), plugins, skills (md files lazy-loaded), hooks (scripts user)
  - Session storage append-only, checkpoints
  - Subagent delegation (Task tool) pour déléguer avec un contexte propre
- **Verdict**: s'inspirer fortement pour architecture (loop minimaliste + extensibilité latérale). Pas de code réutilisable car fermé.

### 2. OpenAI Codex CLI — officiel

- **URL**: https://github.com/openai/codex
- **Langage**: Rust (core) + Python SDK généré + TypeScript (legacy, migré)
- **Licence**: Apache 2.0 (open source)
- **Modulaire**: oui (core agent / exec / mcp / tui / app-server)
- **Mécanisme reset/compact**: oui (compact via slash command, session rollouts)
- **Auto-modification**: non native, mais supporte custom prompts/tools via MCP
- **Sandbox**: oui (native, Landlock Linux + Seatbelt macOS, profils restreints)
- **Points clés**:
  - Stack hybride Rust + SDK Python généré
  - Supporte l'auth ChatGPT (subscription) OU API key — c'est LA raison pour laquelle l'user veut s'y brancher
  - Profils `--full-auto`, `--suggest`, `--ask`
  - Streaming responses, approval modes granulaires
- **Verdict**: Le Rust complique la réutilisation directe. Mais: on peut **consommer le `app-server` / responses API** pour brancher notre agent Python sur l'auth Codex subscription. À étudier en profondeur.

### 3. aider (Aider-AI/aider)

- **URL**: https://github.com/Aider-AI/aider
- **Langage**: Python
- **Licence**: Apache 2.0
- **Modulaire**: oui (coders/, commands/, io/, models/)
- **Mécanisme reset/compact**: partiel (`/clear` + summarizer LLM). Repo-map régénéré à chaque tour.
- **Auto-modification**: non
- **Sandbox**: non (tourne directement; git commit à chaque change)
- **Points clés**:
  - Repo-map via tree-sitter (AST-based) — très intéressant pour alimenter contexte LLM
  - Architect/Editor split: un modèle "planifie", un autre "édite"
  - Commit git automatique par edit
  - Support ~100 LLMs via litellm (réutilisable directement)
- **Verdict**: À CLONER ET ANALYSER. Beaucoup de code Python mûr à réutiliser/piquer.

### 4. OpenHands / OpenDevin (All-Hands-AI/OpenHands)

- **URL**: https://github.com/All-Hands-AI/OpenHands
- **Langage**: Python + TypeScript (front)
- **Licence**: MIT
- **Modulaire**: oui (openhands.agenthub, .events, .runtime, .llm, .controller)
- **Mécanisme reset/compact**: oui (condenser, event stream abstraction)
- **Auto-modification**: partiel (micro-agents, addable via markdown)
- **Sandbox**: OUI, excellent — Docker container par task, REST API server dans le container, actions bash/python/browser
- **Points clés**:
  - Event stream architecture: Action/Observation pairs, agent émet Action, runtime émet Observation
  - Runtime abstrait: Docker, Modal, E2B, local — swappable
  - CodeActAgent = code-generating agent
  - Micro-agents loadable on demand
- **Verdict**: À CLONER ET ANALYSER. Architecture très proche de ce que l'user veut, Python, très modulaire.

### 5. SWE-agent (SWE-agent/SWE-agent)

- **URL**: https://github.com/SWE-agent/SWE-agent
- **Langage**: Python
- **Licence**: MIT
- **Modulaire**: oui (agent / environment / tools)
- **Mécanisme reset/compact**: oui (history processors)
- **Auto-modification**: non (mais tool bundles configurables en YAML)
- **Sandbox**: oui (Docker via `swe-rex`)
- **Points clés**:
  - Concept d'ACI (Agent-Computer Interface): tools LM-centric (view, edit, scroll, search), feedback formaté pour le LM
  - Config YAML pour tout (templates, tools, env)
  - History processors modulaires (summarizer, last_n_observations…)
  - Benchmark-first mindset (SWE-bench)
- **Verdict**: À CLONER ET ANALYSER. Code Python épuré, ACI et history processors pertinents.

### 6. Cline (cline/cline)

- **URL**: https://github.com/cline/cline
- **Langage**: TypeScript
- **Licence**: Apache 2.0
- **Modulaire**: oui (Controller, StateManager, Task, ApiHandler, ToolExecutor, McpHub)
- **Mécanisme reset/compact**: oui
- **Auto-modification**: OUI — "Ask Cline to add a tool" → crée un MCP server et l'installe
- **Sandbox**: non natif (tourne dans VS Code), permissions user-granted
- **Points clés**:
  - HostProvider singleton pour abstraire VS Code vs CLI
  - gRPC-over-postMessage entre webview et extension
  - MCP auto-creation (unique!)
- **Verdict**: S'inspirer du mécanisme d'auto-création de tools MCP. Code TS donc pas réutilisable directement.

### 7. Roo-Code (fork de Cline)

- Fork de Cline avec des améliorations multi-mode (Code/Architect/Ask modes). Même stack TS.
- **Verdict**: même verdict que Cline; explorer les modes pour inspiration.

### 8. Goose (block/goose → Linux Foundation AAIF)

- **URL**: https://github.com/block/goose
- **Langage**: Rust
- **Licence**: Apache 2.0
- **Modulaire**: oui (goose core / goose-mcp / goose-cli / goose-server)
- **Mécanisme reset/compact**: oui (truncate + summarize)
- **Auto-modification**: via extensions MCP ajoutables
- **Sandbox**: non natif
- **Verdict**: Rust, pas direct réutilisable. Bonne référence architecture MCP-centric.

### 9. MetaGPT

- **URL**: https://github.com/geekan/MetaGPT
- **Langage**: Python
- **Licence**: MIT
- **Modulaire**: oui (Environment, Memory, Roles, Actions, Tools)
- **Mécanisme reset/compact**: limité
- **Auto-modification**: non
- **Sandbox**: non
- **Points clés**: multi-agent avec rôles fixes (PM, Architect, Engineer, QA). Approche "SOP".
- **Verdict**: Philosophie différente (multi-agent orchestré) — s'inspirer éventuellement pour le concept de rôles, mais trop opinionated pour réutilisation directe.

### 10. GPT-Engineer / smol-developer / GPT-Pilot

- GPT-Engineer (AntonOsika): "give prompt → full codebase" one-shot puis itératif. Python. Simple.
- smol-developer (smol-ai): même philosophie, très peu de code.
- GPT-Pilot: vise des apps complètes avec human-in-the-loop + agents spécialisés (Dev, Tech Lead, QA).
- **Verdict**: Trop simples (GPT-Engineer, smol-dev) ou trop couplés (GPT-Pilot). Ignorer pour réutilisation, lecture rapide pour idées.

### 11. AutoCodeRover

- **URL**: https://github.com/AutoCodeRoverSG/auto-code-rover
- **Langage**: Python
- **Licence**: GPL-3.0 (attention — viral)
- **Modulaire**: oui (context retrieval / patch generation / fault localization)
- **Mécanisme reset/compact**: non mis en avant
- **Auto-modification**: non
- **Sandbox**: Docker (via SWE-bench)
- **Points clés**:
  - APIs de recherche structure-aware (AST-based) — search_class, search_method_in_class, etc.
  - Fault localization statistique si tests dispos
- **Verdict**: Licence GPL-3.0 = **PROBLÈME** pour une lib qu'on veut permissive. S'inspirer conceptuellement des APIs de recherche AST-based.

### 12. Devin (Cognition) — closed source

- Planner LLM (décomposition goal → steps) + Executor léger qui choisit tool (shell/editor/browser) + sandbox workspace
- Loop "test-debug-fix" itératif
- 2.2: self-verify + auto-fix via computer-use
- **Verdict**: Ignorer pour réutilisation. Garder l'idée **planner/executor split** + **test-debug-fix loop**.

### 13. LangGraph / AutoGen / crewAI

- **LangGraph** (LangChain): graph-based workflow, state machines. Python + TS. Excellente pour workflows explicites, durable execution. **Surkill pour un harness simple** mais utilisable comme backbone si on veut state machine.
- **AutoGen** (Microsoft): conversation-centric multi-agent. Python. Bon pour groupes d'agents qui dialoguent.
- **crewAI**: rôles + tasks façon "team". Python. Plus opinionated que LangGraph.
- **Verdict**: Trop généralistes. On peut potentiellement bâtir sur LangGraph si on veut state persistence gratuit, mais au prix de lourdeur. **Probablement pas pour ce projet** qui vise quelque chose de plus léger et coder-spécifique.

---

## Analyse en profondeur (clones)

(section en cours de remplissage)

---
