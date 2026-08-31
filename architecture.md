# Claude Code Architecture Patterns

> Kurzreferenz für Lauras Architekturentscheidungen. Bei Detailfragen: `~/Laura/memory/topics/claude-code-updates.md`

## Agent-Definition
- Rollen liegen in `~/Laura/agents/roles/` mit **YAML-Frontmatter** (name, description, model, tools, memory)
- **Named Agents** nutzen: `name: "kalender-scout"` beim Agent-Aufruf → per `SendMessage(to: "name")` in der Session wiederverwenden statt neu spawnen
- Model-Wahl: **Haiku** = Datensammlung, einfache MCP-Calls. **Sonnet** = Analyse, Synthese, Schreibarbeit. **Opus** = nur Hauptkontext (Laura selbst)
- **Ausnahme Richter-/Reviewer-Knoten (Verifikation, Gegenprüfung, `/gegenlesen` — AP-0098, 30.07.2026):** geordnete Fallback-Leiter `opus` (Default per Arena-Urteil 30.07.) → `sonnet` (Fallback; False-BLOCKER-Neigung, siehe Matrix in `~/Laura/work/gegenlesen-modell-arena-2026-07-30.md`); Fable nur auf expliziten Florian-Zuruf, **niemals Haiku** — schwache Richter erzeugen False-Positive-Lawinen und übersehen echte Fehler. Die Zeile „Opus = nur Hauptkontext" gilt für Standard-Arbeits-Agenten, NICHT für Richter-Knoten. Modell beim Spawn immer explizit setzen, nie erben lassen.

## Sub-Agent-Patterns
- **Hintergrund** (`run_in_background: true`): Für unabhängige Tasks (Downloads, Sync, Update-Check)
- **Vordergrund**: Wenn Ergebnis Gate für nächsten Schritt ist (Kalender-Scout → Briefing)
- **Proaktive Delegation**: >8k erwartete Token im Hauptkontext → automatisch an Sub-Agent
- **MCP-Vererbung**: Sub-Agents erben automatisch MCP Tools von dynamisch registrierten Servern (ab 2.1.101)
- **Sub-Sub-Agents erlaubt bis Tiefe 3** (gelockert 30.07.2026, Florian-Freigabe; Claude Code erlaubt Nesting-Tiefe 3 seit 2.1.220). Default bleibt flach: Orchestrierung im Hauptkontext mit Tiefe-1-Sub-Agenten. Tiefe 2-3 nur wenn ein Sub-Agent echten Orchestrierungs-Bedarf hat (z.B. Verifier-Orchestrator spawnt Winkel-Reviewer) — nie aus Bequemlichkeit
- **Worktree-Isolation gehärtet seit 2.1.222**: `isolation: "worktree"` schützt jetzt Datei-Edits UND Bash in jedem Session-Typ gegen destruktive Git-Commands gegen das Haupt-Checkout (vorher Lücke). `/fork` erzeugt seit 2.1.221 ebenfalls ein eigenes Worktree statt im Ursprungs-Checkout zu arbeiten.

## Session-Worktrees: was mitwandert und was nicht (26.08.2026, AP-0147 + AP-0251)

Mehrere Bau-Sitzungen parallel sind der Normalfall; je Sitzung ein eigener Worktree. Vier Regeln,
alle gemessen, nicht angenommen (Messprotokolle → guardrails-historie.md [H27]):

- **Code folgt der Sitzung, geteilter Zustand bleibt am Hauptbaum.** Hooks lösen über
  `code/hooks/dispatch.sh` den Baum der aufrufenden Sitzung auf (`LAURA_WURZEL`) und bewachen die
  Vereinigung aus Sitzungs- und Hauptbaum. Am Hauptbaum bleiben `.active-tasks.json`, Not-Aus-Flags,
  Deploy-Freigaben, Browser-Profil, Alarmdateien, Lauflogs, Sperren. Skripte, die beides anfassen,
  brauchen zwei Variablen, nicht eine (`testsuite-lauf.sh` testete sonst aus jedem Baum den Hauptbaum).
