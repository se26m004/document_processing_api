# ADR-001: Modularer Monolith statt Microservices

- Status: Akzeptiert
- Datum: 2026-11-09

## Kontext

Die API soll im MVP Dokumente verarbeiten, LLM-Extraktion durchführen, Jobs asynchron ausführen und die Ergebnisse validieren. Die Anforderungen sind klar definiert, aber das Team ist klein und die Entwicklungszeit begrenzt. Es muss keine extreme Skalierung für Millionen von Anfragen geben.

Die Architektur muss außerdem leicht verständlich, testbar und durch das Team verteidigbar sein.

## Entscheidung

Wir implementieren eine modulare Monolith-Architektur mit klar getrennten Bereichen:

- API-Bereich
- Worker-Bereich
- Datenbank- und Job-Schicht
- Dateispeicher

Die Verarbeitung läuft asynchron über einen Job-Queue; die Komponenten bleiben aber trotzdem im selben Codebase und Deployment-Unit.

## Alternativen

### Microservices / Event-Driven Architecture

- Aufteilung in mehrere Services für API, Worker, Validation und Storage
- erheblicher Betrieb und Debugging-Aufwand
- stärkere Infrastruktur- und Deployment-Complexität

### Pure Serverless-Ansatz

- einzelne Funktionen für Upload, Verarbeitung und Status
- gute Skalierung, aber für das Semesterprojekt zu komplex

## Gründe für die Entscheidung

- leichter verständlich für Projektteam und Prüfer
- weniger Betriebsaufwand
- schnelle Entwicklung und geringere Einarbeitungszeit
- ausreichend für die erwartete Last des MVP
- besser passend zu den Anforderungen aus [docs/requirements.md](requirements.md) und [docs/mvp.md](mvp.md)

## Konsequenzen

### Positiv

- schnellere Implementierung
- einfachere Tests
- geringere Kosten
- gut für Präsentationen und Demo-Läufe

### Negativ

- späterer Austausch zu Microservices wäre möglich, aber aufwändiger
- Skalierung ist nicht so flexibel wie bei vollständig getrennter Service-Architektur

## Bewertung

Diese Entscheidung ist bewusst nicht „die komplexeste Lösung“, sondern die einfachste, die die Anforderungen zuverlässig erfüllt. Genau das ist für ein Semesterprojekt die technische und organisatorische sinnvollere Wahl.
