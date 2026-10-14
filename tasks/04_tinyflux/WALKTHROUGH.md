[Aufgabenstellung](README.md) · [Microblogging-Beispiele](../../demos/microblogging/README.md) · [Musterlösung](task_sample_solution.ipynb) · [Aktenleseführer](../../docs/AKTENLESEFUEHRER.md)

# Walkthrough 4: Was zählt eine Zeitreihe?

## Einstieg: Eine Zahl für die Redaktion

Die Frage nach einem zeitlichen Verlauf verlangt zuerst eine Definition: Welche Einheit zählt, welche Datumsgrundlage gilt und welcher Bestand ist abgedeckt? Unsere Reihe zählt geeignete Katalogeinträge je Katalogstelle und Katalog-Ereignisjahr. Die Musterlösung beginnt unabhängig aus der Jahres-CSV und schreibt eine eigene Datei.

## A: Zeit, Serie und numerischer Wert

Die Ergänzungen lauten:

```python
time = parse_time(row["period_start"])
tags = {
    "agency_id": row["agency_id"],
    "date_basis": row["date_basis"],
    "snapshot_sha256": row["snapshot_sha256"],
}
fields = {"catalog_entries": int(row["catalog_entries"])}
```

Zeit ist ein Bestandteil des Punkts. Die Tags beschreiben die Serie und ihren Ursprung, das Field enthält die Zählgrösse. CSV-Werte kommen als Text an; das Zählfeld wird deshalb bewusst in eine Zahl umgewandelt. TinyFlux 1.2.0 liest numerische Felder aus seiner CSV-Ablage als Gleitkommazahlen zurück. `25.0` und `25` repräsentieren hier dieselbe ganzzahlige Zählgrösse; die Tabellenausgabe verwendet Ganzzahlen.

Die zehn Katalogstellen besitzen denselben Jahresbeginn. Ein Zeitstempel allein ist somit kein Schlüssel. Der Import prüft für die Jahresreihe die Kombination aus Measurement, Zeit und Tags auf Duplikate. Das ist eine Regel des Kursimports, kein vom Speichersystem erzwungener Primärschlüssel.

Nach dem Import stehen 820 Punkte für 82 Jahre × 10 Stellen bereit. Die Summe ihrer Zählfelder beträgt 292. Eine Importwiederholung behält diese Werte bei, weil der Kursimport die ausgewählte Datei nach Prüfung ersetzt. TinyFlux bietet auch Änderungs- und Löschoperationen; das Beispiel setzt keine grundsätzlich unveränderliche Speicherung voraus.

## B: Grenzen und Zähleinheit

```python
annual_query = (
    (Tag.agency_id == fbi_id)
    & (Time >= since)
    & (Time < until)
)
window_total = sum(int(p.fields["catalog_entries"]) for p in window_points)
```

`since` steht für den 1. Januar 2020 und `until` für den 1. Januar 2024, jeweils UTC. Die Abfrage schliesst 2020 ein und 2024 aus. Ein unmittelbar anschliessendes Fenster beginnt bei derselben Obergrenze, ohne einen Randpunkt doppelt zu zählen.

| Kennzahl | Ergebnis |
| --- | ---: |
| Gefundene Jahres-/FBI-Punkte | 4 |
| Summe geeigneter FBI-Katalogeinträge im Fenster 2020–2023 | 25 |
| Vergleich: alle Stellen im selben Fenster | 116 |
| Gesamte Jahresreihe, alle Stellen | 292 |

`len(window_points)` beantwortet die Frage nach Jahrespunkten. Die Redaktion fragt nach Katalogeinträgen und benötigt die Summe des Zählfelds. Der Stellenfilter darf nicht fehlen.

In B3 lautet die y-Achsenbeschriftung **Geeignete Katalogeinträge**. Sie benennt die tatsächliche Zähleinheit. Die Grafik zeigt 2018–2026; das abgefragte Fenster ist orange markiert. Auch Nulljahre bleiben sichtbar. Sie enthält bewusst keine täglichen Beobachtungen: Die Daten haben Jahresgranularität. Die zugrunde liegende FBI-Tabelle umfasst alle 82 Jahre, die Grafik wählt einen lesbaren Ausschnitt.

### Veröffentlichungsaussage

«Im fixierten Katalog-Snapshot liegen 25 geeignete FBI-Katalogeinträge mit eindeutigem Katalog-Ereignisjahr 2020 bis 2023 vor.»

