# Recherche: Context Reset/Compaction & Self-Modifying Agents

**Statut: en cours**
**Date: 2026-04-24**

## Objectif

Documenter deux mécanismes pour un agent Python:
- **(A) Self-reset/compaction**: l'agent demande via tool call au harness de lire des fichiers, puis relance une session fraîche avec ces lectures en pré-contexte. "Mot à soi-même du futur".
- **(B) Auto-modification**: l'agent rajoute des tools à la volée, analyse success/failures, ajuste prompts/tools.

## Plan

1. Partie A - Compaction/memory:
   - Claude Code `/compact` prompt
   - Aider repo map
   - Cursor context
   - MemGPT / Letta
   - Cognition "Don't build multi-agents"
   - Anthropic "Effective agents"
   - Sleep-time compute, scratchpad, reflection
   - "Hand-off to future self" patterns
2. Partie B - Self-modifying:
   - Voyager (NVIDIA)
   - ADAS (Shengran Hu)
   - DSPy (Stanford)
   - Trace (Microsoft)
   - OPRO / APE
   - Gödel Agent / Gödel Machine
   - OpenInterpreter / smol-ai
3. Synthèse designs concrets

## Notes en cours

### 1. Claude Code `/compact` mechanism

Sources: blog posts de reverse-engineering (kirshatrov.com, sabrina.dev, karanprasad.com, Yuyz0112/claude-code-reverse sur GitHub, agiflow.io, signals.aktagon.com).

Findings clés:
- `/compact` peut être déclenché **manuellement** ou **automatiquement** quand le contexte approche de la limite.
- Mécanisme: un *system compact prompt* est chargé + un *compact prompt* est appendé à la fin du contexte courant. Le LLM est instruit de produire un résumé dans un format structuré précis (pas juste du prose libre).
- Le résumé produit devient l'**unique contenu** qui amorce la prochaine session (remplace l'historique).
- Stratégie escalatoire à **6 niveaux** décrite dans la reverse-eng:
  - Tier 1: microcompaction basée sur le temps - après 60+ min d'expiration du cache, les vieux `tool_result` sont clearés en gardant les 5 derniers. Pas d'appel API.
  - Tier 2: microcompaction cachée via `cache_edits` (paramètre Anthropic API) qui retire chirurgicalement des tool_results sans invalider le prefix caché -> ~76% d'économie sur les turns répétés.
  - Tier supérieurs: compaction LLM complète.
- **Circuit breaker**: après 3 échecs consécutifs de compaction, le système abandonne pour éviter les boucles infinies.
- Qui choisit ce qui est gardé: c'est le **LLM lui-même** guidé par un prompt système structuré. Pas de règle déterministe sur les fichiers - c'est le modèle qui choisit d'après son prompt.
- Implication pour notre design: le "quoi garder" est déléguable au LLM si le prompt est bien cadré.

