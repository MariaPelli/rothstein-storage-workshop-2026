[Aufgabenstellung](README.md) · [Microblogging-Beispiele](../../demos/microblogging/README.md) · [Aufgabe](task.ipynb) · [Musterlösung](task_sample_solution.ipynb) · [Workshop-Übersicht](../../README.md)

# Walkthrough 1: Ein sauberer Katalog ist noch keine Ereignisstatistik

Die Leitfrage lautet: **Welche Zahl können wir veröffentlichen, und was zählt sie?** Die Musterlösung beginnt eigenständig bei den CSV-Dateien. Sie speichert in `data/work/uap/storage_sample_solution.sqlite`; die Aufgaben-Datenbank und das Aufgaben-Notebook bleiben erhalten.

## 1. Zuerst die Einheit klären

`catalog_entries` enthält 375 Einträge aus fünf Veröffentlichungsrunden. `entry_id` identifiziert einen technischen Eintrag. `source_key` ist eine Quellbezeichnung und darf mehrfach vorkommen.

| Konkreter Stolperstein | Beobachtung im Snapshot | Konsequenz |
| --- | --- | --- |
| `FBI-UAP-D014` | Ein Eintrag zur Illustration der Western-USA-Schilderung und ein später veröffentlichter Eintrag zu historischer Korrespondenz | Beide technischen IDs erhalten; keine UNIQUE-Regel auf das Kürzel |
| Eintrag und Asset | Ein Eintrag kann mehrere Assets verlinken; mehrere Einträge können dasselbe Asset referenzieren | n:m-Zuordnung mit Zwischentabelle |
| Datum | Veröffentlichungsdatum und `incident_date_raw` beschreiben unterschiedliche Aspekte | Veröffentlichungsdatum aus `releases` lesen; Rohangaben erhalten |
| Originalinhalt | Eine Datei kann mehrere Episoden und nachträgliche Bilder enthalten | Akten-, Datei- und Ereigniszahl nicht gleichsetzen |

Die ID-Ableitung und die Grenzen des festen Snapshots sind im [Datenwörterbuch](../../data/README.md) und im [Ingestion-Code](../../ingestion/README.md) dokumentiert. Ein technischer Hash ist kein Nachweis der Identität eines realen Ereignisses.

**Moderationsfrage:** «Was verlieren wir, wenn wir alle Zeilen mit demselben Aktenkürzel zusammenfassen?»

## 2. Die fehlende Beziehung ergänzen

```sql
CREATE TABLE IF NOT EXISTS entry_assets (
    entry_id TEXT NOT NULL REFERENCES catalog_entries(entry_id),
    asset_id TEXT NOT NULL REFERENCES assets(asset_id),
    source_field TEXT NOT NULL,
    PRIMARY KEY (entry_id, asset_id)
) STRICT;
```

Der zusammengesetzte Schlüssel verhindert eine wiederholte identische Zuordnung. Ein alleiniger Schlüssel auf `entry_id` würde mehrere Assets pro Eintrag ausschliessen; ein alleiniger Schlüssel auf `asset_id` würde gemeinsame Referenzen ausschliessen.

Die Bedeutung des Schlüssels gilt für den vorbereiteten Zuordnungsbestand. Sollte eine künftige Anwendung mehrere getrennte Herkunftsnachweise für dieselbe Eintrag–Asset-Kombination speichern müssen, würde sie dafür eine zusätzliche Herkunftstabelle oder ein entsprechend erweitertes Modell benötigen.

Die Stammtabellen und ihre Constraints stehen in [uap_base.sql](../../schemas/sqlite/uap_base.sql). `release_date` ist im SQL-Modell bewusst nur in `releases` gespeichert; der Import prüft die redundanten CSV-Werte, bevor er diese Spalte beim Laden des Katalogs weglässt.

## 3. Was die technische Probe tatsächlich zeigt

Der Import löscht die Inhalte der vorgesehenen Tabellen in umgekehrter Abhängigkeitsreihenfolge und lädt sie in Eltern-vor-Kind-Reihenfolge neu. Beide Operationen gehören zu **derselben Transaktion**. Erst nach erfolgreichem Laden wird bestätigt. Beim erneuten Import wächst der Bestand nicht.