- **Ein Weg zum Baum:** `worktree.sh neu <slug>` → `~/Laura-worktrees/<slug>`, Zweig `wt/<slug>`,
  rüstet `npm ci` (outlook-resolver) und `.claude/settings.local.json` nach (beides gitignored).
  Wechsel mit `EnterWorktree` und Parameter **`path`** (nicht `name`). Zurück mit
  `git push origin HEAD:main` aus dem Baum, nie `merge --ff-only` im Hauptbaum, nie `--force`.
- **Hooks folgen der Sitzung, Slash-Commands nicht.** `~/.claude/commands` ist ein Symlink auf
  `code/commands` im Hauptbaum — ein geänderter Command wird **im Hauptbaum oder gar nicht** getestet.
- **Nebenläufigkeit auf geteilten Dateien:** atomarer Ersatz genügt nicht, der Lese-Ändere-Schreibe-
  Zyklus braucht eine Sperre — Hausmuster `mkdir` + Verfallsfrist (`active-task.sh`,
  `log-skill-usage.sh`) oder `O_CREAT|O_EXCL` (`pakete-nummer.sh`).

Weitere Grenzen: Hook-Registrierung in `settings.json` ist global; `bin/python3` und `notion-rag`
sind absolute Symlinks (venv/RAG geteilt); ein frischer Baum hat nichts Gitignoriertes; 19 launchd-Jobs
adressieren den Hauptbaum; das Push-Rennen auf `main` bleibt.

