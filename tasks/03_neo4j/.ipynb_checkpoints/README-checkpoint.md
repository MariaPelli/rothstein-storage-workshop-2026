[Workshop-Übersicht](../../README.md) · [Technische Vorbereitung](../../README_technical_preparation.md) · [Microblogging-Beispiele](../../demos/microblogging/README.md) · [SQLite](../01_sqlite/README.md) · [MongoDB](../02_mongodb/README.md) · [Neo4j](README.md) · [TinyFlux](../04_tinyflux/README.md)

---

# Aufgabe 3: Belegte Verbindungen mit Neo4j

Die Forschungsredaktion startet bei der Analyseakte **DOW-UAP-D077**. Welche weiteren Unterlagen erreicht sie über dokumentierte Portalverweise? Was ist mit einem Pfad belegt, und welche Schlussfolgerung würde darüber hinausgehen?

## Deine Dateien

- [task.ipynb](task.ipynb): Deine Bearbeitung;
- [task_sample_solution.ipynb](task_sample_solution.ipynb): eigenständige Musterlösung;
- [WALKTHROUGH.md](WALKTHROUGH.md): gemeinsame Erläuterung;
- [Microblogging-Demo](../../demos/microblogging/03_neo4j.ipynb): Modell und Cypher-Beispiele;
- [Aktenleseführer](../../docs/AKTENLESEFUEHRER.md): Quellenkontext und PDF-Seitenbelege.

## Einstieg

Verwende den eingerichteten Neo4j-Dienst über [Docker/Codespaces](../../README_technical_preparation.md) oder [Neo4j Desktop](../../docs/setup/NEO4J_DESKTOP.md) und den Kernel **Python (rothstein-storage-workshop-2026)**. Neue Pakete und Erweiterungen sind nicht erforderlich; APOC wird nicht verwendet.

Die Graph-Eingaben liegen unter `data/input/graph/`: 764 Knoten und 1'483 gerichtete Beziehungen. Davon repräsentieren 375 Knoten Katalogeinträge, 374 Dateiverweise, zehn Katalogstellen und fünf Veröffentlichungstranchen. Die Aufgabe benötigt keine Ergebnisse vorheriger Workshops.

### Vor der ersten Codezelle: den gewählten Neo4j-Dienst verwenden

**Wähle genau einen Betriebsweg. Eine neue Desktop-Instanz ist nur für Weg B nötig, wenn noch keine passende Instanz vorhanden ist.**

