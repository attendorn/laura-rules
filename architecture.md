# Claude Code Architecture Patterns

> Kurzreferenz für Lauras Architekturentscheidungen. Bei Detailfragen: `~/Laura/memory/topics/claude-code-updates.md`

## Agent-Definition
- Rollen liegen in `~/Laura/agents/roles/` mit **YAML-Frontmatter** (name, description, model, tools, memory)
- **Named Agents** nutzen: `name: "kalender-scout"` beim Agent-Aufruf → per `SendMessage(to: "name")` in der Session wiederverwenden statt neu spawnen
- Model-Wahl: **Haiku** = Datensammlung, einfache MCP-Calls. **Sonnet** = Analyse, Synthese, Schreibarbeit. **Opus** = nur Hauptkontext (Laura selbst)

## Sub-Agent-Patterns
- **Hintergrund** (`run_in_background: true`): Für unabhängige Tasks (Downloads, Sync, Update-Check)
- **Vordergrund**: Wenn Ergebnis Gate für nächsten Schritt ist (Kalender-Scout → Briefing)
- **Proaktive Delegation**: >8k erwartete Token im Hauptkontext → automatisch an Sub-Agent
- **MCP-Vererbung**: Sub-Agents erben automatisch MCP Tools von dynamisch registrierten Servern (ab 2.1.101)
- **Keine Sub-Sub-Agents**: Agents spawnen keine eigenen Agents

## Agent Teams (aktiviert 11.04.2026)
- **Experimentell**, aktiviert via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json
- **Unterschied zu Sub-Agents**: Teammates kommunizieren UNTEREINANDER (Mailbox), nicht nur zurück zum Parent
- **Architektur**: Team Lead (Laura) + Teammates (eigene Claude-Instanzen) + Shared Task List + Mailbox
- **Tools**: `TeamCreate`, `TeamDelete`, `SendMessage` (zwischen Teammates)
- **Wann nutzen**: Komplexe Aufgaben mit Diskussion/Koordination (Research, parallele Reviews, Multi-Perspektiven, competing hypotheses)
- **Wann NICHT**: Einfache Einzelaufgaben, sequenzielle Arbeit, same-file Edits
- **Team-Größe**: 3-5 Teammates, 5-6 Tasks pro Teammate
- **Hooks**: `TeammateIdle`, `TaskCreated`, `TaskCompleted` für Quality Gates
- **Display**: In-Process (Shift+Down) oder Split-Panes (tmux/iTerm2)
- **Docs**: https://code.claude.com/docs/en/agent-teams

## Hooks (settings.json)
- **Conditional Hooks**: `matcher`-Feld für Tool-spezifische Hooks (z.B. nur bei "Write|Edit")
- **PreToolUse**: Validierung/Blocking vor Ausführung. Stdout → User-Feedback
- **PostToolUse**: Logging, Nachverarbeitung. `$CLAUDE_TOOL_INPUT` als JSON im Script lesen (nicht Shell-Expansion im command)
- **SessionStart**: Terminal-Titel, Context Injection via stdout
- **Plugin-Hooks** (ab 2.1.94): Hooks im Skill-Frontmatter funktionieren jetzt zuverlässig
- **Aktive Hooks**: Skill-Usage-Log (PostToolUse/Skill), Test-Coverage-Check + Gedankenstrich-Warnung (PostToolUse/Write|Edit), Destruktive-Commands-Block (PreToolUse/Bash), Read-Token-Limit-Block (PreToolUse/Read, eingeführt 18.05.2026)
- **PostToolUse feuert NICHT bei Tool-Errors** (18.05.2026 empirisch belegt). Bei Validierungs-Bedarf der Tool-Inputs/-Outputs IMMER PreToolUse, nie PostToolUse — PostToolUse läuft nur nach Erfolg.

## Permission Model
- **Allow-Liste**: Laura-Scripts + MCP-Server (auto-approved)
- **Deny-Liste**: Destruktive Commands (`git push`, `rm -rf`, `vercel --prod`) werden technisch geblockt
- **Auto Mode**: Nutzbar wenn Deny-Liste vollständig. Respektiert explizite User-Grenzen seit 2.1.90

## Anthropic-SDK Patterns (eingeführt 18.05.2026 aus Karpathy-Wiki-Spike)
- **Strukturierter Output: IMMER Tool-Use, nie freier JSON-Text-Parse.** Tools mit `input_schema` definieren, `tool_choice={"type":"tool","name":...}` forced, dann `block.input` aus dem `tool_use`-Block extrahieren. Free-form JSON-Parse von Anthropic-Text-Responses ist brüchig bei großen Markdown-/Code-Inhalten (unescapte Quotes/Newlines sprengen `json.loads()`).
- **Tool-Use kann char-by-char streamen** wenn Output > `max_tokens` — Schema-Violation. Symptom: `block.input["array_field"]` ist eine Liste von 1-Zeichen-Strings statt typed Array. Workaround: bei `isinstance(field, list) and all(isinstance(x, str) for x in field)` joinen + `json.loads()`. Besser: `max_tokens` hoch (32k+) oder Input-Volumen begrenzen.
- **System-Prompt-Templates mit literalen `{}` (JSON-Beispiele): NIE `.format()`** — KeyError beim ersten unbekannten Klammer-Schlüssel. Statt dessen eigene Platzhalter `<<X>>` via `.replace()`. Robuster, kein Escape-Stress.
- **Anthropic-SDK-Standard:** `anthropic>=0.69.0`. Default-Modell `claude-sonnet-4-6` für Komplex-Tasks, `claude-haiku-4-5-20251001` für Einfach-Tasks. Cost-Logging via `resp.usage.input_tokens` / `output_tokens`.

