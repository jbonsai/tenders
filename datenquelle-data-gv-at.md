# Datenquelle data.gv.at – Erkundung der API

Stand: 27.09.2026. Zwei einzelne Live-Aufrufe, beide mit HTTP 200.

## Ergebnis in Kürze

- Die Katalog-API von data.gv.at ist ohne Anmeldung nutzbar und liefert JSON.
- Die Metadatensätze nach BVergG 2018 erkennt man am Tag `ausschreibung`. Die Volltextsuche nach „Ausschreibung“ ist unbrauchbar, weil sie vor allem Stellenausschreibungen nach dem Ausschreibungsgesetz 1989 liefert.
- Jeder Metadatensatz hat eine Distribution mit dem Titel `Kerndatenquelle`, Format XML und Lizenz CC BY 4.0. Deren `access_url` ist die URL der Kerndatenquelle auf dem Server des Auftraggebers oder der Vergabeplattform.
- Mit dem Tag-Filter gibt es **57 Treffer**. Nicht alle sind Metadatensätze nach BVergG (siehe unten), und ob die 57 vollständig sind, ist offen.

## Endpunkt

```
GET https://www.data.gv.at/api/hub/search/search
```

| Parameter | Wert | Bedeutung |
|---|---|---|
| `filters` | `dataset` | nur Datensätze |
| `facets` | `{"keywords":["ausschreibung"]}` | Tag-Filter (JSON als String) |
| `limit` | `5` (max. 1000) | Treffer pro Seite |
| `page` | `0`, `1`, … | Seite, nicht getestet |

Die API gehört zu piveau (Hub-Search). Die OpenAPI-Beschreibung von piveau nennt außerdem `searchAfter`/`pitId` für große Ergebnismengen. Bei 57 Treffern reicht `limit`.

### Aufruf 1 – Volltext (verworfen)

```
GET /api/hub/search/search?q=Ausschreibung&filters=dataset&limit=3
```

Das ergab 169 Treffer, darunter vor allem Funktionsausschreibungen des BMLV nach dem Ausschreibungsgesetz 1989, IFG-Veröffentlichungen und Regierungsbeschlüsse. Für uns ist das Rauschen.

### Aufruf 2 – Tag-Filter (verwendet)

```
GET /api/hub/search/search?filters=dataset&facets={"keywords":["ausschreibung"]}&limit=5
```

Das ergab 57 Treffer. Die Formate unter den Treffern: XML 41, PDF 11, JSON 8.

## Aufbau der Antwort

```
result
  index    "dataset"
  count    57
  facets   [...]   Aggregationen (catalog, publisher, keywords, format, …)
  results  [...]   Datensätze
```

Beispiel eines Metadatensatzes (gekürzt, nur relevante Felder):

```json
{
  "id": "39e2a593-d508-4f64-8a04-8d6dbf730f9d",
  "title": { "de": "Metadatensatz zur Kerndaten-Sammlung gemäß BVergG 2018 von 'Österreichische Post AG'" },
  "description": { "de": "Verweis auf die Kerndaten-Sammlung gemäß BVergG 2018 von 'Österreichische Post AG'" },
  "keywords": [ { "language": "de", "id": "ausschreibung", "label": "Ausschreibung" } ],
  "publisher": { "name": "Österreichische Post AG", "type": "Organization" },
  "catalog_record": { "modified": "2025-11-20T09:15:33Z", "issued": "2025-11-20T09:15:33Z" },
  "distributions": [
    {
      "title": { "en": "Kerndatenquelle" },
      "access_url": [ "https://www.post.at/kdq/KDQ.xml" ],
      "format": { "id": "XML" },
      "license": { "resource": "https://creativecommons.org/licenses/by/4.0/" },
      "issued": "2019-06-06T09:23:53.336222Z",
      "modified": "2025-08-28T15:37:27.372810Z"
    }
  ],
  "resource": "https://www.data.gv.at/katalog/datasets/39e2a593-d508-4f64-8a04-8d6dbf730f9d",
  "catalog": { "id": "legacy" }
}
```

Für uns relevante Felder:

| Pfad | Verwendung |
|---|---|
| `id` | stabile ID des Metadatensatzes |
| `publisher.name` | veröffentlichende Stelle (Auftraggeber oder Plattform) |
| `distributions[].title.en == "Kerndatenquelle"` | Erkennungsmerkmal |
| `distributions[].access_url[0]` | URL der Kerndatenquelle (XML) |
| `distributions[].license.resource` | Lizenz, bisher immer CC BY 4.0 |
| `distributions[].modified` | Hinweis, ob sich die Quelle geändert hat |

Weitere Kerndatenquellen aus den ersten 5 Treffern:

| Veröffentlichende Stelle | Kerndatenquelle |
|---|---|
| Energie Burgenland AG | `https://vergabe.energieburgenland.at/NetServer/kerndaten` |
| Österreichische Post AG | `https://www.post.at/kdq/KDQ.xml` |
| vergabeplattform – Mandacis GmbH | `https://vergabeplattform.at/open-data/data-gv/core` |
| ABH Generalplanung GmbH | `https://www.abh.at/assets/bilder/ausschreibungen/Kerndatenquelle.xml` |

## Auffälligkeiten

- **Nicht jeder Treffer ist ein Metadatensatz nach BVergG.** Beispiel: „Video Content Management System für die Lehre“ der PH Wien ist eine einzelne Bekanntgabe, die direkt als Datensatz angelegt wurde, ohne Kerndatenquelle. Deshalb nur Datensätze nehmen, die eine Distribution `Kerndatenquelle` haben. Zusätzlich lässt sich am Titelanfang „Metadatensatz zur Kerndaten-Sammlung“ prüfen.
- **Vergabeplattformen bündeln.** Plattformen wie Mandacis oder ANKÖ treten als eine veröffentlichende Stelle mit einer Kerndatenquelle für viele Auftraggeber auf. 57 Datensätze bedeuten also deutlich mehr als 57 Auftraggeber.
- **Die Hosts sind sehr unterschiedlich**, von Plattform-Servern bis zu statischen XML-Dateien auf Firmen-Websites. Das Rate-Limit muss deshalb pro Host gelten, und wir müssen mit langsamen oder toten Quellen rechnen.
- **Nicht alle Treffer stammen aus dem Katalog `legacy`.** Auch `it-kommunal`, `l7-elak` (Land Tirol) und `harvest-bmlv` kommen vor. Der Katalog taugt nicht als Filter.

## Offene Fragen

- [ ] **Vollständigkeit:** Sind 57 Datensätze wirklich alle? Gegenprobe: Suche nach dem Titel „Metadatensatz zur Kerndaten-Sammlung“ ohne Tag-Filter und die Trefferzahlen vergleichen.
- [ ] **Aufbau einer Kerndatenquelle:** noch nicht abgerufen. Nächster Schritt ist eine einzelne, kleine Kerndatenquelle (z. B. Post oder ABH), um die XML-Liste und `lastmod` zu sehen.
- [ ] **eForms ab 01.10.2026:** Ob die Kerndatenquellen nach der Umstellung gleich aussehen und nur auf eForms-Dokumente verweisen, lässt sich erst ab Oktober an echten Daten prüfen.
- [ ] **Paginierung** (`page`) ist nicht getestet.