**Structure exacte du compact prompt** (repo Piebald-AI/claude-code-system-prompts):
Le prompt demande au LLM de produire un résumé structuré en 9 sections:
1. Primary Request and Intent (explicite, en détail)
2. Key Technical Concepts
3. Files and Code Sections (fichiers lus/modifiés + pourquoi c'est important, incluant snippets verbatim)
4. Errors and fixes
5. Problem Solving
6. All user messages (tous les messages user, pas juste le dernier)
7. Pending Tasks
8. Current Work (ce qu'on faisait juste avant compaction)
9. Optional Next Step

Directive spéciale: "When you are using compact - please focus on test output and code changes. Include file reads verbatim."

-> Ce format est directement transposable pour notre **Design A (self-reset)**: on peut demander à l'agent de produire ce même template avant reset, puis le réinjecter comme premier message user dans la session fraîche.

### 2. Anthropic "Building effective agents"

Takeaways pertinents pour nos designs:
- Distinction centrale: **workflows** (code paths prédéfinis) vs **agents** (LLM dirige son propre flow). Les agents s'imposent seulement pour les problèmes open-ended où on ne peut pas hardcoder.
- 5 patterns d'orchestration: Prompt Chaining, Routing, Parallelization (sectioning/voting), **Orchestrator-Workers**, **Evaluator-Optimizer**.
- Le pattern Evaluator-Optimizer (feedback loop générateur/critique) est directement pertinent pour notre Design C (analyse success/failure).
- Le pattern Orchestrator-Workers est proche du self-reset: un orchestrator qui délègue à des workers frais.
- Sur les tools: "tool design deserves equivalent effort to prompts" - inclure exemples, edge cases, input format, boundaries. Eviter escaping/comptage manuel (poka-yoke).
- Mémoire/handoff: peu détaillé dans ce post, mais insiste sur "ground truth from the environment at each step" -> ne pas compter sur la mémoire LLM, re-lire les fichiers.
- Philosophie: "start simple, add agentic complexity only when simpler solutions fall short". Donc notre self-reset devrait être désactivable pour des tâches courtes.

### 3. Cognition "Don't build multi-agents"

Thèse: les architectures multi-agents sont fondamentalement peu fiables en prod. Préférer un agent **single-threaded** avec context management robuste.

Deux principes énoncés:
1. "Share context, and share full agent traces, not just individual messages"
2. "Actions carry implicit decisions, and conflicting decisions carry bad results"

Failure mode illustratif: Flappy Bird - subagent 1 fait un background style Super Mario, subagent 2 fait un bird sprite incompatible, l'agent final hérite d'un merge impossible. Les décisions se dispersent quand le contexte n'est pas partagé.

Recommandation pour les tâches longues: **introduire un LLM spécialisé dans la compression de l'historique** en "key decisions and events" avant que la window sature. C'est exactement le pattern /compact de Claude Code, et c'est exactement ce que vise notre Design A.

Implication directe pour nous:
- Préférer un seul agent qui se reset plutôt qu'un orchestrateur qui spawn des subagents parallèles.
- Si on doit faire du sub-agent, partager le trace complet, pas juste un message.
- Le self-reset doit préserver les "key decisions" et pas seulement les fichiers bruts - d'où la note textuelle associée.

### 4. MemGPT / Letta (arxiv 2310.08560)

Métaphore OS: mémoire hiérarchique comme RAM/disk. L'agent gère sa propre mémoire via tool calls.

**3 tiers de mémoire**:
- **Core memory** (in-context, taille fixe): blocs édités par l'agent via tools, pinned dans la window. Persona, user facts, current task. Analogue RAM.
- **Recall memory**: historique complet des interactions, persisté sur disque, searchable via text/date tools. L'historique raw quoi.
- **Archival memory**: knowledge structuré externe (vector DB, graph DB). Agent y écrit explicitement via tools read/write.

**Tools par défaut exposés à l'agent**:
- `core_memory_append`, `core_memory_replace` (édition in-context)
- `archival_memory_insert`, `archival_memory_search` (DB externe)
- `conversation_search`, `conversation_search_date` (recall memory)

**Control flow**: interrupts. Quand le contexte sature, un heartbeat événement interrompt l'agent pour qu'il déplace des choses core->archival.

Pour nous: Letta confirme qu'il est OK de donner des tools de gestion mémoire à l'agent. Mais leur modèle est complexe (3 tiers). Pour notre cas **code agent**, on peut simplifier: le "archival" est remplacé par **le filesystem lui-même** (les fichiers du projet sont la mémoire externe naturelle). Donc self-reset = "re-injecte ces fichiers + cette note" suffit, pas besoin d'un vector store.

### 5. Aider repo map

Pipeline:
1. Tree-sitter parse tous les fichiers (130+ langages), extrait definitions/references via `tags.scm`.
2. Construit un graphe dirigé: noeuds = fichiers, arêtes = dépendances (X définit, Y référence).
3. **Personalized PageRank** où le "bias" initial pointe vers les fichiers actuellement mentionnés dans la conversation.
4. Top-ranked definitions rendues en vue "elided" (signatures uniquement, corps cachés) jusqu'à saturer un token budget.

Points clefs:
- Automatique, pas besoin pour l'user de choisir les fichiers.
- Mis à jour à chaque tour (PageRank personalized selon la conversation courante).
- Cache par mtime pour les tags, fast.
- ~15B tokens/semaine processés.

Pour nos designs:
- Intéressant comme **fallback automatique** quand l'agent ne précise pas les files à reset.
- Mais pour notre Design A, on privilégie le **choix explicite** par l'agent (il sait quels fichiers sont load-bearing pour la suite de sa tâche). Le repo map peut être un tool auxiliaire `get_repo_map()` que l'agent peut appeler avant de décider du reset.
- Le rendu "elided" (signatures only) est une bonne optimisation: on peut injecter les signatures au lieu du contenu complet pour des fichiers "context" vs les fichiers "actif".

### 6. "Context handoff" / "baton passing" pattern

Terme le plus courant: **"handoff"** (OpenAI Agents SDK, LangChain, AG2, LiveKit l'utilisent tous). "Baton passing" est plus informel.

Implémentations types:
- **OpenAI Agents SDK**: handoff représenté comme un tool `transfer_to_<agent_name>`. Option `input_filter` pour forwarder seulement un subset.
- **LangChain**: handoffs dans multi-agent setups, agent reçoit soit la conv complète, soit filtrée, soit résumée.
- **AG2 / LiveKit**: pattern "Agent Transfer" où sub-agent hérite d'une vue sur la Session.

Design considerations reported:
- Forwarder la conv complète au receiving agent est souvent mauvais: pollution par reasoning interne, coût tokens.
- Recommandation: **résumer avant handoff** si la conv est longue.
- Par défaut beaucoup d'implems démarrent le receiving agent avec un contexte frais.

Pour notre self-reset (qui est un handoff "à soi-même du futur"):
- Suivre le pattern **tool-as-handoff** (tool call `prepare_reset` vu comme un transfer_to_self).
- Filtrer agressivement: on garde seulement files + note résumé + prompt système. **Pas** l'historique des tool calls.
- Analogue conceptuel: c'est une handoff vers un sub-agent qui s'appelle "moi mais demain matin".

### 7. Claude Code subagents (context passing)

Findings pertinents pour Design A:
- Le subagent démarre avec un **contexte frais**, zéro historique du parent.
- **Unique canal parent -> subagent**: la string prompt passée au tool `Agent`/`Task`. Le coordinator doit mettre dedans: file paths, error messages, decisions nécessaires.
- **Unique canal subagent -> parent**: le message final. Les tool calls intermédiaires et leurs results restent internes au subagent.
- Tool permissions au subagent: explicitement granted, subset du parent.
- Invocation: soit implicite (match sur description), soit explicite par nom, soit `@mention`.
- Context isolation complète: parent ne voit pas le raisonnement interne, subagent ne voit pas l'historique parent.

Analogie directe avec notre self-reset:
- Notre `prepare_reset(files, note)` = équivalent de créer un subagent "future-self" avec:
  - prompt initial = `note` (+ system prompt standard)
  - "pre-granted context" = contenu des `files`
  - continuation de la session = return du subagent (sauf que chez nous, c'est la session principale qui continue, pas le parent qui reprend)
- La leçon: un subagent frais avec juste un prompt bien écrit + les fichiers pertinents **fonctionne en pratique** dans Claude Code. Donc notre pari est plausible.

### 8. OpenAI Codex CLI - resume & /compact

Findings:
- **Sessions auto-sauvegardées** en **JSONL** dans `~/.codex/sessions/YYYY/MM/DD/`. Transcript complet persisté sans config.
- Commande `codex resume` ouvre un picker des sessions récentes (scope: cwd par défaut, `--all` pour toutes).
- Commande `/compact` (inline): résume la conv et remplace les earlier turns par le résumé. Free du contexte tout en gardant les détails critiques.
- Extended tasks: Codex peut tourner jusqu'à **7h en continu** en auto-compactant quand la window sature.
- `--cd` et `--add-dir` pour ajouter des racines de working dir avant resume.

Pour nos designs:
- **JSONL per-session sous un dossier daté** = bon pattern pour notre Design C (trace log).
- Le pattern "auto-compact on limit" confirme que c'est mainstream en 2025-2026 et que les utilisateurs attendent de la continuité.
- Resume = reload une session sauvée avec le même état repo. Différent de notre self-reset (qui *nettoie* le contexte volontairement), mais le mécanisme de sérialisation est similaire.

