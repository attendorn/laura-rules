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
- **Aktive Hooks**: Skill-Usage-Log (PostToolUse/Skill), Test-Coverage-Check + Gedankenstrich-Warnung (PostToolUse/Write|Edit), Destruktive-Commands-Block (PreToolUse/Bash)

## Permission Model
- **Allow-Liste**: Laura-Scripts + MCP-Server (auto-approved)
- **Deny-Liste**: Destruktive Commands (`git push`, `rm -rf`, `vercel --prod`) werden technisch geblockt
- **Auto Mode**: Nutzbar wenn Deny-Liste vollständig. Respektiert explizite User-Grenzen seit 2.1.90

## MCP-Integration
- MCP Result Size Override: Bis 500K Zeichen via `_meta["anthropic/maxResultSizeChars"]` (Server-seitig)
- Tool-Routing: Einzelne MCP-Calls direkt, Batch (5+) via Sub-Agent
- **Paula-MCP** (`~/paula/apps/paula-mcp/`, gebaut 11.05.2026): 10 `paula_*`-Tools für Provinzial-Operationen via Stdio. Registrierung: `claude mcp add paula --scope user node /Users/floriansiepe/inge/apps/paula-mcp/dist/index.js`. ENV: `PAULA_API_BASE` (default `localhost:3001`), `PAULA_DEMO_MODE=on` blockt Schreib-Tools. Tools: kontakt_lookup, get_kunde, get/aktualisiere/lege_beratung, lege/aktualisiere_aufgabe, lege_kommunikation, get/aktualisiere_termin. Plan: `~/Laura/work/plan-paula-mcp.md`.

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
