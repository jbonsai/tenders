# CLAUDE.md – Ausschreibungs-Scraper

## Zweck
Sammelt öffentliche Ausschreibungen aus festgelegten Quellen, bringt sie in ein einheitliches Datenmodell, entfernt Duplikate, filtert nach Relevanz (Suchprofil in `config/profile.toml`) und lässt die verbleibenden Treffer von einer KI bewerten: Passt die Ausschreibung zu mir?
Ergebnis: lokale SQLite-Datenbank plus Export der neuen, passenden Treffer (CSV/JSON).

Status: im Aufbau, Einzelentwickler, kein Produktivbetrieb.

**Zielbild vs. Ist-Stand:** Stack, Struktur und Befehle unten beschreiben das Zielbild. Heute gibt es nur einen Prototyp in `scraper/` (`requests` + BeautifulSoup gegen die HTML-Tenderliste des USP). Er wird in die Zielstruktur überführt und dann entfernt. Nicht annehmen, dass Dateien aus dem Zielbild schon existieren.

## Datenquelle (Österreich)
Erste und vorerst einzige Quelle: die gesetzlich vorgeschriebenen Open-Data-Bekanntmachungen nach BVergG 2018, **nicht** das HTML des USP.

- Auftraggeber veröffentlichen **Metadaten** auf data.gv.at (Tag „Ausschreibung“, meist ein Datensatz pro veröffentlichender Stelle).
- Jeder Metadatensatz verweist auf eine **Kerndatenquelle**: eine maschinenlesbare XML-Liste von Verweisen auf einzelne Bekanntmachungen.
- Die Bekanntmachungen liegen auf den Servern der Auftraggeber bzw. Vergabeplattformen → viele Hosts, Rate-Limit pro Host.
- **Ab 01.10.2026 sind die Bekanntmachungen eForms** (EU-Standard, Schema aus dem eForms-SDK des EU-Amts für Veröffentlichungen). Das alte Kerndaten-Schema kommt nur noch bei Datensätzen mit `lastmod` bis 30.09.2026 vor.
- Das USP (ausschreibungen.usp.gv.at) zeigt nur an, was über diese Metadaten gefunden wird, und hat keine API. Es dient höchstens zum Gegenprüfen per Hand.
- Rechtsgrundlage: §§ 54, 59, 64 BVergG 2018 idF BGBl. I Nr. 8/2026 (Vergaberechtsgesetz 2026); BMJ-Rundschreiben „Einführung eForms“ (Stand 01.06.2026). Recherche vom 27.09.2026.

API-Details (Endpunkt, Filter, Antwortstruktur): siehe `datenquelle-data-gv-at.md`. Noch zu klären: ob die Nachfolgeverordnung der Kerndaten-VO (BGBl. II Nr. 57/2019) inzwischen kundgemacht ist.

## Stack
- Python 3.14 (siehe `.python-version`), Paketverwaltung ausschließlich mit **uv** (kein pip, kein poetry)
- HTTP: `httpx` (async), Retries mit `tenacity`
- XML-Parsing (eForms, Kerndatenquellen): `lxml`
- HTML-Parsing, falls nötig: `selectolax`; `beautifulsoup4` nur, wenn selectolax nicht reicht
- Datenmodelle: **Pydantic v2** (keine dataclasses), Settings über `pydantic-settings`
- CLI: `typer` (Entry-Point `scraper` in `pyproject.toml`)
- Speicher: SQLite (`sqlite3` aus der Standardbibliothek), Datei unter `data/`
- Tests: `pytest`, `respx` zum Mocken von httpx
- Lint/Format: `ruff`; Typprüfung: `basedpyright`
- KI-Bewertung: Anbieter und Modell noch offen – nicht eigenmächtig festlegen
- Playwright nur, wenn eine Quelle ohne JavaScript-Rendering nachweislich nicht funktioniert – vorher fragen

## Befehle
```zsh
uv sync                                   # Abhängigkeiten installieren
uv run pytest                             # alle Tests (offline)
uv run pytest -m live                     # Live-Tests gegen echte Quellen – nur auf Anweisung
uv run ruff check . --fix && uv run ruff format .
uv run basedpyright                       # Typprüfung
uv run scraper run --source <name> --dry-run   # Quelle abrufen, nichts speichern
uv run scraper run --all                  # alle aktiven Quellen
uv run scraper assess                     # noch nicht bewertete Treffer von der KI bewerten lassen
uv run scraper export --since 7d --format csv
```
Eine Aufgabe ist erst fertig, wenn `ruff check`, `ruff format`, `basedpyright` und `pytest` durchlaufen.

