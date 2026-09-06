---
paths:
  - "code/**"
  - "scripts/**"
  - "hooks/**"
  - "tests/**"
  - "skills/**"
  - "**/*.py"
  - "**/*.sh"
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.mjs"
  - "**/*.bats"
  - "**/*.sql"
  - "**/*.swift"
---

# Coding-Regeln – Bauen, Testen, Committen, Deployen

> Rollen-Datei für Bau-Arbeit (AP-0336, 04.09.2026). Umgezogen aus `guardrails.md`, `architecture.md`,
> `persona-global.md` und `ANLEITUNG-kern.md`/`ANLEITUNG.md`, Wortlaut unverändert. **Zustellung:** im
> Projektbaum `~/Laura` blendet Claude Code diese Datei beim ersten Lesen einer Code-Datei ein (`paths:`
> oben, Globs aus den Adressen der offenen Pakete); für Code außerhalb (`~/paula`, `~/admin-gettheflo`,
> `~/laura-huelle`, …) lädt `laura-work.md` Schritt 3b-Paket sie als erste Adresse der Lade-Liste.
> Dieselbe Datei ist die Rollen-Datei des Coding-Agenten im Fabrik-Strang (AP-0324).
> `[Hn]`-Marker = Anlass, Datum und Memory-Lesson stehen in `memory/topics/guardrails-historie.md`.

## Git, Commit und Push

**VERBOTEN (ohne explizite Freigabe):**
- **Git push — außer auf der Push-Allowlist.** Auto-Push ohne Rückfrage ist erlaubt für `laura-pa`, `laura-rules` und `admin-gettheflo`: dort sitzt kein weiterer Mitarbeiter drin, bei `admin-gettheflo` ist der Vercel-Live-Deploy bewusst in Kauf genommen — deshalb dort **nach** dem Push Smoke-Test der betroffenen Route. Alles andere (Paula/`inge`, unbekannte Repos) und **jeder** Force-Push behalten den Freigabe-Dialog. Umgesetzt in `hooks/git-push-guard.sh` über eine Remote-URL-Allowlist. **Der pre-push-Test-Hook ist von dieser Freigabe ausdrücklich NICHT erfasst:** rote Tests blocken den Push weiterhin, `--no-verify` ist verboten, der Fehlschlag wird gefixt. (Herkunft → guardrails-historie.md [H34])
- **Pauschales Git-Staging.** `git add .`, `git add -A`, `git add <verzeichnis>` und `git commit -a` sind verboten. Gestaged wird **immer** eine explizite Pfadliste der Dateien, die zur eigenen Arbeit gehören. Vor jedem Commit `git status --short` lesen und fremde Pfade bewusst auslassen. **Die Pfadliste gehört auch an den Commit selbst (`git commit -m … -- <pfade>`), nicht nur ans add:** ein Commit ohne Pfadliste nimmt die gesamte Staging-Area mit, inklusive dessen, was eine Parallel-Session dort vorgestagt hat. `active-task.sh` ist ein Awareness-Signal, kein Schutz — es sagt wer woran arbeitet, verhindert aber kein Staging fremder Pfade. Wird eine Vereinnahmung erst nach dem Commit bemerkt: **kein Rebase, kein Reset, kein force-push.** Auf einem Baum mit aktiver Fremd-Session ist das destruktiv; der Vorfall wird benannt und Florian entscheidet. Technisch durchgesetzt durch `hooks/git-commit-guard.sh` (PreToolUse/Bash): Commit ohne Pfadliste, `-a`/`--all`, Amend ohne Pfadliste, pauschale Pfadliste und `git add -A/./-u` werden hart geblockt, jeder Treffer landet in `work/.commit-guard.log`. Notausgang `LAURA_COMMIT_GUARD_OVERRIDE=1` protokolliert sich selbst. **Der Hook prüft Form, nicht Eigentum:** Bearbeiten zwei Sessions dieselbe Datei, nimmt auch eine korrekte Pfadliste die fremden Zeilen mit — dagegen hilft nur Arbeitsteilung nach Dateien. [H22]

**Policies:**
- **Kein `git reset --hard` auf Florians Workspace ohne Vorab-Approval.** Bei Eingriff in einen Git-Repo unter `~/Laura/` (oder anderem Florian-Workspace) ist `git reset --hard` destruktiv für lokale Working-Tree-Edits. Auch wenn Diff-Check vorher zeigt dass alles identisch ist: Florian-OK vorher einholen. Backup-Verzeichnis allein reicht nicht — der Mac-Workspace ist nicht meine Sandbox. [H6]

