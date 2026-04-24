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

(Sera complété au fur et à mesure)

---
