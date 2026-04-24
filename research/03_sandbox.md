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

---

## 1. Modèle de menace

On part du principe que **le LLM est hostile par défaut** (prompt injection, hallucination, sycophancy sur un rm -rf). Les vecteurs à couvrir:

| # | Scénario | Impact | Vecteur typique |
|---|----------|--------|-----------------|
| T1 | `rm -rf ~` hallucination | Perte de données utilisateur | Bash tool non-filtré |
| T2 | Lecture de `~/.ssh/id_rsa`, `~/.aws/credentials`, `.env` | Exfiltration secrets | Read tool + output LLM vers provider |
| T3 | Écriture `~/.bashrc`, `~/.zshrc`, cronjobs, `~/.ssh/authorized_keys` | Persistance malware | Edit tool ciblant hors-cwd |
| T4 | Egress réseau vers attacker.com (POST secret) | Exfiltration via prompt-injection dans fichier tiers | `curl`, `python -c "requests.post..."` |
| T5 | `pip install malicious-package` avec post-install hook | Exécution code arbitraire hôte | npm/pypi non-filtré |
| T6 | `docker run --privileged` ou accès `/var/run/docker.sock` | Évasion conteneur, root hôte | Socket monté, socket leak |
| T7 | Fork bomb / mémoire infinie | DoS machine | Pas de limite cgroup |
| T8 | Process zombies non reapable | DoS long-terme | PID 1 non-init |
| T9 | Binding port privilégié | Conflit host / spoofing | CAP_NET_BIND_SERVICE |
| T10 | Auto-modification du harness pour retirer les gardes | Escalade définitive | Agent écrit dans son propre code |

**Principe directeur:** la sandbox doit être **fail-closed** (par défaut tout bloqué, whitelist explicite), avec défense en profondeur (FS + network + syscall + user namespace + cgroup).

---

## 2. Tour d'horizon

### 2.1 Claude Code (Anthropic)

Source primaire: https://code.claude.com/docs/en/sandboxing et l'open-source package `@anthropic-ai/sandbox-runtime`.

