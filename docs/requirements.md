# Anforderungen – Document Processing API

Priorisierung nach **MoSCoW**: **Must** = ohne das ist der MVP-Anwendungsfall nicht demonstrierbar · **Should** = wichtig, MVP bleibt ohne nutzbar · **Could** = nice to have · **Won't (jetzt)** = bewusst ausgeschlossen. MVP-Umfang und Abgrenzung: [`mvp.md`](mvp.md).

## 1. Glossar

| Begriff | Bedeutung |
|---|---|
| Dokument | Hochgeladene Datei (im MVP: PDF mit Textebene) |
| Job | Verarbeitungsauftrag zu einem Dokument, Status `queued` · `processing` · `done` · `failed` |
| Schema | Festgelegte Struktur der zu extrahierenden Felder (im MVP: Rechnung) |
| Extraktion | Auslesen der Feldwerte aus dem Dokumenttext mit Hilfe eines LLM |
| Confidence | Heuristischer Wert 0–1, wie sicher ein Feld ist (siehe W-4) |
| `needs_review` | Markierung, dass ein Mensch das Feld prüfen sollte |
| Pflichtfelder (Rechnung) | Rechnungsnummer, Rechnungsdatum, Lieferant, Bruttobetrag, Währung. Netto- und Steuerbetrag sind optionale Felder (können auf der Rechnung fehlen) |
| Pflichtfeld im Schema | Muss im Ergebnis vorhanden sein, darf aber `null` sein, wenn es im Dokument nicht steht (`["string","null"]`) |

## 2. Personas

| Persona | Beschreibung |
|---|---|
| **Dev (Entwickler:in)** | Baut eine Buchhaltungs-/HR-/Archiv-Software und integriert die API. Will wenig Boilerplate, gute Doku, stabile Fehlercodes. |
| **Fach (Sachbearbeiter:in)** | Nutzt indirekt das Ergebnis. Will wissen, welchen Feldern sie trauen kann. |
| **Admin (Betreiber)** | Verwaltet Zugänge, überwacht Betrieb und Kosten, verantwortet Datenschutz. |

## 3. User Stories

Format: *Als … möchte ich …, damit …* · Priorität in Klammern · Akzeptanzkriterien (AK) testbar formuliert. Jede Must-Story hat AK.

**US-01 Dokument hochladen (Must)**
Als Dev möchte ich ein PDF per `POST /v1/documents` hochladen, damit es verarbeitet wird.
- AK1: Gültiges PDF (≤ 10 MB) → `202` mit `document_id` und `job_id`.
- AK2: Anderer Dateityp → `415`, Datei > 10 MB → `413`; Body enthält `error.code` und `error.message`.
- AK3: Beschädigtes oder passwortgeschütztes PDF → `422` mit Code `unreadable_document`. Der Dateityp wird an den Magic Bytes (`%PDF`) erkannt, nicht an Endung oder Header.
- AK4: PDF ohne extrahierbaren Text (z. B. Scan) → `422` mit Code `no_text_layer`.
- AK5: Mehr als 5 Seiten → `422` mit Code `too_many_pages`.
- AK6: Fehlendes Datei-Feld → `400`. Rate Limiting (`429`) ist nicht Teil des MVP.

**US-02 Ergebnis abrufen (Must)**
Als Dev möchte ich per `GET /v1/documents/{id}/result` das extrahierte JSON abrufen, damit ich es weiterverarbeiten kann.
- AK1: Bei Status `done` enthält die Antwort `document_type`, `fields`, `warnings`, `schema_version` (Beispiel in Abschnitt 6).
- AK2: Bei Status `queued`, `processing` oder `failed` → `409` mit aktuellem Status und, bei `failed`, `error.code`; nie ein Teilergebnis.
- AK3: Das Ergebnis validiert gegen das veröffentlichte JSON-Schema der Rechnung.

**US-03 Status verfolgen (Must)**
Als Dev möchte ich per `GET /v1/jobs/{id}` den Status abfragen, damit meine Anwendung warten oder erneut versuchen kann.
- AK1: Status ist einer von `queued`, `processing`, `done`, `failed`.
- AK2: Bei `failed` enthält die Antwort `error.code` (z. B. `extraction_invalid`, `llm_unavailable`, `timeout`).
- AK3: Ein Job, der länger als 120 s im Status `processing` ist, wird `failed` mit Code `timeout`.