**Automatische Durchsetzung** (settings.json / `.git/hooks`):
- `hooks/git-push-guard.sh` (PreToolUse/Bash): Freigabe-Dialog vor `git push`, außer für die Allowlist-Repos oben; Force-Push immer Dialog
- `.git/hooks/pre-push`: Unit-Tests laufen vor jedem Push, rot blockt — unabhängig von der Allowlist. Seit AP-0344 (06.09.2026) laufen daneben parallel Basis-Lint (Syntax + Pyflakes) und Strict-Burndown, jeder rote Teil blockt
- `.git/hooks/pre-commit` Schritt 5: Strict-Lint auf neuen Zeilen gestagter Python-Dateien (`lint-clean.py --staged`, Index-Blob, nur wenn Pfade im Strict-Scope liegen); Regeln, Schwellen und Reparaturwege in `code/lint/LINT.md`. Ein Regress wird repariert, nie in die Baseline geschrieben (Decision `376575b7`)

## Tests

### Superpowers-Skills (proaktiv nutzen)

| Situation | Superpowers-Skill | Lauras Regel |
|---|---|---|
| Vor "fertig"-Meldung | `verification-before-completion` | Drei Stufen, aufsteigend: der Reflex („keine Erfolgsmeldung ohne frische Evidenz“), die Selbstkritik-Checkliste in `policies.md`, und `/gegenlesen` als Zweitmeinung mit frischem Kontext |
| Bei Fehlern/Bugs | `systematic-debugging` | Root-Cause-Analyse |
| 2+ unabhängige Tasks | `dispatching-parallel-agents` | Sub-Agent-Nutzung |
| Multi-Step-Plan | `writing-plans` | Plan-First |
| Nach Code-Skills | – | `/test` vorschlagen (s.u.) |

### Test-Policy (seit 07.04.2026)

**Alles was in Produktion geht oder fester Bestandteil der Architektur wird, bekommt Tests.** Keine Ausnahme. Nicht für Prototypen oder Einmal-Scripts.

**Nach Code-erzeugenden Skills** (`/website`, Bau aus einem Arbeitspaket) prüft Laura automatisch:
1. Tests vorhanden? → `/test scripts` oder `/test [script]` vorschlagen
2. Keine Tests? → `/test write [script]` vorschlagen
3. Deploy? → `/test deploy [URL]` vorschlagen

**Test-Infrastruktur:** `~/Laura/code/tests/` (Unit/Integration/Smoke), Runner: `run_tests.py`, Audit-Trail: Supabase `test_results`.
**Hooks:** PostToolUse bei Script-Änderung (Hinweis), Pre-Push (Unit-Tests blockend).

**Policies:**
- **Vollständigkeits-Check vor „X ist gefixt".** Bevor ein Code-Fix als erledigt kommuniziert wird, MUSS das Symptom/Pattern repo-weit gegrept werden. Dann gilt: ALLE Vorkommen fixen — oder explizit benennen welche bewusst offen bleiben. Verboten: einen Fix an einer Stelle machen und „erledigt" sagen, ohne geprüft zu haben an wie vielen Stellen dasselbe Pattern existiert. Wurzel-Bias `sample_size_blind`. Diese Regel deckt die Lücke der bestehenden sample_size-Regeln (die nur DB-Lookups + Klassifikations-Tags adressieren). [H15]
- **Migrations-Vollständigkeit vor „umgestellt".** Eine Umstellung von einem alten auf einen neuen Weg (Provider, Endpoint, Tabelle/Spalte, RPC, Script-Pfad, Bibliothek) gilt erst dann als abgeschlossen, wenn **alle Aufrufstellen des alten Wegs repo-weit gegriffen und die Liste dokumentiert** ist — inklusive der Gegenrichtung (wer ruft die alte Funktion/Spalte noch?) und der automatisierten Starter (launchd-Plists, crontab, package.json-Scripts, CI). Der Grep gehört **vor** die Umstellung, nicht danach, und seine Trefferliste in den Beleg. **Sonderfall:** Ein Grep nach der falschen Sache zählt nicht — bei einem Fehlerbild „eine Stelle übersehen" ist die Suche nach Konstanten oder Dimensionszahlen wertlos, gesucht werden muss nach **Aufrufern**. Abgrenzung zu [H15]: dort geht es um Vollständigkeit beim **Fix eines Fehlermusters**, hier um Vollständigkeit beim **Wechsel eines Wegs**. Vorerst Regel, kein Hook — „Ursache vor Hook" [H16] gilt, Prüfpunkt ist die nächste Migration. (Herkunft → guardrails-historie.md [H33])
- **Root-Cause-Analyse bei Errors.** Nicht nur fixen, sondern prüfen ob architektonisch etwas falsch läuft.

