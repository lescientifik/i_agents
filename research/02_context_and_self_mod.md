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