**US-04 Rechnung extrahieren (Must)**
Als Fach möchte ich, dass Rechnungsnummer, Datum, Lieferant, Währung sowie Netto-, Steuer- und Bruttobetrag erkannt werden, damit ich sie nicht abtippen muss.
- AK1: Auf dem Testset (≥ 30 synthetische/öffentliche Rechnungen mit Layout-Varianten, Anteil DE/EN, Ground Truth manuell) sind alle Pflichtfeld-Werte zusammen zu ≥ 90 % korrekt und jedes einzelne Pflichtfeld zu ≥ 80 % (Metrik siehe NFR-02).
- AK2: Sind Netto, Steuer und Brutto alle vorhanden und gilt Netto + Steuer ≠ Brutto (Toleranz 0,01), enthält `warnings` den Eintrag `sum_mismatch`. Fehlt einer der drei Beträge, wird nicht geprüft.
- AK3: Fehlt ein Feld im Dokument, ist es `null` – es wird nie ein Wert erfunden. Prüfbar mit einem Testdokument ohne Rechnungsnummer.

**US-10 Authentifizierung (Must)**
Als Admin möchte ich, dass nur Clients mit gültigem API-Key zugreifen, damit keine fremden Dokumente einsehbar sind.
- AK1: Ohne oder mit ungültigem Key → `401`.
- AK2: Ein Key sieht nur eigene Dokumente; fremde ID → `404`.
- AK3: Keys werden nur als Hash gespeichert und per Skript angelegt oder widerrufen; eine Benutzerverwaltung gibt es nicht.

**US-11 Daten löschen (Must)**
Als Dev möchte ich ein Dokument samt Ergebnis per `DELETE /v1/documents/{id}` löschen, damit Datenschutzvorgaben erfüllt werden.
- AK1: Nach `204` liefern Ergebnis- und Statusabruf `404`. Gelöscht sind Original-PDF, extrahierter Text, Ergebnis-JSON und Queue-Eintrag; das Dateiverzeichnis enthält keine Datei zur ID mehr.
- AK2: Erneutes Löschen derselben ID → `404`.
- AK3: Löschen während `processing` bricht den Job ab; ein später eintreffendes Ergebnis wird verworfen.
- Hinweis: Daten, die bereits an den LLM-Anbieter gesendet wurden, kann die API nicht löschen (siehe NFR-04).

**US-07 Confidence & Prüfhinweise (Should)**
Als Fach möchte ich je Feld einen Confidence-Wert und `needs_review` sehen, damit ich nur unsichere Felder prüfen muss.
- AK: Jedes Feld hat `confidence` ∈ [0,1]; Felder unter konfigurierbarer Schwelle sind mit `needs_review: true` markiert.

**US-05 Beleg extrahieren (Should)**
Als Fach möchte ich Händler, Datum, Gesamtbetrag und Währung aus Kassenbons erhalten, damit Spesen schneller erfasst werden.
- AK: Wie US-04 mit reduziertem Schema.

**US-12 Scans und Bilder (Should)**
Als Fach möchte ich auch gescannte PDFs und JPG/PNG verarbeiten, damit reale Eingänge funktionieren.
- AK: Scans ab 200 dpi werden per OCR gelesen; Genauigkeit wird getrennt vom Text-PDF gemessen.

**US-06 Dokumenttyp erkennen (Should)**
Als Dev möchte ich keinen Typ angeben müssen, damit ich gemischte Eingänge verarbeiten kann.
- AK: Ohne `type` wird klassifiziert; bei zu niedriger Sicherheit → `unknown` statt Raten.

**US-08 Eigenes Schema definieren (Should)**
Als Dev möchte ich ein JSON-Schema mitgeben, damit ich beliebige Felder extrahieren kann.
- AK: Ungültiges Schema → `422` mit Fehlerstelle; Ergebnis validiert gegen das Schema oder der Job ist `failed`.