## Agent Teams (aktiviert 11.04.2026)
- **Experimentell**, aktiviert via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json
- **Unterschied zu Sub-Agents**: Teammates kommunizieren UNTEREINANDER (Mailbox), nicht nur zurück zum Parent
- **Architektur**: Team Lead (Laura) + Teammates (eigene Claude-Instanzen) + Shared Task List + Mailbox
- **Tools**: `TeamCreate`, `TeamDelete`, `SendMessage` (zwischen Teammates). Seit 2.1.222 läuft jede `SendMessage` an eine andere Agent-Session zusätzlich durch den Auto-Mode-Permission-Classifier vor Dispatch.
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
- **PreToolUse-Auto-Allow-Bypass in Background-Agent-Nebenoperationen gefixt (2.1.222)**: Summaries, Compaction und Renames von Hintergrund-Agents umgingen bisher Tool-Restrictions aus Auto-Allow-Hooks. Lauras Hooks (`check-overcompletion.sh`, `block-provinzial-write.sh`) greifen jetzt auch dort zuverlässig.

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
- **UUID-Validation bei Chat-getriggerten Tools (eingeführt 18.05.2026 nach Studio-Halluzination).** Wenn ein Tool als Input eine UUID aus dem Chat-Kontext annimmt (z.B. `entity_id` für `studio_propose`), muss die Tool-`execute()`-Funktion eine Regex-Validation VOR DB-Insert/API-Call machen. Grund: LLM neigt dazu Anzeige-Nummern („Workshop #2") oder Sequenz-Indizes („2") als UUID zu missinterpretieren. Plus Hinweis im Tool-Schema-Description: „MUSS UUID aus vorherigem find/list-Tool sein, NIEMALS aus Titel/Anzeige-Nummer ableiten." (Herkunft → guardrails-historie.md [H17])

## Architektur-Aufträge mit Buzzwords ("generalisieren", "modul", "studio")
- **Mental-Model VOR Datenmodell klären (eingeführt 18.05.2026 nach Studio-Pfad-B-Missverständnis).** Bei jedem Architektur-Auftrag mit abstrakten Wörtern wie „generalisieren", „X als Modul", „aus jeder Entity X" zuerst rückfragen WIE der Trigger/Einstieg aussehen soll, nicht nur WAS technisch generalisiert wird. Rückfrage-Pflicht in der Brainstorm-Phase: „Wo soll Florian den Trigger sehen — Button in der Detail-Page? Chat-Eingabe? Slash-Command? Listen-Aktion?" Erst Mental-Model-Bestätigung, dann Datenmodell + UI gemeinsam. (Herkunft → guardrails-historie.md [H18])

## Modell oder Code? (eingeführt 31.08.2026)

**KI nur dort, wo Verstehen nötig ist — alles andere ist deterministischer Code.** Ein Modellaufruf
ist zu rechtfertigen, nicht zu unterstellen: Er kostet Geld, ist nicht reproduzierbar und kann
plausibel danebenliegen, ohne es zu melden. Code ist schneller, billiger, wiederholbar und lässt
sich testen. Die Prüffrage vor jedem Aufruf: *Muss hier etwas verstanden, abgewogen oder formuliert
werden — oder wird nur verglichen, gezählt, umgeformt, gesucht?* Nur der erste Fall ist Modellarbeit.

Belegender Anlass (Video-Abgleich 31.08.): Techniken aus einem Transkript zu extrahieren ist
Verstehen — dafür lief ein Modell. Zu prüfen, ob ein Beleg-Zitat wörtlich in der Quelle steht, ist
Vergleichen — dafür lief ein zehnzeiliges Python-Skript. **Das Skript fand einen Zitierfehler des
Modells, den kein zweiter Modellaufruf zuverlässig gefunden hätte.** Umgekehrt gilt die Regel auch:
Wo ein deterministischer Prüfer gebaut wird, muss er selbst geprüft werden — der erste Anlauf jenes
Skripts löschte Zeilenumbrüche statt sie zu ersetzen und hätte fast einen falschen Befund erzeugt.

### Schattenlauf bei teurer Arbeit (Florian-Vorgabe 31.08.2026)

**Immer wenn etwas Token-Intensives startet — oder wenn im Nachhinein auffällt, dass es teuer war —
läuft der lokale Qwen (MLX, Port 8081) zusätzlich über dieselbe Aufgabe.** Nicht als Ersatz: Sein
Ergebnis wird protokolliert und mit dem echten verglichen, nie in die Auslieferung gemischt.

Der Grund ist nicht Sparen, sondern **Erfahrung sammeln, wo lokale Arbeit trägt und wo nicht** — und
das kostet nichts, weil das Modell auf Florians Rechner läuft. Ohne solche Schattenläufe bleibt die
Modellwahl eine Meinung; mit ihnen wird sie nach ein paar Fällen eine belegte Entscheidung.

Auslöser (jeder für sich genügt): ein Lauf über 100k Token · eine Aufgabe, die dieselbe Struktur aus
langem Text zieht · mehrere parallele Sub-Agenten auf demselben Material · ein `/gegenlesen`-Lauf ·
jede Auswertung, die sich wiederholt. Wo der Vergleich nicht geht, gehört das benannt statt
übergangen — ein lokales Modell ohne Netz kann keine Primärquellen nachschlagen, und das ist eine
fehlende Fähigkeit, kein schlechtes Ergebnis.

Betriebswissen (Denkmodus abschalten, zwei getrennte Aufträge, Codezäune strippen, Beleg-Prüfung
mechanisch gegenlesen): `work/video-abgleich-2026-08-31/LIESMICH.md`, Memory-Lesson `d83d3111`.
Laufende Messstrecke: AP-0286.

## Fremde Lösungen: Baustein, Muster oder Messlatte? (eingeführt 31.08.2026)

Bevor eine fremde Lösung (Bibliothek, Framework, Plugin, fremder Workflow) bewertet wird, ihre
**Rolle** benennen — sie ist genau eine der drei:

| Rolle | Bedeutung | Frage |
|---|---|---|
| **Baustein** | Der Code selbst wird übernommen und läuft bei uns | Wollen wir von deren Versions-Uhr abhängen? |
| **Muster** | Nur die Idee wird übernommen, gebaut wird selbst | Was genau ist die Idee, losgelöst vom Werkzeug? |
| **Messlatte** | Weder Code noch Idee — sie zeigt, was gut genug wäre | Woran erkennen wir, dass wir gleichauf sind? |

**Fremde Lösungen lösen fremde Probleme.** Ohne diese Trennung wird eine Sache als Ganzes
angenommen oder als Ganzes verworfen — beides meist falsch. Anlass: Der Rat verwarf am 20.08.2026
BMAD einstimmig als **Baustein** (Vertrauensinstanz an fremder Versions-Uhr, Memory `ea074be5`) —
richtig, aber einzelne BMAD-Bausteine wären als **Muster** brauchbar gewesen, und der Befund über
seine löchrige Durchsetzung war eine **Messlatte**, die uns den eigenen blinden Fleck zeigte.
Gleiches Bild am 31.08. bei LangGraph: Konzept ja (Muster), Bibliothek nein (Baustein).

Die Rolle gehört in die Prior-Art-Prüfung des Arbeitspakets, nicht in den Kopf.

## Paula-API: Routen-Bau-Disziplin (eingeführt 11.05.2026)
- **Vor neuem Endpoint-Bau IMMER drei Stellen greppen**: `~/paula/apps/api/src/index.ts`, `~/paula/apps/api/src/routes/*`, **`~/paula/apps/api/src/lib/openapi-routes.ts`**. Letzteres registriert generische CRUD-Routes über `app.route('/', openapiRoutes)` als erstes — fängt deshalb gleichnamige Pfade vor allen anderen ab.
- Sauberer Pfad: **bestehende Generic-Routes erweitern, nicht parallel bauen**.
- Audit-Spalten-Disziplin: beim Datenbestand-Audit nie auf eine Spalte (`titel`) beschränken — komplette Repräsentation prüfen (`name`, `beschreibung`, etc.). (Welten-Zusammenführung 11.05.; Herkunft → guardrails-historie.md [H19])
- **Status-/Enum-Wert-Disziplin (eingeführt 22.05.2026).** Bevor ein neuer Wert für eine Status-/Enum-Spalte in Paula (oder jeder Postgres-Tabelle) im Code geschrieben wird, IMMER die echten DB-`CHECK`-Constraints prüfen — nicht nur die Drizzle-Schema-Datei. Drizzle-`text()`-Spalten mit Freitext-Kommentar (`// offen|beantwortet|…`) verbergen, dass die DB eine `CHECK`-Constraint trägt, die den neuen Wert ablehnt. Prüf-Kommando: `SELECT pg_get_constraintdef(oid) FROM pg_constraint WHERE conrelid='<tabelle>'::regclass AND contype='c';`. Wenn eine Constraint existiert → Migration die sie per `DROP CONSTRAINT IF EXISTS` + `ADD CONSTRAINT` mit dem erweiterten `ARRAY` neu setzt. Memory-Lesson `750e2bb4`. (Herkunft → guardrails-historie.md [H20])

## PDF-Export
- **Standard: Typst** – `bash ~/Laura/scripts/typst-pdf.sh template.typ [output.pdf] [json_data]`
- Templates: `~/Laura/templates/typst/` (Rechnung, Vorsorgekonzept, weitere nach Bedarf)
- Deterministischer Seitenumbruch, Header/Footer nativ, 50-200ms, kein Browser
- **Legacy: HTML-to-PDF** – `bash ~/Laura/scripts/html-to-pdf.sh` für bestehende HTML-Templates
- Rechnungen: Typst-Template mit `{{EPC_QR}}`. HTML-Template + QR-Script als Fallback
- Neue Dokumente → immer Typst. Details: `memory/topics/html-print-css.md`

## Versions-Awareness
- Changelog + Workflow-Impact: `~/Laura/memory/topics/claude-code-updates.md` (dort auch Prüf-/Update-Befehle)
- **Default Effort = High** seit 2.1.94 (steuerbar via `/effort`)