## Fehlervermeidung (→ memory/topics/fehler.md)

- MCP installieren/updaten → vorher `fehler.md` lesen
- TCC-/Berechtigungsfehler → `fehler.md` durchsuchen
- Technischer Fehler → `fehler.md` durchsuchen

## Hooks (settings.json)
- **Conditional Hooks**: `matcher`-Feld für Tool-spezifische Hooks (z.B. nur bei "Write|Edit")
- **PreToolUse**: Validierung/Blocking vor Ausführung. **Bei Exit 0 erreicht weder stdout noch stderr irgendjemanden** (nur Debug-Log) — sichtbar ist dann ausschließlich JSON auf stdout: `hookSpecificOutput.additionalContext` erreicht das Modell, `systemMessage` den Nutzer. Bei Exit 2 (Block) ist stderr sichtbar. Ein Hook, der bei Exit 0 nur Text ausgibt, warnt niemanden. Testrahmen müssen stdout/stderr TRENNEN, sonst ist der Kanalfehler unsichtbar. (Herkunft → guardrails-historie.md [H37])
- **PostToolUse**: Logging, Nachverarbeitung. `$CLAUDE_TOOL_INPUT` als JSON im Script lesen (nicht Shell-Expansion im command)
- **SessionStart**: Terminal-Titel, Context Injection via stdout
- **Plugin-Hooks** (ab 2.1.94): Hooks im Skill-Frontmatter funktionieren jetzt zuverlässig
- **Aktive Hooks**: Skill-Usage-Log (PostToolUse/Skill), Test-Coverage-Check + Gedankenstrich-Warnung (PostToolUse/Write|Edit), Destruktive-Commands-Block (PreToolUse/Bash), Read-Token-Limit-Block (PreToolUse/Read)
- **PostToolUse feuert NICHT bei Tool-Errors.** Bei Validierungs-Bedarf der Tool-Inputs/-Outputs IMMER PreToolUse, nie PostToolUse — PostToolUse läuft nur nach Erfolg.
- **Heredoc-Wächter** (`hooks/check-python-heredoc.sh`, PreToolUse/Bash, AP-0347): parst jedes Python-Here-Doc (`python3 - <<'PY'`, auch `<<-` wie von der Shell dedentiert) vor der Ausführung. Ein ASCII-`"`, das ein zuvor geöffnetes „ in einem `"…"`-String schließt, wird zu `\"` escaped und der Befehl per `updatedInput` umgeschrieben (Inhalt bleibt byte-identisch, Meldung als additionalContext + systemMessage). Jeder andere Syntaxfehler blockt den ganzen Befehl, damit kein Commit hinter einem kaputten Skript herläuft. `python3 -c` wird nur geprüft und gemeldet. Unquotierter Delimiter mit `$`/Backtick im Rumpf: Hinweis statt Block. Protokoll `work/.heredoc-guard.log`. Der Wächter ersetzt nicht die Regel „Bau im Auto-Modus" (Lesson e0f465b6), er fängt die Gewohnheit, der Modus senkt die Gelegenheit.
- **PreToolUse-Auto-Allow-Bypass in Background-Agent-Nebenoperationen gefixt (2.1.222)**: Summaries, Compaction und Renames von Hintergrund-Agents umgingen bisher Tool-Restrictions aus Auto-Allow-Hooks. Lauras Hooks (`check-overcompletion.sh`, `block-provinzial-write.sh`) greifen jetzt auch dort zuverlässig.

## Session-Worktrees: was mitwandert und was nicht

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

## Agenten, SDK und MCP

**Agent-Definition:**
- Rollen liegen in `~/Laura/code/agents/roles/` mit **YAML-Frontmatter** (name, description, model, tools, memory)
- **Named Agents** nutzen: `name: "kalender-scout"` beim Agent-Aufruf → per `SendMessage(to: "name")` in der Session wiederverwenden statt neu spawnen

