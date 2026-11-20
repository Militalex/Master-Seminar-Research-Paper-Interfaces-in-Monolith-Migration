# agents.md – Typst-Wissen für zukünftige KI-Agenten

Erwartungen/Projektkontext: Master-Seminar-Dokumente in **Typst**, bearbeitet in VS Code mit der
**TinyMist-Extension** (Typst-LSP). Hilfsfunktionen liegen z. B. in
`00 assets/template-Functions-V1.0a.typ`.

## Geprüfte Umgebung (Stand: November 2026)

TinyMist's Typst-Engine ist **neu genug für alle modernen Array-Methoden** (≥ 0.12). Der Fehler `type array has no method 'sort'` resultierte aus einem **falschen Methodennamen** – die korrekte Methode heißt `sorted()`, nicht `sort()`. Somit liegt kein Versionsproblem vor.

- **Verifiziert (kompiliert im aktuellen Template):**
  - Grundlegende Methoden: `.len()`, `.zip()`, `.map()`, `.enumerate()`, `.at()`
  - Sortierung & Extremwerte: `.sorted()`, `.first()`, `.last()`
  - Hinweis: `sorted()` ist eine **Methode**; es gibt zudem die globale Funktion `sort(array)`, die ein neues, sortiertes Array zurückgibt.

### Maximum/Minimum eines Arrays (idiomatisch)

```typ
let y-sorted = y-values.sorted()
let yMin = y-sorted.first()
let yMax = y-sorted.last()
```

- `.max()` ist **keine** Array‑Methode in Typst (steht nicht in der Array‑Doku). Die globale, variadische Funktion `max(..arr)` funktioniert, ist jedoch unhandlich, wenn sowohl Minimum als auch Maximum benötigt werden.
- **Fallback für sehr alte Engines (< 0.12):**
  ```typ
  let m = 0
  for y in y-values {
    if y > m { m = y }
  }
  ```
  (Verwendet keine Methoden, die erst ab 0.12 eingeführt wurden.)

## Inline‑if in Typst

- `if … else` ist ein **Ausdruck** (kein ternärer Operator `?:`). Beispiel:
  `#let x = if 5 > 3 { "ja" } else { "nein" }`
- Ein reines `if …` ohne `else` liefert `none`. Verwende das nur, wenn `none` an der jeweiligen Stelle akzeptiert wird.
- **Spreading mit `..if` (verifiziert im Template):**
  ```typ
  ..if showNumbers {
    range(n).zip(values).map(((x, y)) => lq.place(...))
  }
  ```
  - `..(array)` zerlegt das Array in einzelne Argumente.
  - Der `else`‑Zweig ist **nicht nötig**: fällt die Bedingung aus, liefert `..none`, wodurch **keine Argumente** gespreadet werden (No‑op).
  - Für **ältere Typst‑Versionen (< 0.12)** kann ein expliziter `else { () }` nötig sein, weil dort `none` nicht automatisch als leerer Spreading‑Wert behandelt wird.

## lilaq (Plotting‑Bibliothek, `#import "@preview/lilaq:0.6.0" as lq`)

Doku: https://lilaq.org (Reference: `/docs/reference/<name>`)

### Werte über Säulen beschriften (offizielles Muster)

Aus dem lilaq-Beispiel "Bar plot with numbers" – `lq.place` + zip/map + Spreading:

```typ
lq.diagram(
  ...,
  lq.bar(range(ys.len()), ys),
  ..range(ys.len()).zip(ys).map(((x, y)) =>
    lq.place(x, y, align: bottom, pad(0.2em)[*#y*])
  )
)
```

- `lq.place(x, y, body, align: bottom)` → Label **auf** der Säulenspitze (darunter ausgerichtet)
- `align: top` → Label **in** der Säule (nur sinnvoll bei hohen Säulen + halbtransparente Füllung)
- Kollision mit Diagrammrand vermeiden: Label abhängig von y-Wert innen/aussen schalten:
  `let align = if y > 0.85 * maxVal { top } else { bottom }`
- `place` ist default `clip: false`, `z-index: 21` → Labels erscheinen über allen Plots

### Y-Achse begrenzen

```typ
yaxis: (
  lim: (yMin, yMax),   // Tupel; einzelne Seiten auch: (auto, 35)
),
```

- Puffer für oberhalb stehende Labels einbauen, z. B. `yMax = maxVal + 4`
- (Äquivalent: `ylim` direkt auf `lq.diagram`.)

### X-Achsen-Labels drehen (45°)

```typ
show: lq.show_(
  lq.tick-label.with(kind: "x"),
  it => box(
    width: 0pt,
    align(right, rotate(-45deg, reflow: true, it))
  ),
)
```

Kombiniert mit `xaxis: (ticks: x-labels.enumerate(), subticks: none)`.

### lilaq-Quellen zum Nachschlagen

- `lq.bar`: https://lilaq.org/docs/reference/bar
- `lq.place`: https://lilaq.org/docs/reference/place
- `lq.axis` (u. a. `lim`, `ticks`): https://lilaq.org/docs/reference/axis
- Beispiel mit Zahlen: https://lilaq.org/docs/examples/bar-plot-with-numbers

## Workflow-Hinweise

- Bei Typprobieren: Fehlermeldungen von TinyMist (Problems-Panel) als Autorität behandeln –
  sie referenzieren die lokal gebündelte Typst-Engine, nicht "die neueste".
- Version prüfen: VS Code `Strg+Shift+P` → "Typst: Show Log"; nach Engine-Updates
  "Typst: Restart language server".
- Doku-Links mit Anker (`#...`) sind in Typst-Doku-URLs oft nicht direkt auflösbar (404);
  Seitenbasis-URLs ohne Anker nutzen.

