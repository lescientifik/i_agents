# Sandboxing pour agents LLM qui écrivent/exécutent du code

**Statut: en cours**
**Date: 2026-04-24**

## Plan

1. Modèle de menace
2. Tour d'horizon des approches
   - OS-level: Docker, Podman, bubblewrap, firejail, nsjail, landlock, seccomp, seatbelt
   - VM-level: Firecracker, gVisor
   - Harnesses existants: Claude Code, Codex CLI, OpenHands, aider
   - Cloud sandboxes: E2B, Modal, Daytona
   - WASM: wasmtime, wasmer
3. Tableau comparatif
4. Recommandation concrète (Dockerfile, compose, script de lancement, network policy)
5. Discussion: agent qui modifie son propre harness
