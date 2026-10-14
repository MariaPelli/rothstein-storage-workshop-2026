[Aufgabenstellung](README.md) · [Microblogging-Beispiele](../../demos/microblogging/README.md) · [Musterlösung](task_sample_solution.ipynb) · [Aktenleseführer](../../docs/AKTENLESEFUEHRER.md)

# Walkthrough 3: Was bedeutet eine Verbindung?

## Einstieg: Von der Analyse zur weiteren Recherche

Beginnt mit D077. Die Akte ist eine datierte Analyse zu einem Teilkomplex der Western-USA-Berichte. D079 und D080 sind Zeugenschilderungen. Der Graph soll die Recherchewege zwischen katalogisierten Unterlagen nachvollziehbar machen. Er bewertet keine Zeugenaussagen.

Die Musterlösung startet eigenständig in `uap_solution`; die Aufgabenbearbeitung in `uap_task` bleibt erhalten. Beide verwenden die vorbereiteten JSONL-Dateien, keine vorherige Datenbankaufgabe.

Für den Wechsel zum Browser stehen in der [Musterlösung](task_sample_solution.ipynb) kurze Cypher-Blöcke direkt unter den Notebook-Ergebnissen. Nach der Vorbereitung in 0, A1 und A2 lassen sich dieselben Anzahlen, Verweisketten und Belege dort direkt abfragen. `RETURN p` macht die Pfade interaktiv sichtbar.

## A: Identität, Richtung und Beleg

Der Grundimport erzeugt 764 Knoten und 1'147 Beziehungen. Die 336 Portalverweise kommen in A2 hinzu. Technische IDs identifizieren Knoten; insbesondere das doppelte Kürzel `FBI-UAP-D014` darf nicht zu einem Knoten zusammenfallen.

In A2 lauten die Ergänzungen **`PORTAL_PAIRS_WITH`** und **`->`**:

```cypher
UNWIND $rows AS row
MATCH (a:StorageNode:StorageCatalogEntry {scope:$scope, node_id:row.source})
MATCH (b:StorageNode:StorageCatalogEntry {scope:$scope, node_id:row.target})
MERGE (a)-[r:PORTAL_PAIRS_WITH {scope:$scope, edge_id:row.edge_id}]->(b)
SET r += row.properties
RETURN count(r) AS processed
```

Die Richtung folgt dem originalen Verweis: Quelleintrag → aufgelöster Zieleintrag. Sie ist keine Reihenfolge realer Ereignisse. Im Bestand vorhandene Gegenverweise werden als separate gerichtete Kanten gespeichert. Der Import erzeugt keine zusätzlichen Gegenverweise.

`MERGE` enthält die technische Kanten-ID. Zusammen mit den gebundenen Endpunkten ermöglicht dies den wiederholten Import derselben Beziehung. Die vorbereitenden Prüfungen sichern die Identitäten im Eingabestand. Ein `MERGE` allein wäre keine allgemeine Garantie für beliebige parallele Importe.

Die Properties `source_field`, `raw_reference` und `resolution_rule` dokumentieren, weshalb eine Verknüpfung entstanden ist. Über `source_entry_id`, den Katalogeintrag und dessen `source_row` lässt sich die Angabe zum Rohsnapshot zurückverfolgen. `source_row` ist die vom Ingestion-Code festgelegte Quellzeilenkennung, keine PDF-Seitennummer.

13 nicht eindeutig aufgelöste Pairing-Token bleiben im Qualitätsbericht dokumentiert. Sie sind keine geratenen Kanten. Nach A2 stehen insgesamt 1'483 Beziehungen im Bereich.

## B: Genau zwei Schritte ab D077

Das gesuchte Muster ist `[:PORTAL_PAIRS_WITH*2]`. Die Musterlösung filtert Rückwege zum Start und begrenzt alle Knoten und Kanten auf denselben Kursbereich. Sie liefert geordnete IDs und die IDs der beiden Belegkanten.

