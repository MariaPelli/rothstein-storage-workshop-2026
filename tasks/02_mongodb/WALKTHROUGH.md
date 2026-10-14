[Aufgabenstellung](README.md) · [Microblogging-Beispiele](../../demos/microblogging/README.md) · [Aufgabe](task.ipynb) · [Musterlösung](task_sample_solution.ipynb) · [Workshop-Übersicht](../../README.md)

# Walkthrough 2: Ein Dokument, unterschiedliche Arten von Information

Der Kern dieses Workshops ist die Trennung zwischen **Quellkatalog, Originaldatei und eigener Lesernotiz**. MongoDB erlaubt eine inhaltsnahe Darstellung. Diese Flexibilität nimmt uns die Entscheidung über Herkunft, Schlüssel und Zähleinheiten nicht ab.

## 1. Das vorbereitete Dokumentmodell lesen

| Element | Umsetzung | Begründung und Konsequenz |
| --- | --- | --- |
| Technischer Eintrag | `_id = entry_id` | Stabiler Import; die doppelte Quellkennung FBI-UAP-D014 bleibt erhalten |
| Katalogstelle | Eingebettetes `agency` plus erhaltene `agency_id` | Gemeinsam lesbare Snapshot-Angabe; eine spätere Namenskorrektur muss die Kopien berücksichtigen |
| Asset-Metadaten | Array `assets` | Verweise einer Akte gemeinsam abrufen; gleiche Assets können in mehreren Aktenansichten erscheinen |
| Original-PDF | Datei ausserhalb der Collection, per Asset-ID/Locator referenziert | Originalbytes werden nicht in jedem Katalogdokument dupliziert |
| Originale Zusatzfelder | Objekt `source_metadata` | Unterschiedlich vorhandene Quellfelder bleiben sichtbar |
| Aufgelöste Portalbezüge | Array `related_entries` mit technischen Ziel-IDs | Zielakte bleibt ein eigenes Dokument; kein rekursives Kopieren ihrer Inhalte |
| Eigener Befund | Dokument in `reading_notes` | Eigener Bearbeitungsstand mit Herkunftskennzeichnung und Seitenbeleg |