Diese Formulierung benennt Bestand, Stelle, Zähleinheit und Datumsgrundlage. Sie behauptet weder 25 unabhängige reale Ereignisse noch bestätigte Sichtungen. Mehrere Katalogeinträge können verwandte Unterlagen betreffen; Auswahl und Veröffentlichung beeinflussen die sichtbare Verteilung.

### Warum die Originalakten zur Zeitreihe gehören

**FBI-UAP-D024, Seiten 1–3:** Der Bericht schildert mehrere Beobachtungen. Die Katalogangabe `2002, 2023-2024` erlaubt keine eindeutige Jahreszuordnung dieses einen Eintrags. Ihn jedem genannten Jahr zuzuschlagen, würde eine andere Zählregel schaffen und seine Gewichtung verändern. Er bleibt im Katalog und im Ausschlussbericht erhalten, aber ausserhalb dieser Jahresreihe.

**FBI-UAP-D026, Seiten 1–3:** Die berichtete Beobachtung liegt laut Katalog im Jahr 2023; das Interview stammt aus Januar 2026. Eine Auswertung nach Interviewjahr wäre eine andere Kennzahl. In der vorhandenen Reihe verwenden wir das Katalog-Ereignisjahr und erfinden aus der Jahresangabe keinen Tag. Siehe [Aktenleseführer](../../docs/AKTENLESEFUEHRER.md).

## C: Null, Ausschluss und gröbere Intervalle

`Field.catalog_entries > 0` liefert 72 positive Punkte. Die 748 anderen Punkte enthalten explizit Null. Zusammen ergeben sie weiterhin 820 Punkte; die Summe der positiven Werte bleibt 292.

Die 83 ausgeschlossenen Katalogeinträge sind keine 83 Nullpunkte. Für sie fehlt die eindeutige Zuordnung zu einer Jahres-/Stellenkombination. Nullpunkte stammen dagegen aus der vollständigen Kombination aller 82 Jahre mit den zehn vorhandenen Stellen.

Die Dekadensummen entstehen nach dem TinyFlux-Abruf in pandas. Die erste Dekade ist nur ab 1945, die letzte bis zum Snapshot in 2026 abgedeckt. Auch die dazwischen liegenden Dekaden repräsentieren einen ausgewählten Katalog, keine vollständige Ereigniserhebung. Aus unterschiedlichen Summen lässt sich deshalb keine einfache Aussage über reale Ereignisraten ableiten.

## Modellvergleich

Die abgeleitete Reihe unterstützt Zeitfenster und Jahres-/Stellenvergleiche. Akten-IDs, Rohdatierungen, Texte und belegte Verbindungen sind in dieser Aggregation nicht mehr enthalten. Für Nachfragen bleibt der detaillierte Katalog mit Originalquellen nötig.

TinyFlux führt in diesem Paket Zeit-, Tag- und Feldfilter aus. Gruppierung, Summen über Ergebnisse, Verknüpfungen und Diagramme stehen in Python/pandas. Das Microblogging-Beispiel zeigt diese Grenze zusätzlich an der Follow-Pfadsuche. SQL kann dieselben Jahreswerte speichern und filtern; hier wird eine Modellperspektive verglichen, keine allgemeine Leistungsrangliste erstellt.

## Typische Stolperstellen

| Beobachtung | Erklärung |
| --- | --- |
| Ergebnis 4 statt 25 | Punkte gezählt; die Zählfelder wurden nicht summiert. |
| Ergebnis 31 | 2024 wurde zusätzlich eingeschlossen. |
| Ergebnis 116 | Alle Stellen statt nur FBI wurden berücksichtigt. |
| Gleicher Zeitstempel mehrfach | Unterschiedliche Katalogstellen gehören zu unterschiedlichen Serien. |
| 1. Januar erscheint als Beobachtungsdatum | Technische Jahresmarke und echte Datumspräzision verwechselt. |
| Aufgabe meldet OFFEN | Markierte Stellen sind noch nicht ergänzt; vorgesehener Ausgangszustand. |
| Nach Import fehlen eigene Änderungen | Der Neuimport ersetzt die gewählte Arbeitsdatei. Lösung und Aufgabe haben eigene Dateien. |

Die technischen Prüfmeldungen bestätigen Code und Referenzwerte. Die Qualität der Veröffentlichungsaussage und der Quellenbegründung wird in der gemeinsamen Besprechung beurteilt.