# Zotero‑Export‑Agent – Überblick für KI‑Agenten

## Ziel & Kontext
- Das Skript `zotero_export.py` exportiert eine Zotero‑Sammlung als **Typst**‑Dokumente (und optional PDFs).

## Kern‑Workflow
1. **Datenbank‑Kopie** – Eine temporäre Kopie von `zotero.sqlite` wird erstellt, um die Original‑DB nicht zu blockieren.
2. **Sammlungsauflösung** – Der Ziel‑Collection‑Name (`TARGET_COLLECTION`) wird über `load_collections`, `find_target_collection` und `get_descendant_collection_ids` ermittelt.
3. **Item‑Verarbeitung** – Für jedes Item werden
   - Metadaten (`ItemMetadata`)
   - PDF‑Attachments
   - Notizen (annotated‑bibs & reguläre Notizen)
   - **Tags** analysiert.
4. **Tag‑Parsing** (`parse_ethereal_tags`)
   - Trennen von **Etheral‑Etcher‑Tags** (beginnen mit `#`) und **normalen Tags**.
   - Erkennen von Ratings, Lesefortschritt, Paper‑Fortschritt, Source‑Info, Refinement‑IC/EC‑Kriterien und freien Extra‑Refinement‑Hinweisen.
   - Aufbau einer Baum‑Struktur für die Etheral‑Tags.
5. **Metadaten‑Blöcke erzeugen** (`build_metadata_blocks`)
   - `Citation`‑ und `Rating`‑Blöcke immer zuerst.
   - Rekursive Darstellung der Tag‑Hierarchie als verschachtelte `#block`‑Strukturen.
   - Abschließender `Tags`‑Block für alle nicht‑`#`‑Tags.
   - Keine separaten Blöcke mehr für *Paper Information*, *Own Keywords*, *Reading Progress* usw.; sie sind jetzt in der generischen Hierarchie abgebildet.
6. **Datei‑Erstellung** – `write_note_file`, `write_pdf`, `write_info_file` erzeugen `.typ`‑Dateien und optional PDFs (mittels `typst compile`).
7. **Parallel‑Compilation** – PDFs werden über einen Thread‑Pool (`TYPST_COMPILE_WORKERS`) gebaut.

## Datenmodelle (Dataclasses)
- `ItemMetadata`: speichert alle relevanten Felder eines Zotero‑Eintrags (Titel, Autoren, Jahr, Tags, etc.).
- `StagedNote`: enthält den gerenderten Notiz‑Text, zugehörigen Tag und Index‑Informationen.
- `ParsedTags`: hält strukturierte Listen für
  - `ratings`
  - `reading_progress`
  - `paper_progress`
  - `source_info`
  - `slr_criteria`
  - `extra_refinement`
  - `category_tags`
  - `ethereal_tags`
  - `normal_tags`

## Anforderungen / Abhängigkeiten
- **Python ≥ 3.11** (type hints, `match`‑Expression).
- **pypandoc** – konvertiert HTML‑Notizen zu Typst.
- **typst** (CLI) – zum Kompilieren von Typst‑Dateien zu PDFs.
- **sqlite3** – zum Lesen der Zotero‑Datenbank.
- **shutil**, **tempfile**, **Pathlib**, **re**, **collections** (Standard‑Bibliothek).

## Wichtige Design‑Entscheidungen
- **Generische Tag‑Hierarchie**: Jede Etheral‑Etcher‑Tag‑Sequenz (`#A/B/C`) wird automatisch zu verschachtelten Blöcken, ohne hard‑coded Fallunterscheidungen.
- **Farbschema**: 5 Graustufen werden zyklisch vor‑ und rückwärts durchlaufen (0→1→2→3→4→3→2→1→0…).
- **Rating‑Platzierung**: Direkt nach dem Citation‑Block, weil das in den Anforderungen gefordert war.
- **Keine separaten Classic‑Blöcke** mehr – alles wird über die Hierarchie‑Logik abgedeckt, wodurch das System leicht erweiterbar ist.
- **Normal‑Tags** werden am Ende in einem einzigen `Tags`‑Block gesammelt, sodass nicht‑hierarchische Tags nicht verloren gehen.

## Erweiterbarkeit
- Neue Etheral‑Tags benötigen **keine Code‑Änderung** – das Parsen ist generisch, basierend auf dem führenden `#`.
- Zusätzliche Block‑Typen können durch Anpassen von `add_block` in `build_metadata_blocks` eingeführt werden.
- Für weitere Farben kann die `_depth_colors`‑Liste erweitert werden; die Logik für den Vor‑/Rückwärts‑Durchlauf bleibt unverändert.

## Nutzungshinweise für KI‑Agenten
- Beim Aufruf von `export_zotero()` wird automatisch geprüft, ob Pandoc verfügbar ist; falls nicht, wird ein interaktiver Download angeboten.
- Der Export‑Pfad `EXPORT_DIR` ist relativ zum Skript‑Verzeichnis; das Skript erstellt die Verzeichnis‑Struktur (inkl. Sub‑Folders) nach den generierten Dateinamen.
- Die erzeugten `.typ`‑Dateien enthalten bereits die strukturierten Metadaten‑Blöcke – weitere Nachbearbeitung ist nicht nötig, solange das Tag‑Schema korrekt verwendet wird.
- **Fehler‑Handling** ist robust: fehlende PDFs, leere Demo‑Notizen oder nicht‑gefundene Tags führen zu Warnungen, aber nicht zum Abbruch des gesamten Exports.

---
*Dieses Dokument dient als kompakte Wissensbasis für weitere KI‑Agenten, die das Zotero‑Export‑Skript warten, anpassen oder in andere Workflows einbinden müssen.*