## Projektstruktur (Zielbild)
```
src/tender_scraper/
  cli.py            # Typer-Einstiegspunkt
  settings.py       # pydantic-settings, liest .env
  models.py         # Tender, Assessment, Quellen-Metadaten, Enums
  http.py           # gemeinsamer Client: User-Agent, Rate-Limit pro Host, Retries, Roh-Cache
  sources/
    base.py         # Protokoll `Source`
    <quelle>.py     # eine Datei pro Quelle
  formats/          # offen, noch gemeinsam festzulegen (siehe Architektur)
    eforms.py       # eForms-Parser, quellenunabhängig (auch für TED wiederverwendbar)
  normalize.py      # Datum, Beträge, CPV-Codes, Auftraggebernamen
  dedupe.py
  filters.py        # regelbasierte Vorauswahl nach Suchprofil
  assess.py         # KI-Bewertung
  storage.py        # SQLite-Zugriff, Schema-Migration
tests/
  fixtures/<quelle>/   # gespeicherte Original-Antworten (HTML/JSON/XML)
config/
  sources.toml      # aktive Quellen, Intervalle, Rate-Limits
  profile.toml      # Suchprofil: CPV-Codes, Schlagwörter, Ausschlüsse, Regionen, Beschreibung für die KI
data/               # gitignored: SQLite, Roh-Cache, Exporte
```

## Architektur
Pipeline pro Quelle: **fetch → parse → normalize → dedupe → filter → store → assess**.
Jede Stufe ist eine eigene Funktion; nur `storage.py` schreibt in die Datenbank.

- **Eine Quelle, aber abstrahiert:** Wir starten mit genau einer Quelle und bereiten die Abstraktion (Protokoll `Source` usw.) von Anfang an vor. **Wie genau (Zuschnitt des Protokolls, ob `formats/` für eForms eigenständig wird) ist noch nicht entschieden und wird gemeinsam festgelegt**, bevor Code dafür entsteht.
- `http.py` ist der einzige Ort, der Netzwerk-Requests macht. Quellen importieren nie direkt `httpx`.
- Jede Rohantwort wird vor dem Parsen unter `data/raw/<quelle>/<datum>/` abgelegt → Parser können ohne neuen Abruf erneut laufen.
- Parser arbeiten nur auf Rohdaten (str/bytes), nie auf dem Netzwerk → direkt mit Fixtures testbar.

### KI-Bewertung (`assess`)
- Läuft nur auf Treffern, die den regelbasierten Filter passiert haben (spart Kosten).
- Jede Ausschreibung wird höchstens einmal pro Profilversion bewertet; das Ergebnis wird gespeichert.
- Die KI bekommt nur Daten aus dem `Tender`-Modell und das Profil, keine personenbezogenen Daten.
- Antwort strukturiert (Pydantic-validiert), nicht als Freitext parsen.

### Neue Quelle hinzufügen
1. Prüfen, ob es eine offizielle API, einen Open-Data-Feed oder RSS gibt. Wenn ja: **das** statt HTML-Scraping verwenden.
2. robots.txt und Nutzungsbedingungen lesen; Ergebnis als Kommentar oben in `sources/<quelle>.py` festhalten (URL + Datum).
3. Beispielantworten nach `tests/fixtures/<quelle>/` speichern (einmaliger Abruf, nur mit Freigabe).
4. Protokoll `Source` implementieren, Parser gegen Fixtures testen.
5. In `config/sources.toml` mit Rate-Limit eintragen, zunächst `active = false`.

## Datenmodell (Kern)
`Tender` (Pydantic). Pflicht: `source`, `source_id`, `title`, `contracting_authority`, `url`, `published_at`, `fetched_at`.
Optional: `deadline`, `cpv_codes`, `procedure_type`, `estimated_value` + `currency`, `region` (NUTS), `lots`, `description`, `documents` (URLs), `raw_fields`.

`Assessment` (Pydantic): `source`, `source_id`, `verdict` (passt / passt nicht / unklar), `reasoning`, `model`, `profile_version`, `assessed_at`.

Regeln:
- Eindeutiger Schlüssel: `(source, source_id)`. Dieselbe Ausschreibung in mehreren Quellen über die Referenznummer erkennen und **verknüpfen, nicht löschen**.
- Alle Zeitpunkte als zeitzonenbewusstes `datetime`: beim Parsen die Zeitzone der Quelle (Default Europe/Vienna), gespeichert wird in UTC.
- Beträge als `Decimal`, nie `float`.
- **Nichts raten:** Wenn sich ein Feld (vor allem Frist oder Betrag) nicht eindeutig parsen lässt → Feld `None`, Originaltext in `raw_fields` behalten, Warnung loggen. Keine Default-Fristen, keine geschätzten Werte.
- `raw_fields: dict[str, str]` immer mitspeichern, damit die Normalisierung nachvollziehbar bleibt.