**US-09 Webhook (Should)**
Als Dev möchte ich per Callback-URL benachrichtigt werden, damit ich nicht pollen muss.
- AK: Signierter `POST` mit Job-Status; bis zu 3 Wiederholungen bei Fehler.

**US-13 Nutzung & Kosten (Could)** – Als Admin möchte ich Jobs, Fehlerquote und Kosten je API-Key sehen.
**US-14 Korrektur zurückspielen (Could)** – Als Fach möchte ich korrigierte Werte zurückgeben, damit Fehler zur Qualitätsmessung genutzt werden.
**US-15 Batch-Upload (Could)** – Als Dev möchte ich mehrere Dokumente in einem Request senden.

**Zähler:** 15 Stories – 6 Must, 6 Should, 3 Could.

## 4. Anforderungen

### 4.1 Funktionale Anforderungen

| ID | Anforderung | Prio | Story |
|---|---|---|---|
| FR-01 | Upload von PDF (max. 10 MB) über REST | Must | US-01 |
| FR-02 | Verarbeitung asynchron über Job-Queue; Jobs liegen persistent (Datenbank), hängende Jobs laufen in `timeout` | Must | US-01, US-03 |
| FR-03 | Statusabfrage je Job | Must | US-03 |
| FR-04 | Abruf des Ergebnisses als JSON | Must | US-02 |
| FR-05 | Textextraktion aus PDF mit Textebene | Must | US-04 |
| FR-06 | LLM-Extraktion mit festem Schema für **Rechnung** | Must | US-04 |
| FR-08 | Validierung des LLM-Outputs gegen JSON-Schema; bei Verstoß ein inhaltlicher Korrekturversuch (1×), dann `failed` mit `extraction_invalid` | Must | US-02, US-03, US-04 |
| FR-24 | Summenprüfung Netto + Steuer = Brutto mit `warnings`-Eintrag | Must | US-04 |
| FR-11 | API-Key-Authentifizierung mit Trennung je Key | Must | US-10 |
| FR-12 | Löschen von Dokument und Ergebnis | Must | US-11 |
| FR-20 | Einheitliches Fehlerformat mit maschinenlesbarem Code | Must | alle |
| FR-07 | Extraktion für **Beleg** | Should | US-05 |
| FR-09 | Confidence und `needs_review` je Feld | Should | US-07 |
| FR-10 | Weitere Plausibilitätsregeln (Datum, Währung) als `warnings` | Should | US-04 |
| FR-13 | Automatische Dokumenttyp-Erkennung | Should | US-06 |
| FR-14 | Benutzerdefiniertes Schema | Should | US-08 |
| FR-15 | Webhook bei Abschluss | Should | US-09 |
| FR-21 | OCR für Scans und Bildformate | Should | US-12 |
| FR-22 | Upload von JPG/PNG | Should | US-12 |
| FR-23 | Automatische Löschung nach Aufbewahrungsfrist | Should | US-11 |
| FR-16 | DOCX als Eingabe | Could | – |
| FR-17 | Nutzungsstatistik je Key | Could | US-13 |
| FR-18 | Korrektur-Endpunkt | Could | US-14 |
| FR-19 | Batch-Upload | Could | US-15 |

Retry-Semantik: Der **Korrekturversuch** (FR-08) betrifft ungültigen LLM-Output und erfolgt 1×. Die **technische Wiederholung** (NFR-07) betrifft Ausfälle wie `llm_unavailable` und erfolgt bis zu 2×.

### 4.2 Nichtfunktionale Anforderungen

