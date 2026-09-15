# laura-rules — umgezogen nach laura-pa

> **Dieses Repo wird nicht mehr gepflegt.** Lauras Regeln liegen seit dem 14.09.2026 mit ihrer
> ganzen Historie im Repo **laura-pa**, dort unter `rules/` (Plan `0935-rules-zurueck-nach-laura-pa`,
> uebernommen per `git subtree` ab Commit `511e097`). Hier wird nichts mehr geaendert; das Repo
> bleibt als Historie und als Anker des eingefrorenen Cockpit-Submoduls `vendor/laura-rules`
> (admin-gettheflo) bestehen.
>
> Grund des Umzugs: als eigenes Repo hatte eine Regeldatei keinen Arbeitsbaum — jede Aenderung galt
> sofort in jeder laufenden Sitzung und konnte weder auf einem Zweig reifen noch durch ein Gate. Der
> urspruengliche Grund fuer die Trennung (das Cockpit baute seinen Stimm-Systemprompt aus dem
> Submodul) ist seit dem 03.09.2026 stillgelegt.

Lauras Wesenskern als geteilte Quelle (rules/*.md).

**Single Source of Truth** für:
- Lauras Voice & Charakter (`voice.md`)
- Kommunikations-Konventionen (`communication.md`)
- Guardrails (`guardrails.md`)
- Architecture-Patterns (`architecture.md`)
- Coding-Regeln (`coding.md`, `paths:`-gebunden — lädt nur bei Code-Arbeit im Projektbaum, sonst über die Lade-Liste; AP-0336, 04.09.2026)

## Konsumenten

- **Mac/Filesystem:** `rules/` ← Florians lokaler Edit-Pfad
- **Cockpit/admin-gettheflo:** Submodule `vendor/laura-rules/` liegt noch, der Codegen `scripts/build-persona.mjs` ist seit 03.09.2026 **stillgelegt** (Florian-Entscheid) — der Voice-Systemprompt ist eingefroren, Edits hier erreichen das Cockpit nicht mehr automatisch. Neuer Ansatz offen.
- **Zukünftige Clients:** Claude Code auf Hetzner-VM, VS Code mit Claude-Plugin etc.

## Read-Only-Vertrag für Konsumenten

Edits passieren in `rules/` (Mac). Push hierhin = Single Source of Truth aktualisiert. Konsumenten ziehen den neuen Stand via `git submodule update --remote` oder analoge Mechanismen.

## Lizenz

Public-Repo aus pragmatischen Gründen (CI-Builds wie Vercel können ohne PAT-Auth klonen). Inhalte sind nicht sensitiv — Persona-Definition für Florians persönliche Assistentin Laura. Kein Anspruch auf Wiederverwendung; Persona-Identität bleibt bei Florian.

---

Hintergrund: Identitäts-Backbone Pfad 3, Wave 1 Schritt #2 / Submodule-Refactor 17.05.2026.