## Scraping-Regeln (verbindlich)
- Ehrlicher User-Agent mit Kontaktadresse aus `.env` (`SCRAPER_CONTACT`).
- robots.txt respektieren. Rate-Limit pro Quelle aus `sources.toml`; Standard: höchstens 1 Request/Sekunde, keine parallelen Requests auf denselben Host.
- Bei 429/503: exponentielles Backoff, `Retry-After` beachten; nach 3 Fehlversuchen die Quelle (bzw. den Host) für diesen Lauf abbrechen.
- Bedingte Requests (`ETag` / `If-Modified-Since`) nutzen, wo die Quelle sie unterstützt; bei Kerndatenquellen zusätzlich `lastmod` auswerten, um nur Geändertes abzurufen.
- Keine Logins, keine Paywalls, keine kostenpflichtigen Portale, kein Umgehen von Captchas oder Sperren.
- Ausschreibungsunterlagen (PDFs) nur verlinken; Download nur einzeln und auf Anweisung.
- Personenbezogene Daten (Namen, Telefonnummern von Kontaktpersonen) nicht in Exporte und nicht an die KI übergeben.
- `.env` und `data/` werden nie eingecheckt.

## Tests
- Der Standard-Testlauf ist **offline**: Kein Test darf das Netzwerk erreichen. respx blockt nur innerhalb eines aktiven Mocks – deshalb eine globale Netzwerksperre per autouse-Fixture (oder `pytest-socket`, nach Freigabe).
- Jeder Parser hat Tests gegen echte, gespeicherte Fixtures, inklusive Grenzfällen: fehlende Frist, mehrere Lose, abweichendes Datumsformat.
- Die KI-Bewertung wird in Tests gemockt; echte KI-Aufrufe nur als Live-Test.
- Live-Tests mit `@pytest.mark.live` markieren; sie laufen nur explizit.
- Wenn eine Quelle bricht (Layout- oder Schemaänderung): zuerst neue Fixture sichern, dann den Parser anpassen.

## Code-Stil
- Code, Bezeichner und Commit-Messages auf Englisch; Doku und Kommentare zur Fachlogik dürfen Deutsch sein.
- Durchgehend Type Hints, basedpyright ohne Fehler. Kein `Any` ohne Begründungskommentar.
- Kleine, reine Funktionen; kein globaler Zustand außer den Settings.
- Logging über `logging` mit Kontext (Quelle, source_id, Stufe); `print` nur in der CLI.
- Kein stilles `except Exception: pass`. Fehler bei einem Datensatz: loggen und weitermachen. Fehler bei einer ganzen Quelle: nach oben melden.

## Editor: Zed
- Projekteinstellungen in `.zed/settings.json` (eingecheckt).
- Typprüfung mit basedpyright, konfiguriert in `pyproject.toml` unter `[tool.basedpyright]`. **Keine** `pyrightconfig.json` anlegen, sie hätte Vorrang und würde die pyproject-Konfiguration überdecken.
- Ruff ist Formatter und Linter, konfiguriert in `pyproject.toml` unter `[tool.ruff]`.
- Diese `CLAUDE.md` ist die einzige Regeldatei. **Keine** `.rules`, `AGENTS.md`, `.cursorrules` o. ä. anlegen: Zed verwendet nur die erste gefundene Regeldatei, und `CLAUDE.md` steht in dieser Reihenfolge weit hinten.

## Arbeitsweise
- Kommunikation mit mir auf Deutsch.
- Vor neuen Abhängigkeiten fragen (`uv add` nur nach Freigabe).
- Vor jedem Live-Abruf gegen eine echte Quelle fragen, auch für Fixtures.
- Keine Commits, Pushes oder geplanten Läufe (cron/launchd) ohne ausdrückliche Anweisung.
- Annahmen über das Format einer Quelle offenlegen und mit Fixtures belegen, nicht aus dem Gedächtnis.
- Bei Fachbegriffen des Vergaberechts (Verfahrensarten, Schwellenwerte) nachfragen statt annehmen.

## Offene Punkte
- [x] Erste Quelle: Open-Data-Bekanntmachungen über data.gv.at (siehe „Datenquelle“)
- [x] Endpunkt der data.gv.at-API geklärt, siehe `datenquelle-data-gv-at.md`
- [ ] Aufbau einer Kerndatenquelle ansehen (einzelner Live-Abruf, mit Freigabe)
- [ ] Abstraktion gemeinsam festlegen: Zuschnitt des `Source`-Protokolls, eigenständiges `formats/` für eForms ja/nein
- [ ] Suchprofil (CPV-Codes, Schlagwörter, Regionen, Beschreibung für die KI) in `config/profile.toml`
- [ ] KI-Anbieter und Modell für die Bewertung
- [ ] Benachrichtigung: vorerst keine, Treffer werden händisch über den Export gesichtet. Später Slack oder E-Mail.
- [ ] Ausführung: vorerst lokal von Hand. Geplante Läufe erst später.
