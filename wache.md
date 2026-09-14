---
paths:
  - "code/scripts/*-wache.sh"
  - "code/scripts/*-signale.sh"
  - "code/scripts/routinen-health.sh"
  - "code/scripts/melder/**"
  - "code/hooks/rm-guard.sh"
  - "code/hooks/check-read-token-limit.sh"
---

# Wache – Regeln für Melder, Wächter und Betriebsbeobachtung

> Abgetrennt aus `rules/guardrails.md` und aus der mit Plan 0930 entfernten Datei `rules/architecture.md`
> (14.09.2026). Diese Datei
> lädt **nicht** in jeder Sitzung: sie hängt an den Pfaden oben und wird gelesen, wer an der Wache baut.
> Die Regeln für Laura im Gespräch stehen weiter in `rules/guardrails.md`.
> `[Hn]`-Marker = Anlass, Datum und Memory-Lesson stehen in `memory/topics/guardrails-historie.md`.

## Aufnahme-Regel vor jedem neuen Melder/Wächter

Bevor ein Alarm-Ordner, eine Startkanal-Zeile oder eine `wartet_pruefung` gebaut wird: die vier Sätze in
`memory/topics/pa-system.md` → „Aufnahme-Regel für Melder und Wächter" durchgehen. Kurzform:
eine Datei je **Zustand** statt je Lauf · jede Meldung muss **auflösbar** sein · die Sonde misst
das **Ereignis**, nicht benachbarten Erfolg, und wird in beide Richtungen gegengeprüft · die
Benachrichtigung ist nicht das Archiv. [H28]

## Was die Hooks erzwingen — und was das im Bau bedeutet

Im Sockel steht je ein Kurzsatz; hier stehen die Ausführungsdetails, die beim Bau an der Wache gebraucht
werden.

- **Read-Token-Limit-Pflicht.** Wenn ein `Read`-Tool-Call mit `"exceeds maximum allowed tokens"` fehlschlägt,
  MUSS die Datei in Chunks über `offset`/`limit` nachgeladen werden — Default 150 Zeilen pro Chunk. Verboten:
  Datei überspringen, Zusammenfassung aus anderer Quelle als Ersatz nehmen, „reicht erstmal"-Annahme. Die
  Information die gerade gebraucht wird kann im ungeladenen Teil stehen. Hook `code/hooks/check-read-token-limit.sh`
  (PostToolUse/Read) injiziert die Pflicht-Anweisung in den nächsten LLM-Turn. [H5]
- **Rekursives Löschen.** `code/hooks/rm-guard.sh` (PreToolUse/Bash): **rekursives** Löschen (`rm -r`, `rm -rf`)
  nur in Wegwerf-Pfaden — Scratchpad, `/tmp`, `node_modules`, `.next`, `.cache`, `.turbo`, `.playwright-mcp`.
  Überall sonst geblockt, mit Hinweis auf den gangbaren Weg. Einzelne Dateien zu löschen war nie gesperrt und
  bleibt erlaubt; für Kunden- und OneDrive-Inhalte gilt weiter der Weg über `dokumente.py` bzw. Florian fragen. [H44]

## Schattenlauf bei teurer Arbeit

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

## Versions-Awareness

- Changelog + Workflow-Impact: `memory/topics/claude-code-updates.md` (dort auch Prüf-/Update-Befehle)
- **Default Effort = High** seit 2.1.94 (steuerbar via `/effort`)
