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

