# MVP – Document Processing API

## Demonstrierbarer Anwendungsfall

> Ein:e Entwickler:in lädt mit einem API-Key eine PDF-Rechnung hoch, fragt den Status ab und erhält ein schema-validiertes JSON mit Rechnungsnummer, Datum, Lieferant, Währung und Beträgen. Danach löscht sie das Dokument wieder.

Das ist ein vollständiger Durchlauf von Upload bis Löschung und wird am Semesterende live vorgeführt.

## Enthalten (Must)

| Funktion | Anforderung | Story |
|---|---|---|
| PDF-Upload (≤ 10 MB) | FR-01 | US-01 |
| Asynchrone Verarbeitung per Job-Queue | FR-02 | US-01, US-03 |
| Statusabfrage | FR-03 | US-03 |
| Ergebnisabruf als JSON | FR-04 | US-02 |
| Textextraktion aus PDF mit Textebene | FR-05 | US-04 |
| LLM-Extraktion Rechnung (Kopfdaten und Summen) | FR-06 | US-04 |
| Schema-Validierung des Ergebnisses | FR-08 | US-02, US-03, US-04 |
| Summenprüfung (`sum_mismatch`) | FR-24 | US-04 |
| Seitenlimit 5, Fehler `no_text_layer` / `too_many_pages` | FR-01 | US-01 |
| API-Key-Authentifizierung | FR-11 | US-10 |
| Löschen von Dokument und Ergebnis | FR-12 | US-11 |
| Einheitliche Fehlerantworten | FR-20 | alle |
| OpenAPI-Dokumentation, Quickstart, `/v1`-Pfade | NFR-09, NFR-13 | – |
| CI und automatisierte Tests mit gemocktem LLM | NFR-08 | – |

## Ausgeschlossen

| Funktion | Grund | Wann |
|---|---|---|
| Beleg-Extraktion (US-05) | Zweites Schema und Testset, MVP braucht nur einen Fall | Ausbaustufe 1 |
| Scans, OCR, JPG/PNG (US-12) | OCR-Qualität ist ein eigenes Risiko | Ausbaustufe 1 |
| Confidence und `needs_review` (US-07) | Technisch heikel (W-4) | Ausbaustufe 1 |
| Typ-Erkennung, eigenes Schema, Webhook (US-06/08/09) | Nicht nötig für den Kernfall | Ausbaustufe 1–2 |
| Rechnungspositionen | Deutlich schwerer als Kopfdaten (U-8) | Ausbaustufe 1 |
| Automatische Löschfrist (FR-23) | Manuelles Löschen reicht für die Demo | Ausbaustufe 1 |
| Statistik, Korrektur-Endpunkt, Batch (US-13–15) | Could | später |
| Web-Oberfläche, Handschrift, Mehrsprachigkeit über DE/EN, On-Premise, ZUGFeRD/XRechnung | Won't | – |

## Erfolgskriterien

1. Der Demo-Anwendungsfall läuft end-to-end ohne manuelle Eingriffe.
2. Genauigkeit auf dem Testset (≥ 30 Rechnungen) ist nach NFR-02 gemessen und dokumentiert; Ziel ≥ 90 % gesamt, ≥ 80 % je Pflichtfeld.
3. Alle Akzeptanzkriterien der sechs Must-Stories sind als automatisierte Tests (gemockter LLM) bestanden (NFR-08).
4. Eine Person außerhalb des Teams erreicht mit dem Quickstart in ≤ 15 Minuten ein Ergebnis (NFR-09).

## Abhängigkeiten und Kosten

LLM-API als externer Dienst; Anbieter und Budget sind noch offen (U-6) und vor Projektstart zu klären. Kosten je Dokument werden am Testset gemessen (NFR-11). Details und Risiken: [`requirements.md`, Abschnitt 5](requirements.md#5-abhängigkeiten-kosten-und-risiken).

## Realismus-Check

Sechs Stories, kein OCR, kein Frontend, ein einziges Schema, maximal 5 Seiten. Die größten Risiken sind die Genauigkeit der LLM-Extraktion und die Kosten. Beides wird früh am Testset gemessen, bevor weitere Funktionen gebaut werden. Wird der Aufwand zu groß, bleibt der Kernpfad Upload → Ergebnis erhalten; Authentifizierung und Löschen werden nicht gestrichen.