| Tabelle | Zeilen nach jedem vollständigen Import |
| --- | ---: |
| agencies | 10 |
| releases | 5 |
| catalog_entries | 375 |
| assets | 374 |
| entry_assets | 397 |
| portal_pairings | 336 |

Die Probe kontrolliert beide Fremdschlüssel und die wiederholte Zuordnung. Jeder Versuch schreibt zuerst eine gültige technische Probe-Stelle. Der zweite Schreibzugriff scheitert an einem Constraint; das explizite `ROLLBACK` nimmt auch den ersten Zugriff zurück. Der Code rollt selbst bei einem unvollständigen Schema zurück und meldet danach den fehlenden Constraint.

Das demonstriert **Atomarität** und **Konsistenz im Sinne definierter Integritätsregeln**. Es ist kein Test aller Isolationssituationen oder eines Stromausfalls. Die Anwendung übernimmt den Rollback der gesamten Probe; aus dem Fehler eines einzelnen SQL-Statements folgt nicht automatisch der Rollback aller vorherigen Statements.

SQLite-Fremdschlüssel sind verbindungsspezifisch aktiviert. `STRICT` begrenzt die zulässigen Speichertypen; ISO-Text in einer Datumsspalte erhält dadurch noch keine vollständige Kalenderprüfung. [SQLite: Fremdschlüssel](https://www.sqlite.org/foreignkeys.html), [STRICT-Tabellen](https://www.sqlite.org/stricttables.html).

## 4. Die drei Western-USA-Akten recherchierbar machen

Die Musterlösung verbindet Katalog, Stelle und Veröffentlichungsrunde und ergänzt die Assets per `LEFT JOIN`. Sie gruppiert nach `entry_id` und den ausgegebenen Attributen.

| Aktenkürzel | Rolle im Rechercheauftrag | Veröffentlichung | Asset-Verweise |
| --- | --- | --- | ---: |
| DOW-UAP-D077 | AARO-Analyse eines Teilaspekts | 12. Juni 2026 | 1 |
| DOW-UAP-D079 | Narrative Statement 1 | 12. Juni 2026 | 1 |
| DOW-UAP-D080 | Narrative Statement 2 | 12. Juni 2026 | 1 |

Alle drei haben im Katalog die Stelle `Department of War`. Diese Katalogzuordnung ersetzt keine differenzierte Aussage darüber, wer den jeweiligen Inhalt verfasst oder geschildert hat.

**Gemeinsam am Original besprechen:**

- [D079, S. 1–2](../../data/raw/originals/DOW-UAP-D079.pdf): ungewöhnliche Lichter und scheinbar schwebende Objekte als subjektive Zeugenschilderung.
- [D080, S. 5](../../data/raw/originals/DOW-UAP-D080.pdf): Der Bericht kennzeichnet die Abbildungen auf S. 5–8 als nachträglich KI-generierte Illustrationen. Sie sind keine Fotografien des beschriebenen Ereignisses.
- [D077, S. 1–4](../../data/raw/originals/DOW-UAP-D077.pdf): Die AARO-Bewertung betrifft den Teilaspekt der Kugeln, aus denen weitere Lichter zu kommen scheinen. Sie ist keine pauschale Erklärung sämtlicher Episoden und Objekte aus beiden Berichten.

**Moderationsfrage:** «Welche der drei SQL-Zeilen ist eine unabhängige Bestätigung eines ausserirdischen Objekts?» Aus der Existenz der Einträge folgt eine solche Bestätigung nicht. Der [Aktenleseführer](../../docs/AKTENLESEFUEHRER.md) liefert weitere Seitenbelege und Recherchespuren.

## 5. Drei verschiedene Zählungen

```sql
SELECT COUNT(DISTINCT e.entry_id) AS catalog_entries,
       COUNT(ea.asset_id) AS entry_asset_links,
       COUNT(DISTINCT ea.asset_id) AS distinct_asset_references
FROM catalog_entries e
LEFT JOIN entry_assets ea ON ea.entry_id = e.entry_id;
```

| Ergebnis | Wert | Bedeutung |
| --- | ---: | --- |
| catalog_entries | 375 | Technische Katalogeinträge |
| entry_asset_links | 397 | Vorhandene Eintrag–Asset-Zuordnungen |
| distinct_asset_references | 374 | Unterschiedliche referenzierte Asset-IDs |

Im vorhandenen Bestand besitzt jeder Eintrag mindestens einen Verweis. Der `LEFT JOIN` erhält jedoch auch einen zukünftigen Eintrag ohne Asset. `COUNT(ea.asset_id)` zählt dann null Verweise; `COUNT(*)` zählt weiterhin die erhaltene Zeile. Nach dem aktuellen Join ergeben `COUNT(*)` und `COUNT(ea.asset_id)` zufällig beide 397. Keiner der beiden Ausdrücke zählt damit Katalogeinträge.

`DISTINCT` ist keine allgemeine Reparatur für einen falschen Join. Entscheidend ist, **welche ID** die gewünschte Einheit bezeichnet. Auch die 374 Assets sind keine vollständige Deduplizierung sämtlicher Medien nach Dateiinhalten: Ihre Identität folgt normalisierten Referenzen.

**Möglicher Veröffentlichungstext:** «Der Snapshot umfasst 375 Katalogeinträge mit 397 Zuordnungen zu 374 unterschiedlichen referenzierten Assets. Diese Bestandszahlen beschreiben keine Anzahl verifizierter Sichtungen oder unabhängiger Bestätigungen.»

## 6. Vertiefung: Stelle und Index

Für `Department of War` ergeben sich **190 Einträge, 211 Zuordnungen und 189 unterschiedliche referenzierte Assets**. Die beiden Richtungen der n:m-Beziehung sind damit im selben Teilbestand sichtbar. Summiere unterschiedliche Assets je Stelle nicht ohne Prüfung zu einer globalen Distinct-Zahl: Ein Asset könnte stellenübergreifend vorkommen.

```sql
CREATE INDEX IF NOT EXISTS idx_catalog_source_key
ON catalog_entries(source_key);
```

Der Index ist absichtlich nicht eindeutig. Beide Einträge zu `FBI-UAP-D014` müssen erhalten bleiben. Ein `UNIQUE`-Index wäre kein Reparaturmechanismus für die Quelle.

Die Ausgabe von `EXPLAIN QUERY PLAN` kann zwischen Versionen variieren. Erwartbar ist ein Zugriff über den Index auf `source_key`; die genaue Textzeile ist keine stabile Testschnittstelle. Bei erneutem Notebook-Durchlauf existiert der Index bereits vor dem gezeigten CREATE-Statement. Ein echter Vergleich ohne Index benötigt einen frischen Arbeitsstand. [SQLite: Abfrageplan](https://www.sqlite.org/eqp.html).

## 7. Häufige Fehler und Rückfragen

| Symptom oder Aussage | Ursache oder sinnvolle Rückfrage |
| --- | --- |
| Ungültiger Verweis wird akzeptiert | Beide FOREIGN KEY-Regeln und `PRAGMA foreign_keys` auf dieser Verbindung prüfen |
| Neue DDL-Zelle ändert das alte Schema nicht | `IF NOT EXISTS` ist keine Migration; eigenen Arbeitsstand wie in der README umbenennen und neu aufbauen |
| Mehr Zeilen nach erneutem Import | Append ohne Wiederholungsstrategie; Importgrenze und Schlüssel prüfen |
| «397 UFO-Fälle» | Eintrag–Asset-Zuordnungen wurden als Ereignisse bezeichnet |
| «Alle Bilder belegen die Beobachtung» | Herkunft und nachträgliche Illustration am Original prüfen |
| «ACID garantiert wahre Inhalte» | Technische Integritätsregeln und externe Sachverhaltsprüfung trennen |
| «Der Index macht jedes System schneller» | Konkreten Zugriff, Schreibaufwand, Datenmenge und gemessene Leistung unterscheiden |

## 8. Modellvergleich festhalten

SQL eignet sich hier für kontrollierte Beziehungen, nachvollziehbare Joins und reproduzierbare Bestandszählungen. Die PDFs liegen als Originaldateien ausserhalb der SQLite-Datei; die Datenbank speichert ihre Metadaten und Referenzen.

Als Transferfrage kann die Gruppe eine Aktenansicht mit unterschiedlich aufgebauten Zusatzfeldern oder eine Suche über mehrere Portalverweise formulieren. Diese Anforderungen greifen die folgenden MongoDB- und Neo4j-Pakete auf. Ein Wechsel des Speichermodells ersetzt dabei keine Prüfung von Quellen und Zähleinheiten.
