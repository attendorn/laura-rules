# laura-rules

Lauras Wesenskern als geteilte Quelle (rules/*.md).

**Single Source of Truth** für:
- Lauras Voice & Charakter (`voice.md`)
- Kommunikations-Konventionen (`communication.md`)
- Guardrails (`guardrails.md`)
- Architecture-Patterns (`architecture.md`)

## Konsumenten

- **Mac/Filesystem:** `~/Laura/rules/` ← Florians lokaler Edit-Pfad
- **Cockpit/admin-gettheflo:** Submodule `vendor/laura-rules/` liegt noch, der Codegen `scripts/build-persona.mjs` ist seit 03.09.2026 **stillgelegt** (Florian-Entscheid) — der Voice-Systemprompt ist eingefroren, Edits hier erreichen das Cockpit nicht mehr automatisch. Neuer Ansatz offen.
- **Zukünftige Clients:** Claude Code auf Hetzner-VM, VS Code mit Claude-Plugin etc.

## Read-Only-Vertrag für Konsumenten

Edits passieren in `~/Laura/rules/` (Mac). Push hierhin = Single Source of Truth aktualisiert. Konsumenten ziehen den neuen Stand via `git submodule update --remote` oder analoge Mechanismen.

## Lizenz

Public-Repo aus pragmatischen Gründen (CI-Builds wie Vercel können ohne PAT-Auth klonen). Inhalte sind nicht sensitiv — Persona-Definition für Florians persönliche Assistentin Laura. Kein Anspruch auf Wiederverwendung; Persona-Identität bleibt bei Florian.

---

Hintergrund: Identitäts-Backbone Pfad 3, Wave 1 Schritt #2 / Submodule-Refactor 17.05.2026.