| ID | Kategorie | Anforderung | Prüfmethode | Prio |
|---|---|---|---|---|
| NFR-01 | Performance | `202`-Antwort beim Upload ≤ 2 s; Verarbeitung jedes Dokuments des Testsets (≤ 5 Seiten) ≤ 60 s, Median ≤ 30 s | Zeitmessung über alle Testdokumente | Must |
| NFR-02 | Genauigkeit | Mikro-Mittel über alle Pflichtfeld-Werte aller Testdokumente ≥ 90 %, jedes einzelne Pflichtfeld ≥ 80 %. Nicht erreichtes Ziel wird gemessen und dokumentiert, nicht verschwiegen. Vergleich normalisiert (Datum ISO-8601, Zahl ohne Tausendertrenner, Beträge auf 2 Stellen, Strings getrimmt, Groß-/Kleinschreibung ignoriert) | Auswertungsskript gegen Ground Truth des Testsets | Must |
| NFR-03 | Sicherheit | Zugriff nur mit API-Key; keine Dokumentinhalte in Logs. TLS-Terminierung übernimmt im Deployment ein Reverse Proxy und ist nicht Teil des MVP-Codes | Test auf `401`; Test mit Marker-Text im Dokument, danach `grep` in den Logs ohne Treffer | Must |
| NFR-04 | Datenschutz | Im MVP werden ausschließlich synthetische oder öffentliche Dokumente verarbeitet; Löschung gemäß US-11. Der Text wird an den LLM-Anbieter übertragen; dessen Aufbewahrung ist nicht steuerbar. Vor Produktivbetrieb mit echten Daten: AV-Vertrag und Zero-Retention-Option prüfen (siehe W-2) | Testset-Herkunft dokumentiert; Test auf US-11 AK1 | Must |
| NFR-09 | Dokumentation | OpenAPI-Spezifikation mit Beispiel je Endpunkt; Quickstart führt in ≤ 15 Minuten zum ersten Ergebnis | Zeitmessung mit einer Person außerhalb des Teams, die Docker und curl installiert hat | Must |
| NFR-08 | Wartbarkeit | CI läuft bei jedem Push; die Akzeptanzkriterien der Must-Stories sind als automatisierte Tests mit **gemocktem LLM** abgedeckt. Die Genauigkeitsmessung (NFR-02) ist ein separater, manuell gestarteter Lauf gegen die echte API | CI-Ergebnis, Testbericht | Must |
| NFR-13 | Kompatibilität | Alle Pfade unter `/v1` | Review der OpenAPI | Must |
| NFR-07 | Zuverlässigkeit | Bei technischen Fehlern (`llm_unavailable`) bis zu 2 Wiederholungen; Jobs gehen bei Neustart nicht verloren | Test: Worker während Job beenden | Should |
| NFR-14 | Robustheit | LLM-Aufruf mit Timeout (20 s), Temperature 0, Backoff bei Rate Limit; Dokumentinhalt wird nur als Daten behandelt (Prompt-Injection-Test mit präpariertem Dokument) | Test mit gemocktem LLM, Injektions-Testdokument | Should |
| NFR-10 | Beobachtbarkeit | Strukturierte Logs mit Job-ID, Dauer, Status | Log-Stichprobe | Should |
| NFR-11 | Kosten | Kosten je Dokument gemessen und dokumentiert; Obergrenze je Testlauf konfigurierbar (Budget siehe U-6, vor Projektstart zu klären) | Abrechnungsdaten des Anbieters nach Testlauf | Should |
| NFR-12 | Reproduzierbarkeit | Ergebnis enthält zusätzlich `model_version` (Modell-ID des Anbieters + Prompt-Version) | Schematest | Should |
| NFR-05 | Verfügbarkeit | 99 % im Pilotbetrieb | – | Could |
| NFR-06 | Skalierbarkeit | Worker horizontal skalierbar; ≥ 50 parallele Jobs | – | Could |

## 5. Abhängigkeiten, Kosten und Risiken

| Abhängigkeit | Wofür | Kosten / Risiko | Umgang |
|---|---|---|---|
| LLM-API (externer Anbieter) | Extraktion (FR-06) | Kosten pro Dokument noch **unbekannt**; Ausfall, Rate Limits, Datenschutz (W-2) | Kosten am Testset messen (NFR-11), Testlauf-Limit setzen, Fehlercode `llm_unavailable` |
| PDF-Textextraktion (Bibliothek) | FR-05 | Kein Text bei Scans | Scans bewusst Should (US-12) |
| Hosting / Queue | FR-02 | Lokal oder kleiner Cloud-Dienst | Im MVP lokal lauffähig (Docker) |
| Testdaten | NFR-02 | Echte Rechnungen datenschutzkritisch | Synthetisch erzeugen oder öffentliche Muster nutzen (U-2 entschieden) |