**Sub-Agent-Patterns:**
- **MCP-Vererbung**: Sub-Agents erben automatisch MCP Tools von dynamisch registrierten Servern (ab 2.1.101)
- **Worktree-Isolation gehärtet seit 2.1.222**: `isolation: "worktree"` schützt jetzt Datei-Edits UND Bash in jedem Session-Typ gegen destruktive Git-Commands gegen das Haupt-Checkout (vorher Lücke). `/fork` erzeugt seit 2.1.221 ebenfalls ein eigenes Worktree statt im Ursprungs-Checkout zu arbeiten.

### Anthropic-SDK Patterns
- **Strukturierter Output: IMMER Tool-Use, nie freier JSON-Text-Parse.** Tools mit `input_schema` definieren, `tool_choice={"type":"tool","name":...}` forced, dann `block.input` aus dem `tool_use`-Block extrahieren. Free-form JSON-Parse von Anthropic-Text-Responses ist brüchig bei großen Markdown-/Code-Inhalten (unescapte Quotes/Newlines sprengen `json.loads()`).
- **Tool-Use kann char-by-char streamen** wenn Output > `max_tokens` — Schema-Violation. Symptom: `block.input["array_field"]` ist eine Liste von 1-Zeichen-Strings statt typed Array. Workaround: bei `isinstance(field, list) and all(isinstance(x, str) for x in field)` joinen + `json.loads()`. Besser: `max_tokens` hoch (32k+) oder Input-Volumen begrenzen.
- **System-Prompt-Templates mit literalen `{}` (JSON-Beispiele): NIE `.format()`** — KeyError beim ersten unbekannten Klammer-Schlüssel. Statt dessen eigene Platzhalter `<<X>>` via `.replace()`. Robuster, kein Escape-Stress.
- **Anthropic-SDK-Standard:** `anthropic>=0.69.0`. Default-Modell `claude-sonnet-4-6` für Komplex-Tasks, `claude-haiku-4-5-20251001` für Einfach-Tasks. Cost-Logging via `resp.usage.input_tokens` / `output_tokens`.