Die Aufbereitung bettet bewusst einige Informationen ein. Das ist ein gewähltes Lesemodell für diesen festen Snapshot, keine automatische Normalform für jede Anwendung. [MongoDB: Datenmodellierung](https://www.mongodb.com/docs/manual/data-modeling/).

**Moderationsfrage:** «Welche Information würdest Du bei häufigen Änderungen lieber zentral referenzieren, und welche Daten liest die Redaktion tatsächlich gemeinsam?»

## 2. Die Lesernotiz an D080 belegen

[DOW-UAP-D080, Seite 5](../../data/raw/originals/DOW-UAP-D080.pdf) kennzeichnet die Abbildungen auf den Seiten 5–8 als nachträglich KI-generierte Illustrationen. Sie sind keine Fotografien des geschilderten Ereignisses. Die Notiz hält diesen konkreten Befund fest; sie bewertet nicht pauschal die Wahrheit aller Zeugenschilderungen.

Die Musterlösung bettet `finding`, `statement` und `evidence` unter `review` ein. `evidence` enthält die technische Asset-ID und die Liste `[5]`. Die Katalog-ID bleibt eine Referenz. Unter `origin` steht ausdrücklich `workshop_manual_review`.

`replace_one(..., upsert=True)` speichert wiederholt unter `note:<entry_id>`. So bleibt eine Notiz pro Eintrag und Arbeitsbereich. Der erneute Katalogimport berührt die separate Notiz-Collection nicht. Eine fiktive zweite Leserin oder eine Änderungshistorie wäre eine andere Anforderung und müsste in den Schlüssel aufgenommen werden.

**Moderationsfrage:** «Welche Aussage stammt aus dem Portal, welche aus unserer Lektüre, und wie findet eine andere Person den Beleg wieder?»

## 3. Vorhandene und fehlende Quellfelder

```python
pairing_filter = {"source_metadata.PDF Pairing": {"$exists": True}}
```

Im gelieferten Snapshot besitzen **111** Einträge dieses Feld; bei **264** fehlt es. Alle oberen Standardfelder der vorbereiteten Katalogdokumente sind vorhanden. Die Heterogenität liegt hier vor allem im verschachtelten Objekt `source_metadata`; ein `$exists` auf ein immer angelegtes Standardfeld wäre deshalb kein sinnvoller Vergleich.

`$exists: True` akzeptiert grundsätzlich auch einen vorhandenen Wert `null`. Unsere Ingestion übernimmt in `source_metadata` nur nicht leere Originalfelder. Diese Aufbereitungsregel erklärt, warum die 111 Treffer hier tatsächlich gefüllte Quellangaben besitzen. [MongoDB: $exists](https://www.mongodb.com/docs/manual/reference/operator/query/exists/).

Ein fehlendes PDF-Pairing-Feld bedeutet nur, dass diese konkrete Katalogangabe fehlt. Es beweist nicht, dass es keinen Zusammenhang zwischen Dokumenten gibt.

## 4. Die richtige Einheit aggregieren

```python
pipeline_media = [
    {"$match": pairing_filter},
    {"$group": {"_id": "$media_type", "catalog_entries": {"$sum": 1}}},
    {"$project": {"_id": 0, "media_type": "$_id", "catalog_entries": 1}},
    {"$sort": {"catalog_entries": -1, "media_type": 1}}
]
```

| Medienart laut Katalog | Einträge mit PDF-Pairing-Feld |
| --- | ---: |
| PDF | 62 |
| VID | 25 |
| IMG | 24 |
| **Gesamt** | **111** |

AUD erscheint in dieser gefilterten Gruppierung nicht. Der ganze Snapshot enthält trotzdem 15 Audioeinträge. Die gefilterte Teilmenge ist nicht mit der Gesamtverteilung von 211 PDF-, 119 Video-, 30 Bild- und 15 Audioeinträgen gleichzusetzen.

Wir zählen eine Einheit pro Katalogdokument. Ein `$unwind` über `assets` oder `related_entries` würde die Pipeline-Zeilen vervielfachen und damit eine andere Zählung erzeugen.

**Moderationsfrage:** «Warum enthalten 25 Videoeinträge ein Feld namens PDF Pairing?» Der Name bezeichnet die Art des Portalverweises, nicht zwingend die Medienart des Eintrags selbst.

## 5. Vertiefung: dieselbe Array-Position

```python
pdf_asset_filter = {
    "assets": {"$elemMatch": {"locator_type": "url", "format_hint": "pdf"}}
}
```

Die Abfrage findet **223 Katalogeinträge** mit einem entsprechenden PDF-URL-Verweis: 211 als PDF und zwölf als VID katalogisierte Einträge. Damit wird die Unterscheidung zwischen Eintrag und verlinktem Asset konkret sichtbar. `format_hint` stammt aus der Aufbereitung des Verweises; die Abfrage ist keine erneute Prüfung aller Originaldateien.

Zwei getrennte Bedingungen auf `assets.locator_type` und `assets.format_hint` könnten allgemein durch unterschiedliche Array-Elemente erfüllt werden. `$elemMatch` verlangt beide Bedingungen am selben Element. Der Serverprüflauf enthält dazu ein kontrolliertes Gegenbeispiel, das nicht zum Quelldatensatz gehört. [MongoDB: $elemMatch](https://www.mongodb.com/docs/manual/reference/operator/query/elemMatch/).

## 6. Flexibilität und Schemavalidierung

Der [Validator](../../schemas/mongodb/reading_note_validator.json) verlangt IDs, Herkunft und eine strukturierte Notiz. `pdf_pages` muss eine nicht leere Liste positiver Ganzzahlen sein. Zusätzliche Felder bleiben erlaubt; die Regel legt also einen Kern fest, ohne jede mögliche Notizstruktur vorwegzunehmen.

Die Probe speichert zuerst eine gültige Notiz und versucht danach eine Zeichenkette `"5"` als Seitenliste. MongoDB soll diese Änderung mit einer Validierungsverletzung zurückweisen. Anschliessend wird ein formal korrektes Notizdokument mit nicht existierender Katalog-ID akzeptiert: Typprüfung ist keine automatische Fremdschlüsselprüfung. Die Probe-Collection wird gezielt wieder entfernt.

Der eigentliche Speichercode prüft die Katalog- und Asset-Referenzen vorher in Python. Bei konkurrierenden Änderungen zwischen Prüfung und Schreiben wäre diese Vorabprüfung allein nicht ausreichend. Der Workshop arbeitet mit einem festen Katalog und getrennten persönlichen Arbeitsbereichen. [MongoDB: Schemavalidierung](https://www.mongodb.com/docs/manual/core/schema-validation/).

## 7. Import und Atomarität richtig einordnen

Einzelne Dokumentänderungen sind atomar. Der hier verwendete Bulk-Import über viele Dokumente und Collections ist insgesamt nicht atomar. Bei einer Serververletzung kann deshalb ein früherer Schreibzugriff bereits übernommen sein. Ein erneuter Import unter stabilen IDs repariert den Bestand, sobald er vollständig erfolgreich ist. [MongoDB: Atomarität](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/).

Dies ist ein bewusst sichtbarer Unterschied zum SQLite-Snapshot-Import mit einer gemeinsamen Transaktion. Es ist keine allgemeine Aussage, dass MongoDB keine Mehrdokument-Transaktionen unterstützt; dafür wäre eine geeignete Replica-Set- oder Sharding-Konfiguration nötig. Unser dokumentierter Einzelserver bleibt unverändert.

## 8. Typische Fehler

| Beobachtung | Klärung |
| --- | --- |
| Nach erneutem Laden mehr Dokumente | Stabile `_id` und Upsert-Filter prüfen; ein neuer zufälliger ObjectId pro Import verdoppelt den Bestand |
| Änderungen an Notizen verschwinden | Originalkatalog und eigene Notizen wurden gemeinsam ersetzt; getrennte Collections verwenden |
| Alle Einträge besitzen das gesuchte Feld | Wurde ein standardisiertes oberes Feld statt eines optionalen Originalfelds geprüft? |
| «Keine Pairing-Angabe = kein Zusammenhang» | Fehlende Metadaten erlauben diese Schlussfolgerung nicht |
| «Eine PDF-Akte enthält nur Text» | D080 am Original lesen; Medienart ist keine vollständige Inhaltsbeschreibung |
| «Schema valid = Referenz existiert» | Typ- und Existenzprüfung unterscheiden |
| «Der Bulk-Import wurde komplett zurückgerollt» | Auf dem Einzelserver gelten die beschriebenen Dokumentgrenzen |

Die vollständige Referenzimplementierung steht im separaten Musterlösungs-Notebook.