- **macOS:** Seatbelt (`sandbox-exec` avec profil `.sb`). Enforce au niveau kernel via `com.apple.security.sandbox`.
- **Linux/WSL2:** `bubblewrap` (bwrap) + `socat` comme proxy réseau.
- **Config via `settings.json`** (`~/.claude/settings.json`, `<project>/.claude/settings.json`, `managed`):

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true,
    "filesystem": {
      "allowWrite": ["~/.cache/pnpm", "~/.cache/pip", "/tmp/build"],
      "denyRead": ["~/.ssh", "~/.aws", "~/.gnupg", "~/.config/gcloud"],
      "allowRead": ["."]
    },
    "network": {
      "allowedDomains": [
        "registry.npmjs.org",
        "*.npmjs.org",
        "pypi.org",
        "files.pythonhosted.org",
        "api.anthropic.com",
        "github.com",
        "*.githubusercontent.com"
      ],
      "allowManagedDomainsOnly": true,
      "allowLocalBinding": true,
      "httpProxyPort": 8080
    }
  }
}
```

- **Modèle auto-allow:** les commandes bash qui tiennent dans le sandbox s'exécutent sans prompt, celles qui tentent de sortir tombent dans le flow permissions classique.
- **Limite réseau:** proxy HTTP/HTTPS par domaine (pas inspection TLS par défaut). Warning explicite sur domain fronting.
- **Faille connue (Ona 2026):** Claude Code a un "escape hatch" `dangerouslyDisableSandbox` activé par défaut — à **désactiver** via `"allowUnsandboxedCommands": false` sinon le sandbox est contournable.
- **`enableWeakerNestedSandbox`:** pour tourner Claude Code DANS un conteneur Docker (où l'user namespace du bwrap ne peut pas se créer). Affaiblit l'isolation — à n'activer que si Docker fournit déjà l'isolation externe.
- **Ce qui n'est PAS sandboxé:** les tools `Read`, `Edit`, `Write` (ils passent par le layer permission, pas OS-level). Seul le Bash tool et ses enfants sont dans bwrap/seatbelt.

### 2.2 OpenAI Codex CLI (codex-rs)

Cloné dans `/home/user/i_agents/research/clones/codex/codex-rs/`. C'est plus strict que Claude Code.

- **macOS:** Seatbelt via `sandbox-exec -p <sbpl>`. Profile de base trouvé dans `codex-rs/sandboxing/src/seatbelt_base_policy.sbpl`: `(deny default)` puis whitelist sysctl/iokit/mach-lookup (inspiré de Chrome). Réseau séparé dans `seatbelt_network_policy.sbpl`.
- **Linux:** **deux couches** combinées:
  1. **Bubblewrap** (bwrap) pour le filesystem — mount `--ro-bind /`, puis writable roots explicites, avec protection `.git` et `.codex` en read-only même à l'intérieur d'un root writable. Network namespace isolé (`--unshare-net`) sauf mode FullAccess.
  2. **seccomp** + `PR_SET_NO_NEW_PRIVS` appliqué in-process (filtre syscalls).
  3. **Landlock LSM** comme 3ᵉ couche filesystem (plus fine que bwrap mount).
- **3 modes** (`~/.codex/config.toml`):
  - `read-only` (défaut): lecture partout, aucune écriture, aucun réseau.
  - `workspace-write`: lecture partout, écriture limitée au cwd + `~/.codex/memories` + `/tmp`, réseau bloqué.
  - `danger-full-access`: pas de sandbox (équivalent `--yolo`).
- **Orthogonal:** `approval_policy` (`never` / `on-request` / `on-failure`). Le preset `--full-auto` = `workspace-write` + `on-request`.
- **Implémentation notable:** l'exécutable `codex-linux-sandbox` est un helper qui reçoit la policy en JSON, fait les bwrap mounts, applique seccomp, puis exec la commande. Ça permet que Codex se self-invoque comme helper sandbox — moins de surface d'attaque.

### 2.3 OpenHands (All-Hands)

Architecture client-serveur sur Docker (https://docs.all-hands.dev/modules/usage/architecture/runtime):

- L'utilisateur fournit une image Docker de base. OpenHands construit une "OH runtime image" par-dessus (ajoute le runtime client).
- Au démarrage, lance le conteneur, spawne un ActionExecutor avec bash + Jupyter + Playwright/Chromium.
- Backend parle au client via **REST API** (actions -> observations). C'est la frontière de confiance.
- Isolation = Docker standard (namespaces + cgroups). Pas d'isolation VM par défaut.
- En 2026, issue #13203 propose un backend **QEMU microVM** (alternatif à Docker) pour un niveau d'isolation supérieur.
- Daytona fournit un runtime tiers pour OpenHands avec microVM intégré.

### 2.4 aider

aider n'implémente **pas** de sandbox intégré. La recommandation officielle est de lancer aider dans Docker:

```
docker run -it --rm -v "$(pwd):/app" paulgauthier/aider-full --model gpt-5
```

Pas de restriction réseau, pas de seccomp — dépend entièrement de l'user pour l'isolation externe. C'est une architecture "bring your own sandbox".

### 2.5 E2B (e2b-dev)

Cloud-first. Firecracker microVMs, mêmes que AWS Lambda.
- Boot < 200ms, kernel dédié par sandbox, network namespace dédié.
- Defense-in-depth: `jailer` process autour du VMM (cgroups + namespaces + drop privileges).
- SDK Python/JS. Session jusqu'à 24h. ~$0.05/h pour 1 vCPU.
- **Use case:** parfait si l'agent tourne en serveur (backend), coûte cher si on veut juste exécuter localement sur machine dev.

### 2.6 Modal

- Sandbox basé sur **gVisor** (pas Firecracker).
- Plateforme AI plus large (inference, training, batch GPU).
- Sandbox lifetime 5min défaut (24h max).
- Pay-as-you-go à la seconde.
- Meilleur pour scaler une flotte d'agents sur GPU que pour un dev local.

### 2.7 Daytona

- Pivot vers AI code exec en 2026.
- **Docker-based** (pas microVM). Cold start < 90ms (le plus rapide).
- Kernel partagé — isolation plus faible qu'E2B/Modal.
- Intéressant pour computer-use (browser automation intégré).

### 2.8 Firecracker (AWS / standalone)

- MicroVM KVM, ~125ms boot, ~5 MiB mémoire overhead par VM.
- Kernel invité séparé → isolation très forte (une CVE kernel guest ne touche pas l'host).
- Sous le capot d'E2B, Fly.io Machines, AWS Lambda.
- **Complexité self-host élevée:** il faut un jailer, une image rootfs, un initramfs, un bridge network. Pas "5 min docker run".

### 2.9 gVisor (runsc)

- Application kernel écrit en Go, intercepte les syscalls en userspace (Sentry).
- OCI-compatible (`runsc` remplace `runc`). `docker run --runtime=runsc`.
- **Overhead:** <3% CPU typique, mais I/O-heavy peut être 2-200× plus lent (open/close sur tmpfs externe: 216×).
- Sweet spot: workloads CPU-bound, pas build-heavy.

### 2.10 Bubblewrap (bwrap)

- Low-level unprivileged, utilisé par Flatpak, Claude Code, Codex CLI.
- Namespaces: user/mount/pid/uts/ipc/net + seccomp + drop caps.
- Pas d'abstraction — tout est flag CLI (`--ro-bind`, `--tmpfs`, `--unshare-*`).
- **Pré-requis:** user namespaces activés (`/proc/sys/kernel/unprivileged_userns_clone=1`). Sur WSL1: KO. Dans Docker: KO sauf privilégié.

### 2.11 Firejail

- Desktop-oriented, profiles pré-écrits (Firefox, VLC…).
- Setuid par défaut — controversé, a eu des CVE d'escalation.
- Pas adapté pour un agent CLI headless; préférer bwrap ou nsjail.

### 2.12 nsjail (Google)

- Process isolation via namespaces + seccomp + cgroups.
- Conçu pour juges de code (CTF, online judges).
- Plus granulaire que bwrap en ressources, moins en filesystem.
- Viable pour exec one-shot de code LLM non-trusté.

### 2.13 Landlock LSM

- Primitive **kernel** (5.13+, production-ready 6.7+).
- Hiérarchique, unprivileged — le process restreint ses propres droits.
- Ne couvre QUE le filesystem (pas le réseau avant 6.7, pas les syscalls).
- Utilisé par Codex CLI comme 3ᵉ couche par-dessus bwrap.

### 2.14 seccomp-bpf

- Filtre syscalls via BPF program.
- Deny-list (block `mount`, `reboot`…) ou allow-list (whitelist stricte, fragile).
- Docker fournit un default seccomp profile qui bloque ~44 syscalls dangereux.
- Facile à sur-restreindre et casser une app.

### 2.15 Seatbelt / sandbox-exec (macOS)

- Framework natif macOS, profile TinyScheme (`.sb`). Officiellement déprécié mais encore fonctionnel et utilisé par Chrome, Safari, Claude Code, Codex.
- `sandbox-exec -f profile.sb -- command`.
- Très granulaire (syscall class + mach-lookup + file-read* + network bind/out).
- Pas de vrai équivalent cross-platform.

### 2.16 Docker Sandboxes (Docker Desktop 4.50+, 2026)

Produit récent dédié au cas "coding agent":
- MicroVM par sandbox (hypervisor isolation), Docker daemon propre dans la VM.
- Network isolé, pas de reach vers host/localhost.
- **Credential proxy:** clé API injectée par le host dans les requêtes sortantes, jamais dans le VM. L'agent ne peut pas lire sa propre clé.
- Permet à l'agent de `docker build`/`docker run` dedans sans compromettre l'host.
- Supporte Claude Code, Codex, Gemini, Copilot, Kiro officiellement.

### 2.17 WASM (wasmtime, Pyodide, Monty)

- Mémoire linéaire bounded-check, pas d'accès host sans capability WASI explicite.
- Wasmtime memory-safety formellement vérifiée.
- **Limite majeure:** ne fait tourner que du Python pur subset ou langages recompilés WASM. Pas `docker`, pas `apt`, pas `npm install`, pas subprocess vers clang etc.
- **Sweet spot:** tool "run_code" granulaire (calc, data pipeline rapide), PAS l'environnement d'un agent software engineer.

### 2.18 DevContainers (VSCode / Open Container Initiative)

- `devcontainer.json` + image Docker/Compose.
- Par défaut bind-mount le workspace depuis l'host → l'agent peut **détruire les vrais fichiers**.
- Pour isoler: `"workspaceMount": "type=tmpfs,destination=/workspace"` + copie/sync explicite (mais perd la sync live).
- Ergonomie IDE excellente (rebuild container button), sécurité moyenne (root dans container, pas de network policy).
