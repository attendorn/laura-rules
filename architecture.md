# Claude Code Architecture Patterns

> Kurzreferenz für Lauras Architekturentscheidungen. Bei Detailfragen: `~/Laura/memory/topics/claude-code-updates.md`
> Bau-Regeln (Hooks, Worktrees, SDK/MCP, Paula-API/-Deploy, Git) stehen seit AP-0336 in `rules/coding.md` — im Projektbaum beim ersten Code-Read eingeblendet, außerhalb über die Lade-Liste.

## Agent-Definition
- Model-Wahl: **Haiku** = Datensammlung, einfache MCP-Calls. **Sonnet** = Analyse, Synthese, Schreibarbeit. **Opus** = nur Hauptkontext (Laura selbst)
- **Ausnahme Richter-/Reviewer-Knoten (Verifikation, Gegenprüfung, `/gegenlesen`):** geordnete Fallback-Leiter `opus` (Default; der lokale Qwen ist Vorprüfer, nie Richter — Schattenbetrieb läuft als Dauerregel) → `sonnet` (Fallback, False-BLOCKER-Neigung); Fable nur auf expliziten Florian-Zuruf, **niemals Haiku** — schwache Richter erzeugen False-Positive-Lawinen und übersehen echte Fehler. Die Zeile „Opus = nur Hauptkontext" gilt für Standard-Arbeits-Agenten, NICHT für Richter-Knoten. Modell beim Spawn immer explizit setzen, nie erben lassen. (Herkunft → guardrails-historie.md [H36])

## Sub-Agent-Patterns
- **Hintergrund** (`run_in_background: true`): Für unabhängige Tasks (Downloads, Sync, Update-Check)
- **Vordergrund**: Wenn Ergebnis Gate für nächsten Schritt ist (Kalender-Scout → Briefing)
- **Proaktive Delegation**: >8k erwartete Token im Hauptkontext → automatisch an Sub-Agent
- **Sub-Sub-Agents erlaubt bis Tiefe 3** (Claude Code erlaubt Nesting-Tiefe 3 seit 2.1.220). Default bleibt flach: Orchestrierung im Hauptkontext mit Tiefe-1-Sub-Agenten. Tiefe 2-3 nur wenn ein Sub-Agent echten Orchestrierungs-Bedarf hat (z.B. Verifier-Orchestrator spawnt Winkel-Reviewer) — nie aus Bequemlichkeit (Herkunft → guardrails-historie.md [H39])

## Agent Teams
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

## Permission Model
- **Allow-Liste**: Laura-Scripts + MCP-Server (auto-approved)
- **Deny-Liste**: Destruktive Commands (`git push`, `rm -rf`, `vercel --prod`) werden technisch geblockt
- **Auto Mode**: Nutzbar wenn Deny-Liste vollständig. Respektiert explizite User-Grenzen seit 2.1.90

## MCP-Integration
- Tool-Routing: Einzelne MCP-Calls direkt, Batch (5+) via Sub-Agent

## Modell oder Code?

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

### Schattenlauf bei teurer Arbeit

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
mechanisch gegenlesen): `work/video-abgleich-2026-08-31/LIESMICH.md`.
Laufende Messstrecke: AP-0286.

## Fremde Lösungen: Baustein, Muster oder Messlatte?

Bevor eine fremde Lösung (Bibliothek, Framework, Plugin, fremder Workflow) bewertet wird, ihre
**Rolle** benennen — sie ist genau eine der drei:

| Rolle | Bedeutung | Frage |
|---|---|---|
| **Baustein** | Der Code selbst wird übernommen und läuft bei uns | Wollen wir von deren Versions-Uhr abhängen? |
| **Muster** | Nur die Idee wird übernommen, gebaut wird selbst | Was genau ist die Idee, losgelöst vom Werkzeug? |
| **Messlatte** | Weder Code noch Idee — sie zeigt, was gut genug wäre | Woran erkennen wir, dass wir gleichauf sind? |

**Fremde Lösungen lösen fremde Probleme.** Ohne diese Trennung wird eine Sache als Ganzes
angenommen oder als Ganzes verworfen — beides meist falsch. (Herkunft → guardrails-historie.md [H38])

Die Rolle gehört in die Prior-Art-Prüfung des Arbeitspakets, nicht in den Kopf.

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
