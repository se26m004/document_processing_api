# Hosting – Optionen und Empfehlung

## 1. Ziel und Rahmen

Das Hosting muss für den MVP die folgenden Anforderungen erfüllen:

- einfache lokale Entwicklung
- nachvollziehbare Demo-Umgebung
- geringe Betriebskomplexität
- tragfähige Grundlage für spätere Skalierung
- geeignete Kostenstruktur für ein Studienprojekt

Die Architekturentscheidung basiert auf [docs/architecture.md](architecture.md). Die Empfehlung lautet: kleines Hosting-Setup auf einer VM bzw. leichtem Cloud-Container-Host mit Docker Compose.

---

## 2. Hosting-Alternativen

### Variante A: Lokale VM / Docker Compose

Beschreibung:

- API und Worker laufen als Docker-Container
- PostgreSQL und Redis als lokale Services
- Dokumentenspeicher als lokaler Volume oder MinIO
- Nginx/Traefik als Reverse Proxy

Vorteile:

- sehr geringer Erstaufwand
- leicht nachvollziehbar und gut für Präsentationen
- vollständige Kontrolle über Services
- geringe Betriebskosten

Nachteile:

- kein High-Availability-Setup
- keine automatisierte Multi-Region-Redundanz
- bei längerer Nutzung muss Betrieb und Backups organisiert werden

Typische Kosten:

- 10–40 EUR/Monat für kleine VM oder niedrige Cloud-Instanz
- zusätzlich geringe Kosten für Storage

### Variante B: Cloud-Hosting mit managed Services

Beschreibung:

- managed PostgreSQL
- managed Redis
- Container-Deployment auf Render, Railway, Azure Container Apps, Cloud Run oder ECS
- Objektstorage via S3/MinIO-ähnlich

Vorteile:

- geringerer Betrieb, mehr Fokus auf Code
- einfacher Scaling bei Lastspitzen
- bessere Betriebsqualität für spätere Produktivisierung

Nachteile:

- höhere laufende Kosten
- mehr Abhängigkeit von Cloud-Konfigurationen
- mehr Komplexität bei Entwicklung und Debugging

Typische Kosten:

- 30–200 EUR/Monat je nach Umfang, Storage und Traffic
- bei mehreren Services schnell deutlich höher

### Variante C: Serverless / pure Functions

Beschreibung:

- API-Layer als Serverless-Endpoint
- Worker als FaaS-Funktionen
- Queue mit managed Messaging bzw. Event Bus
- Storage in Cloud-Objektspeicher

Vorteile:

- sehr hohe Skalierung
- wenig Server-Management

Nachteile:

- komplexe Architektur
- teuer bei häufigen Aufrufen
- schwierige lokale Testbarkeit und Präsentation
- insgesamt überdimensioniert für das Semesterprojekt

Typische Kosten:

- 20–150 EUR/Monat bei kleinem Einsatz
- kann bei vielen Jobs schnell zunehmen

---

## 3. Kostenvergleich

| Hosting-Option | Komplexität | Betrieb | Monatliche Kosten | Eignung für MVP |
|---|---|---:|---:|---|
| Docker Compose auf VM | Niedrig | Mittel | 10–40 EUR | Sehr gut |
| Managed Cloud Stack | Mittel | Niedrig | 30–200 EUR | Gut |
| Serverless / Functions | Hoch | Sehr niedrig | 20–150 EUR | Nicht ideal |

### Einschätzung

Für den MVP ist die VM-/Docker-Variante die sinnvollste Lösung. Die Architektur bleibt klein und erklärbar, und die Kosten bleiben überschaubar. Der Betrieb ist zwar nicht so „automatisiert“ wie in einem managed Cluster, aber deutlich leichter zu verstehen und zu dokumentieren.

---

## 4. Empfehlung für das Projekt

### Empfohlener MVP-Deploy

- 1 kleine VM oder Cloud-Instance
- Docker Compose
- PostgreSQL Container
- Redis Container
- API-Container + Worker-Container
- lokaler Volume oder MinIO für PDFs
- Reverse Proxy / TLS-Termination

### Begründung

- sehr gut passend zu [docs/architecture.md](architecture.md)
- einfache Erklärung während der Präsentation
- geringer Aufbauaufwand und übersichtliche Kosten
- bietet ausreichend Ressourcen für wenige parallele Jobs und Demo-Tests

### Für spätere Produktivphase

Wenn das System später mehr Last, mehr Data-Volumes oder mehr Sicherheit braucht, kann das Setup schrittweise auf managed Services umgestellt werden. Die Architektur bleibt dadurch konsistent, obwohl die Infrastruktur später besser skalierbar wird.

---

## 5. Betriebskriterien und Risiken

- Backups: PostgreSQL-Backups und Dokumentenspeicher-Backups müssen regelmäßig geplant werden
- Datensicherung: keine echten Kundendaten im MVP, aber sicherheitsrelevante Dateien müssen geschützt werden
- LLM-Kosten: Betreiber muss Budget- und Rate-Limits definieren
- Logs: keine Dokumentinhalte in Logs; nur Job-IDs, Status und Fehlercodes
- Security: API-Key-Hashing, TLS-Termination via Proxy, Zugriffsbeschränkung auf Admin-Ports

---

## 6. Fazit

Die beste Wahl für dieses Projekt ist ein kleines, verständliches Hosting-Setup mit Docker Compose auf einer VM oder einem günstigen Cloud-Container-Host. Es ist technisch sinnvoll, kosteneffizient und passt zur Anforderung, dass die Architektur für Studierende verständlich und verteidigbar sein muss.