### MCP-Integration
- MCP Result Size Override: Bis 500K Zeichen via `_meta["anthropic/maxResultSizeChars"]` (Server-seitig)
- **Paula-MCP** (`~/paula/apps/paula-mcp/`): 10 `paula_*`-Tools für Provinzial-Operationen via Stdio. Registrierung: `claude mcp add paula --scope user node /Users/floriansiepe/inge/apps/paula-mcp/dist/index.js`. ENV: `PAULA_API_BASE` (default `localhost:3001`), `PAULA_DEMO_MODE=on` blockt Schreib-Tools. Tools: kontakt_lookup, get_kunde, get/aktualisiere/lege_beratung, lege/aktualisiere_aufgabe, lege_kommunikation, get/aktualisiere_termin. Plan: `~/Laura/work/plan-paula-mcp.md`.
- **UUID-Validation bei Chat-getriggerten Tools.** Wenn ein Tool als Input eine UUID aus dem Chat-Kontext annimmt (z.B. `entity_id` für `studio_propose`), muss die Tool-`execute()`-Funktion eine Regex-Validation VOR DB-Insert/API-Call machen. Grund: LLM neigt dazu Anzeige-Nummern („Workshop #2") oder Sequenz-Indizes („2") als UUID zu missinterpretieren. Plus Hinweis im Tool-Schema-Description: „MUSS UUID aus vorherigem find/list-Tool sein, NIEMALS aus Titel/Anzeige-Nummer ableiten." (Herkunft → guardrails-historie.md [H17])

## Paula-API: Routen-Bau-Disziplin
- **Vor neuem Endpoint-Bau IMMER drei Stellen greppen**: `~/paula/apps/api/src/index.ts`, `~/paula/apps/api/src/routes/*`, **`~/paula/apps/api/src/lib/openapi-routes.ts`**. Letzteres registriert generische CRUD-Routes über `app.route('/', openapiRoutes)` als erstes — fängt deshalb gleichnamige Pfade vor allen anderen ab.
- Sauberer Pfad: **bestehende Generic-Routes erweitern, nicht parallel bauen**.
- Audit-Spalten-Disziplin: beim Datenbestand-Audit nie auf eine Spalte (`titel`) beschränken — komplette Repräsentation prüfen (`name`, `beschreibung`, etc.). (Welten-Zusammenführung 11.05.; Herkunft → guardrails-historie.md [H19])
- **Status-/Enum-Wert-Disziplin (eingeführt 22.05.2026).** Bevor ein neuer Wert für eine Status-/Enum-Spalte in Paula (oder jeder Postgres-Tabelle) im Code geschrieben wird, IMMER die echten DB-`CHECK`-Constraints prüfen — nicht nur die Drizzle-Schema-Datei. Drizzle-`text()`-Spalten mit Freitext-Kommentar (`// offen|beantwortet|…`) verbergen, dass die DB eine `CHECK`-Constraint trägt, die den neuen Wert ablehnt. Prüf-Kommando: `SELECT pg_get_constraintdef(oid) FROM pg_constraint WHERE conrelid='<tabelle>'::regclass AND contype='c';`. Wenn eine Constraint existiert → Migration die sie per `DROP CONSTRAINT IF EXISTS` + `ADD CONSTRAINT` mit dem erweiterten `ARRAY` neu setzt. Memory-Lesson `750e2bb4`. (Herkunft → guardrails-historie.md [H20])

## 🚀 Paula-Deploy (Production) — der EINE Weg

**Die Paula-API/-Web wird NICHT über Dokploy deployed.** Die Dokploy-App `Inge/api` ist verwaist (letzter Deploy April 2026), und **`github.com/attendorn/inge` ist eine TOTE Spur** (April-Skeleton, „hello-world api"). Nicht dorthin pushen — das triggert nichts. Der echte Deploy ist ein Blue/Green-Script:

```bash
bash ~/paula/scripts/deploy-paula-prod.sh --dry-run   # IMMER zuerst (zeigt den Plan)
bash ~/paula/scripts/deploy-paula-prod.sh api          # nur API-Pool (Rolling blue→green)
bash ~/paula/scripts/deploy-paula-prod.sh web          # nur Web-Pool
```

**DB-Migrationen separat VOR dem Code-Deploy** (additiv, Migrations-Datei in `~/paula/packages/db/migrations/`). **Cross-Session-Deploy-Check (PFLICHT):** das Script deployt immer den ganzen `main`-HEAD — vor jedem Deploy `git fetch origin main` und auf fremde Commits prüfen; sind welche da, NICHT blind deployen, sondern Florian klären lassen. Ablauf-Details, Server-Pfade und Anlass-Chronik: `memory/topics/paula-deploy.md`.

## Web-Deploy (Vercel), Formulare, Multi-Service

**VERBOTEN (ohne explizite Freigabe):**
- **Website-Deploy (Vercel) ohne Florians explizite Freigabe.** Änderungen zeigen (Vorher/Nachher) → Freigabe abwarten → `vercel --prod`. **Nach jedem Deploy: Smoke-Test** – betroffene Formulare/APIs einmal durchklicken und E-Mail-Eingang verifizieren.
- **Formular-Feld-Änderung OHNE Endpoint-Check.** Bei jedem Umbau von HTML-Formularfeldern (hinzufügen/umbenennen/löschen) MUSS der zugehörige API-Endpoint (`api/*.js`) auf Feld-Konsistenz geprüft werden **bevor** der Deploy rausgeht. Konkret: `name=`-Attribute im HTML vs. `data.<Feldname>`-Zugriffe im Endpoint abgleichen. Hardcoded Feld-Listen im Endpoint sind ein Warnsignal – besser dynamisches Rendering mit Label-Mapping. [H1]
- **Compose-Stop / Multi-Service-Operationen.** Bei Dokploy-`compose.stop`-API oder analogen Multi-Service-Aktionen (Compose mit ≥2 Services) NIEMALS ohne explizite Florian-Klärung was alles offline geht. Default-Verhalten: vorher kurz auflisten welche Services/Container betroffen sind, dann fragen ob alle gestoppt werden sollen — oder nur einzelnen Container via `docker.stopContainer`-API stoppen. [H4]

## Bau-Disziplin

- **DB-Lookup-Konvention (Sample-Size-Disziplin).** Bei jedem DB-Lookup auf Personen/Mitarbeiter/Kontakten/Kunden, der die Identität eines Datensatzes braucht, MUSS ein eindeutiger Identifier als Filter genutzt werden — Email, UUID, externer Schlüssel. **Verboten: nacktes `LIMIT 1` oder `ORDER BY created_at LIMIT 1` ohne semantische Verifikation.** Wenn das Skript nur einen Treffer braucht aber mehrere Kandidaten in der Tabelle stehen können, gehört vorher ein Email-Match dazu. Wurzel-Bias: `sample_size_blind`. [H2]
- **Voraussetzungs-Check vor Bau-Aufträgen.** Bevor Laura einen Workflow/Deliverable/System-Baustein baut (Website, Skill, Agent, CRM-Flow, Automatisierung, Delegations-Struktur), MUSS sie zuerst die Voraussetzungen prüfen: **Welche dauerhaften Grundlagen müssen stehen, damit das System trägt?** Beispiele: Website → Brand Style Guide · Mitarbeiter-Delegation → Mitarbeiter-DB (gepflegt, kuratiert, zugänglich) · Kundenkommunikation → Kontakte-SSOT + Tonalität · Content-Produktion → Zielgruppen + Freigabe-Flow · Assistent → Rollen + Regeln + Memory-Struktur. Fehlt die Grundlage: Erst diese klären/anlegen, dann bauen. Nicht ad hoc im Chat aufbauen was eigentlich ein persistentes System sein muss. Auch fragen: Wie kommen die Grundlagen-Daten rein? Wer kuratiert sie? Wie lange bleiben sie drin? Diese Regel gilt UND gleichzeitig ist sie zentrale Denkfigur für Workshop #1 (Get the Workflow).
- **Zugangsdaten-Maskierung bei Ausgaben.** Bestimmte Kommandos geben Zugangsdaten mit aus, ohne dass man danach gefragt hat. Ihre Ausgabe wird deshalb **immer** durch einen Maskierer geschickt, ohne vorher zu prüfen, ob gerade eines enthalten ist: `git remote -v` / `git remote get-url`, `git config --list` und `git config --get-regexp`, `cat .git/config`, `env` / `printenv` / `set`, sowie `curl -v` und jede Ausgabe von Verbindungs-Strings (DB-URLs, Webhook-URLs, MCP-Configs). Muster: `| sed -E 's#://[^@/]+@#://<credentials>@#'`, bei Env-Dumps zusätzlich Schlüssel mit `TOKEN|KEY|SECRET|PASSWORD|PAT` filtern. **Der Handgriff ersetzt die Aufmerksamkeit, er ergänzt sie nicht** — im Moment des Fehlers ist die Aufmerksamkeit beim gesuchten Geheimnis gebunden, nicht beim beiläufigen Befehl. Zweite Hälfte der Regel: Ein Token gehört nicht in eine Remote-URL, sondern in den Credential-Helper (macOS-Keychain) oder auf einen SSH-Key; in der URL ist es eine Dauerfalle, die bei jedem beiläufigen Kommando erneut herausrutscht. Abgrenzung zu [H13]: der Pre-Send-OpSec-Check prüft, was **nach außen** geht, diese Regel schützt die Ausgabe **an Florian selbst** — Chat-Transkripte sind kein privater Raum. [H23]
- **Pre-Bau-Verify bei Multi-Komponenten-Layout:** Vor jedem CSS-/Layout-Edit, der mehr als eine Komponente betrifft, ZUERST den Verify-Schritt denken: FullPage-Screenshot plus Mobile-Viewport-Test der betroffenen Zustände, bevor „läuft" gesagt wird. Wurzel `Bau-Tempo schlägt Sorgfalt`. (Herkunft → guardrails-historie.md [H30])

## Architektur-Aufträge mit Buzzwords ("generalisieren", "modul", "studio")
- **Mental-Model VOR Datenmodell klären.** Bei jedem Architektur-Auftrag mit abstrakten Wörtern wie „generalisieren", „X als Modul", „aus jeder Entity X" zuerst rückfragen WIE der Trigger/Einstieg aussehen soll, nicht nur WAS technisch generalisiert wird. Rückfrage-Pflicht in der Brainstorm-Phase: „Wo soll Florian den Trigger sehen — Button in der Detail-Page? Chat-Eingabe? Slash-Command? Listen-Aktion?" Erst Mental-Model-Bestätigung, dann Datenmodell + UI gemeinsam. (Herkunft → guardrails-historie.md [H18])

## Architektur-Historie

Bevor eine abgeschaffte Lösung erneut diskutiert wird: **`~/Laura/memory/topics/architektur-historie.md`** nachschlagen. Dort stehen ersetzte/entfernte Features mit Datum, Grund und Nachfolger. Bei jedem neuen "wir könnten das so bauen"-Gedanken zuerst prüfen, ob das schonmal da war. Die Datei wird nicht bei `/laura` automatisch geladen.
