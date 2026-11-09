# ADR-002: PostgreSQL und Redis als zentrale Infrastruktur

- Status: Akzeptiert
- Datum: 2026-11-09

## Kontext

Die Anwendung benötigt:

- persistente Speicherung von Dokument-Metadaten, Status und Ergebnisdaten
- asynchrone Job-Verarbeitung ohne Blockade des Upload-Requests
- zuverlässige Statuswechsel (`queued`, `processing`, `done`, `failed`)
- einfache, nachvollziehbare Operabilität im MVP

## Entscheidung

Wir nutzen:

- PostgreSQL für die primäre Datenhaltung
- Redis für Queue, Status-Updates und einfache Caching-Funktionen

## Alternativen

### Nur Dateisystem + in-memory Queue

- einfach, aber unzuverlässig für Statuspersistenz und Long-Running Jobs
- schlechte kontrollierbare Konsistenz

### Vollwertiger Message-Broker mit komplexem Event-Stack

- sehr flexibel, aber für den MVP viel zu aufwändig
- schwerer zu debuggen und zu dokumentieren

### Nur NoSQL-Datenbank ohne Relationalschema

- für das einfache Datenmodell machbar, aber weniger transparent und weniger gut für Transaktionen und Referenzen

## Gründe für die Entscheidung

- PostgreSQL eignet sich gut für Relationen, Status, API-Key-Verwaltung und JSON-Spalten
- Redis ist für Job-Queue und schnelle Updates sehr geeignet und leicht zu betreiben
- Die Kombination ist ein guter Kompromiss aus Einfachheit, Zuverlässigkeit und Klarheit
- Die Architektur bleibt bis zur späteren Skalierung verständlich

## Konsequenzen

### Positiv

- klare Trennung zwischen persistenten Daten und asynchroner Verarbeitung
- einfache Statuslogik und Job-Management
- gute Eignung für Demo- und Testumgebungen
- geringe Betriebs-Komplexität

### Negativ

- Redis und PostgreSQL müssen überwacht und gesichert werden
- bei sehr großen Lasten ist später eine Skalierung der Infra nötig

## Bewertung

Diese Kombination ist bewusst nicht „überladen“, sondern technischer Standard für kleine bis mittelgroße Async-Architekturen. Sie erfüllt den MVP vollständig und lässt sich mit wenig Aufwand auf eine größere Umgebung erweitern.