| Referenz | Ergebnis |
| --- | ---: |
| Gerichtete Pfade mit genau zwei Schritten ab D077, ohne Rückweg zum Start | 17 |
| Unterschiedliche Zielkatalogeinträge | 17 |
| Zwischenknoten aller 17 Pfade | Western US Event |

Beispiel: **DOW-UAP-D077 → Western US Event → DOW-UAP-D080**.

Die erste Kante verweist im Feld `PDF Pairing` von D077 auf «Western US Event». Die zweite verweist im Feld `Video Pairing` dieses Zwischenknotens auf D080. Bewahrt die Originalfeldnamen auch dann auf, wenn deren Formulierung nicht der erwarteten Medienart entspricht; ihre Bedeutung wird nicht stillschweigend normalisiert.

Die Graphansicht zeigt vier ausgewählte Pfade; die Tabelle enthält alle Ergebnisse. Die Knotenbezeichnung «Western US Event» macht diesen Knoten nicht zu einem quellenübergreifend bestätigten Ereignis. Er bleibt ein Eintrag im Portal.

### Fachliche Interpretation

Zulässig: «Der fixierte Portal-Snapshot enthält eine gerichtete Verweiskette von D077 über den genannten Katalogeintrag zu D080.»

Nicht aus diesem Pfad ableitbar: Alle verknüpften Unterlagen beschreiben dasselbe Ereignis, sind unabhängige Bestätigungen oder zeigen einen aussergewöhnlichen Ursprung.

Lest D080, Seite 5: Die anschliessenden Abbildungen werden als nachträglich KI-generierte Illustrationen eingeordnet. Ein Portalverweis ändert diesen Status nicht. Die Inhaltsprüfung mit PDF-Seitenbeleg ergänzt die Metadatenprüfung. Es werden daraus keine automatisch erschlossenen Wahrheitslabels oder zusätzlichen Ereigniskanten erzeugt.

## C: Mehrere Wege zu einem Ziel

Ab D080 liefert dieselbe Abfrage **24 Pfade zu 18 unterschiedlichen Zielen**. «Western US Event» ist über mehrere Zwischenakten erreichbar. Eine Ergebniszeile ist deshalb eine Verweiskette, keine neue Zielakte und kein zusätzliches Ereignis.

Bei der Frage «Wie viele Zielakten?» zählen wir unterschiedliche Ziel-IDs. Bei der Frage «Über welche Wege?» erhalten wir die unterschiedlichen Pfade und ihre Belege. Beide Ergebnisse sind sinnvoll, beantworten aber verschiedene Fragen.

## Modellvergleich

Die Stärke des Graphmodells liegt hier darin, typisierte Verbindungen, Richtungen und Pfadmuster direkt auszudrücken. Eine relationale Datenbank kann dieselben Beziehungen mit Tabellen und rekursiven Abfragen darstellen. Der Lehrbestand erlaubt keine allgemeine Leistungsrangliste.

Diskutiert: Welche Beziehung sollte im führenden Katalog gepflegt werden? Welche Graphsicht könnte daraus abgeleitet werden? Welche zusätzliche Prüfung wäre nötig, bevor die Redaktion Ereignisse zusammenfasst?

## Hinweise zu typischen Fehlern

| Beobachtung | Erklärung |
| --- | --- |
| Nach erneutem Grundimport keine Pfade | A1 entfernt im gewählten Bereich auch die später ergänzten Pairing-Kanten. A2 erneut ausführen. |
| Falsche Richtung wird abgewiesen | Die Prüftransaktion vergleicht Endpunkte und Belegproperties vor dem Commit. |
| Mehr als 17 Ergebnisse ab D077 | Länge, Richtung, Rückwege und Mehrfachkanten prüfen. `*1..2` enthält auch direkte Nachbarn. |
| Gleiches Kürzel erscheint mehrfach | Kürzel und technische Knoten-ID unterscheiden. |
| Notebook bleibt bei OFFEN | Markierte Stellen wurden noch nicht ergänzt; das ist der vorgesehene Aufgabenstand. |

Die technische Abschlussmeldung ersetzt die gemeinsame Beurteilung der Quelleninterpretation nicht.