## MCP-Integration
- MCP Result Size Override: Bis 500K Zeichen via `_meta["anthropic/maxResultSizeChars"]` (Server-seitig)
- Tool-Routing: Einzelne MCP-Calls direkt, Batch (5+) via Sub-Agent
- **Paula-MCP** (`~/paula/apps/paula-mcp/`, gebaut 11.05.2026): 10 `paula_*`-Tools für Provinzial-Operationen via Stdio. Registrierung: `claude mcp add paula --scope user node /Users/floriansiepe/inge/apps/paula-mcp/dist/index.js`. ENV: `PAULA_API_BASE` (default `localhost:3001`), `PAULA_DEMO_MODE=on` blockt Schreib-Tools. Tools: kontakt_lookup, get_kunde, get/aktualisiere/lege_beratung, lege/aktualisiere_aufgabe, lege_kommunikation, get/aktualisiere_termin. Plan: `~/Laura/work/plan-paula-mcp.md`.
- **UUID-Validation bei Chat-getriggerten Tools (eingeführt 18.05.2026 nach Studio-Halluzination).** Wenn ein Tool als Input eine UUID aus dem Chat-Kontext annimmt (z.B. `entity_id` für `studio_propose`), muss die Tool-`execute()`-Funktion eine Regex-Validation VOR DB-Insert/API-Call machen. Grund: LLM neigt dazu Anzeige-Nummern („Workshop #2") oder Sequenz-Indizes („2") als UUID zu missinterpretieren. Plus Hinweis im Tool-Schema-Description: „MUSS UUID aus vorherigem find/list-Tool sein, NIEMALS aus Titel/Anzeige-Nummer ableiten." Auslöser: 18.05.2026 — `studio_propose` wurde mit `entity_id="2"` aus „Workshop #2" aufgerufen, hätte Postgres-UUID-Constraint-Crash erzeugt. Fix in `mcp-bridge.ts` + Prompt nachträglich, bei künftigen Tools von Anfang an.

## Architektur-Aufträge mit Buzzwords ("generalisieren", "modul", "studio")
- **Mental-Model VOR Datenmodell klären (eingeführt 18.05.2026 nach Studio-Pfad-B-Missverständnis).** Bei jedem Architektur-Auftrag mit abstrakten Wörtern wie „generalisieren", „X als Modul", „aus jeder Entity X" zuerst rückfragen WIE der Trigger/Einstieg aussehen soll, nicht nur WAS technisch generalisiert wird. Florian's „Pfad B Studio-Modul generalisieren" 18.05.2026 wurde technisch korrekt als polymorphes Datenmodell gebaut, aber UI-mäßig falsch als „StudioBar pro Entity-Page" interpretiert. Florian's tatsächliche Vision war Chat-Trigger mit Inline-Widget („widget öffnet sich im Chat wo ich auswählen kann"). Rückfrage-Pflicht in der Brainstorm-Phase: „Wo soll Florian den Trigger sehen — Button in der Detail-Page? Chat-Eingabe? Slash-Command? Listen-Aktion?" Erst Mental-Model-Bestätigung, dann Datenmodell + UI gemeinsam.

## Paula-API: Routen-Bau-Disziplin (eingeführt 11.05.2026)
- **Vor neuem Endpoint-Bau IMMER drei Stellen greppen**: `~/paula/apps/api/src/index.ts`, `~/paula/apps/api/src/routes/*`, **`~/paula/apps/api/src/lib/openapi-routes.ts`**. Letzteres registriert generische CRUD-Routes über `app.route('/', openapiRoutes)` als erstes — fängt deshalb gleichnamige Pfade vor allen anderen ab.
- Lesson aus Welten-Zusammenführung 11.05.: zweimal Route in `routes/aufgabe.ts` + `routes/termin.ts` gebaut die `openapi-routes.ts` schon hatte — beide wieder gelöscht, Schemas in OpenAPI erweitert. Sauberer Pfad: **bestehende Generic-Routes erweitern, nicht parallel bauen**.
- Audit-Spalten-Disziplin: beim Datenbestand-Audit nie auf eine Spalte (`titel`) beschränken — komplette Repräsentation prüfen (`name`, `beschreibung`, etc.). Heute drei „leere Stubs" fälschlich als Stubs eingestuft die in Wahrheit Inhalt hatten.

## PDF-Export
- **Standard: Typst** – `bash ~/Laura/scripts/typst-pdf.sh template.typ [output.pdf] [json_data]`
- Templates: `~/Laura/templates/typst/` (Rechnung, Vorsorgekonzept, weitere nach Bedarf)
- Deterministischer Seitenumbruch, Header/Footer nativ, 50-200ms, kein Browser
- **Legacy: HTML-to-PDF** – `bash ~/Laura/scripts/html-to-pdf.sh` für bestehende HTML-Templates
- Rechnungen: Typst-Template mit `{{EPC_QR}}`. HTML-Template + QR-Script als Fallback
- Neue Dokumente → immer Typst. Details: `memory/topics/html-print-css.md`

## Versions-Awareness
- Installierte Version: `claude -v` (CLI) + `code --list-extensions --show-versions | grep claude` (Extension)
- Update: `claude update` (CLI), Extension auto-updated via Marketplace
- Changelog + Workflow-Impact: `~/Laura/memory/topics/claude-code-updates.md`
- **Default Effort = High** seit 2.1.94 (steuerbar via `/effort`)