Konkrete Technologien werden erst in Phase 2 gewählt, nachdem die Anforderungen feststehen.

## 6. Beispiel-Ergebnis (Zielformat)

```json
{
  "document_id": "d_123",
  "document_type": "invoice",
  "schema_version": "1.0",
  "fields": {
    "invoice_number": "RE-2026-0042",
    "invoice_date": "2026-09-30",
    "supplier": "Muster GmbH",
    "currency": "EUR",
    "net_amount": 100.00,
    "tax_amount": 19.00,
    "gross_amount": 119.00
  },
  "warnings": []
}
```

## 7. Prüfung auf Widersprüche und Unklarheiten

### 7.1 Widersprüche / Spannungen

| ID | Befund | Auflösung |
|---|---|---|
| W-1 | **30 s vs. Mehrseiter:** NFR-01 nennt 30 s, Mehrseiter und Scans sind langsamer. | Im MVP sind maximal 5 Seiten erlaubt (US-01 AK5); Zeitziel 60 s je Dokument. Scans sind nicht im MVP (US-12 Should). |
| W-2 | **Datenschutz vs. Cloud-LLM:** NFR-04 steht im Spannungsfeld zu FR-06. | MVP nur mit synthetischen/öffentlichen Daten; für Echtdaten EU-Anbieter mit AV-Vertrag oder lokales Modell, Entscheidung vor Produktivbetrieb. |
| W-3 | **Löschung vs. Reproduzierbarkeit:** FR-12 löscht, NFR-12 will Nachvollziehbarkeit. | Nach Löschung bleiben nur Metadaten ohne Inhalte (Job-ID, Zeit, Version, Status). |
| W-4 | **Confidence vs. Technik:** LLMs liefern keine verlässlichen Wahrscheinlichkeiten. | Confidence heuristisch (Validierung, Plausibilität, Modellselbsteinschätzung); deshalb Should, nicht Must. |
| W-5 | **Genauigkeit vs. Kosten:** 90 % kann größere Modelle erfordern. | Modellwahl erst nach Messung von Genauigkeit und Kosten am Testset. |
| W-6 | **Freies Schema vs. Qualitätsziel:** US-08 widerspricht der Genauigkeitszusage. | Zusage nur für den definierten Typ Rechnung; freie Schemata „best effort". |
| W-7 | **Must-Umfang vs. Semesterrahmen:** Der erste Entwurf hatte 11 Must-FRs plus Beleg, Scans und Confidence – zu viel für ein Semester. | Verkleinert: Beleg, Scans, Confidence, Webhooks → Should (siehe `mvp.md`). |

### 7.2 Unklarheiten und Entscheidungen

| ID | Frage | Status |
|---|---|---|
| U-1 | Welche Sprachen? | **Entschieden:** MVP nur deutsche und englische Rechnungen |
| U-2 | Testdaten? | **Entschieden:** synthetisch/öffentlich, ≥ 20 Rechnungen, Ground Truth manuell |
| U-3 | Ist 90 % realistisch? | **Offen:** Zielwert ist eine Annahme; Abnahmekriterium ist die dokumentierte Messung (NFR-02), nicht das Erreichen |
| U-4 | Wann ist ein Feld korrekt? | **Entschieden:** normalisierter Vergleich (NFR-02) |
| U-5 | Erwartete Last? | **Offen:** MVP-Demo mit wenigen parallelen Jobs, NFR-05/06 deshalb Could |
| U-6 | Hosting und Budget für LLM-Kosten? | **Offen, vor Projektstart zu klären:** Anbieter und Budget je Testlauf |
| U-7 | Mehrere Mandanten mit Admins? | **Entschieden:** nur API-Keys, per Skript verwaltet (US-10 AK3) |
| U-8 | Rechnungspositionen im MVP? | **Entschieden:** nein, nur Kopfdaten und Summen; Positionen sind erste Ausbaustufe |

## 8. Annahmen

- Eingaben im MVP sind maschinell erzeugte PDFs mit Textebene, deutsch oder englisch.
- Der Betrieb erfolgt als Demo ohne SLA.
- Ein externes LLM wird genutzt; Kosten und Anbieter sind noch festzulegen.