| Weg | Vor dem Notebook | Graphansicht nach dem Import |
| --- | --- | --- |
| **A: lokal mit Docker** | Neo4j-Desktop-Instanzen stoppen; den eingerichteten Neo4j-Container verwenden. | [Neo4j Browser](http://localhost:7474/browser/) mit `bolt://127.0.0.1:7687` |
| **B: lokal ohne Docker** | Einen laufenden Neo4j-Kurscontainer zuerst stoppen; die Desktop-Kursinstanz starten. | Bei derselben Instanz **Open → Neo4j Browser** (1.x) bzw. **Connect → Query** (2.x) |
| **C: Codespaces** | Den automatischen Dienststart abwarten; den Host `neo4j` aus der Containerkonfiguration verwenden. | Über die weitergeleitete Codespaces-Adresse; keine lokale Desktop-Instanz starten. |

**Portkonflikt vermeiden:** Weg A und B belegen lokal standardmässig `7474` (Weboberfläche) und `7687` (Bolt). Betreibe dort nur einen Neo4j-Server. Beim Wechsel zu Weg B im bisherigen Repo-Ordner `docker compose stop neo4j` ausführen, solange noch die bisherige Compose-Konfiguration vorliegt. Andere Neo4j-Container gegebenenfalls in Docker Desktop stoppen. Docker darf für MongoDB weiterlaufen.

Die [Neo4j-Startanleitung für A, B und C](../../docs/setup/NEO4J_START.md) beschreibt Start, Verbindung, Graphansicht und den Wechsel zwischen den Wegen. **Nur für Weg B:** [Desktop einrichten](../../docs/setup/NEO4J_DESKTOP.md). Eine bestehende Kursinstanz weiterverwenden. Anzeigename: **rothstein-storage-workshop-2026**, Benutzer: `neo4j`, Standardpasswort: `Storage-Neo4j-2026`, Datenbank: `neo4j`.

Lokal verwenden die Notebooks bei den Standardwerten `127.0.0.1:7687`. Abweichende Zugangsdaten in einer vorhandenen `.env` anpassen; danach den Kernel neu starten. Codespaces setzt die internen Hosts und Ports automatisch. **Erst nach erfolgreichem Dienststart die Python-Codezellen ausführen.**

Die [Musterlösung](task_sample_solution.ipynb) enthält direkt unter den Ergebnissen kurze, vollständig kopierbare **Browser-Abfragen**. Führe dort 0, A1 und A2 für den gemeinsamen Datenbestand aus und vergleiche danach Tabellen und Pfade in Notebook und Browser. Die Browser-Abfragen der Musterlösung verwenden `uap_solution`; für den Aufgabenbestand ersetze diesen Wert durch `uap_task`.

### Nach dem Import: Graph anzeigen (optional)

Führe in [task.ipynb](task.ipynb) die Codezellen der Abschnitte **0 und A1** bis zur Ausgabe **GRUNDIMPORT OK** aus. Das Notebook hat direkt in die laufende Datenbank geschrieben; Du musst keine Datei übertragen.

1. Die Oberfläche passend zu Deinem Weg öffnen: **A:** [Neo4j Browser](http://localhost:7474/browser/) der laufenden Docker-Datenbank; **B:** bei der laufenden Desktop-Kursinstanz **Open → Neo4j Browser** (1.x) oder **Connect → Query** (2.x); **C:** [Neo4j Browser im Codespace](../../docs/setup/NEO4J_START.md#weg-c-codespaces). Immer dieselbe Instanz wie im Notebook und die Datenbank **neo4j** verwenden. Benutzer **neo4j**, Passwort **`Storage-Neo4j-2026`** beziehungsweise Dein tatsächliches Passwort.
2. Die folgende Abfrage in das **Cypher-Eingabefeld dieser Oberfläche** kopieren, mit dem Play-Schalter ausführen und im Ergebnis **Graph** wählen. Diese Abfrage wird in Neo4j ausgeführt, nicht in einer Python-Codezelle.

```cypher
MATCH p = (:StorageCatalogEntry {scope: 'uap_task', source_key: 'DOW-UAP-D077'})
          -->(:StorageNode {scope: 'uap_task'})
RETURN p LIMIT 25;
```

Du siehst zunächst die vorhandenen Verbindungen ab D077. Nach Abschluss von **A2** dieselbe Abfrage erneut ausführen: Jetzt erscheinen auch die ergänzten Portalverweise. Klicke auf eine Beziehung, um ihre Belegproperties anzusehen. Nach Änderungen im Notebook genügt es, die Abfrage erneut auszuführen.

**Für die Graphansicht keinen zweiten Neo4j-Server starten.** Notebook und Oberfläche greifen auf dieselbe Instanz zu. Daten einer lokalen Instanz und Daten im Codespace sind getrennte Bestände.


## Dein Auftrag

| Teil | Tätigkeit | Ergebnis |
| --- | --- | --- |
| A | Grundgraph importieren; Typ und Richtung der Portalverweise ergänzen | 336 Verweiskanten mit Belegproperties; nachvollziehbare Erklärung ihrer Bedeutung |
| B | Genau zwei gerichtete Schritte ab D077 abfragen; Pfade darstellen und Belege prüfen | Lesbare Pfadtabelle, gezielte Graphansicht und begrenzte Quelleninterpretation |
| C, optional | Dieselbe Abfrage ab D080 ausführen | Pfadanzahl und Anzahl unterschiedlicher Zielakten auseinanderhalten |

Ergänze die markierten Stellen im Aufgaben-Notebook. Noch unbearbeitete Zellen melden **OFFEN**. Die Musterlösung überschreibt Dein Notebook nicht.

## Modellentscheidung

`PORTAL_PAIRS_WITH` bedeutet: Ein Metadatenfeld des Quelleintrags nennt einen Zielverweis, der im Snapshot eindeutig aufgelöst wurde. Die Kante speichert Quelle, Rohverweis und Auflösungsregel. Sie bedeutet weder automatisch «gleiches Ereignis» noch «bestätigte Beobachtung».

Der Zwischenknoten **Western US Event** bleibt ein **Katalogeintrag**. Ein sprechender Titel ist kein Nachweis einer eigenständigen Ereignisidentität. Aktenkürzel können zudem mehrfach vorkommen; für Identität und Import gelten technische IDs.

## Arbeitsbereiche und Wiederholung

Die Datenbank heisst wie bisher `neo4j`. Getrennte Bereiche sind `microblogging_demo`, `uap_task` und `uap_solution`, jeweils im Feld `scope`. Kursspezifische Labels beginnen mit `Storage`.

Der Grundimport ersetzt nur den ausgewählten Bereich in einer Datentransaktion. Wenn Du ihn später erneut ausführst, musst Du auch A2 erneut ausführen. Ein wiederholtes `MERGE` in A2 ergänzt dieselben Kanten ohne Mehrfachzählung. Eigene Änderungen innerhalb des neu importierten Bereichs werden zurückgesetzt; die anderen Bereiche bleiben erhalten.

Abgabe: ausgeführtes Notebook, Begründung der Kantenrichtung, Interpretation eines belegten Pfads und Beitrag zur gemeinsamen Modellvergleichstabelle.
