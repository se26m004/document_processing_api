# Document Processing API

Eine REST-API, die Dokumente entgegennimmt, analysiert und die enthaltenen Informationen als **strukturiertes JSON** zurückgibt – validiert gegen ein definiertes Schema. Die Langfristvision umfasst weitere Formate und Dokumenttypen; der MVP beschränkt sich auf PDF-Rechnungen.

> Status: Produktspezifikation (Phase 1). Noch kein Code. Details: [`docs/requirements.md`](docs/requirements.md) · [`docs/mvp.md`](docs/mvp.md) · [`docs/ai-usage.md`](docs/ai-usage.md)

## Problemstellung

Unternehmen erhalten täglich Dokumente (Rechnungen, Belege, Verträge, Lebensläufe) in uneinheitlichen Formaten. Die relevanten Daten werden heute von Hand abgetippt – langsam, fehleranfällig und teuer. Klassische OCR liefert nur Rohtext, keine Struktur; regelbasierte Extraktion bricht bei jedem neuen Layout.

**Idee:** Eine einzige, gut dokumentierte API, die den Text eines Dokuments liest, mit Hilfe eines LLM die gewünschten Felder extrahiert und das Ergebnis gegen ein Schema prüft. Entwickler:innen integrieren sie in ihre eigene Software, statt Extraktionslogik selbst zu bauen.

## Projektziel und Mehrwert

**Ziel:** Im Semester entsteht eine lauffähige API, die PDF-Rechnungen entgegennimmt und die Kerndaten (Rechnungsnummer, Datum, Lieferant, Währung, Beträge) als schema-validiertes JSON liefert.

**Mehrwert:**
- Entwickler:innen sparen sich den Bau eigener Extraktionslogik und bekommen ein stabiles, dokumentiertes Ergebnisformat.
- Fachabteilungen müssen Rechnungsdaten nicht mehr abtippen. Plausibilitätsprüfungen (z. B. Netto + Steuer = Brutto) fangen offensichtliche Fehler ab.
- Messbar wird der Nutzen über die Genauigkeit der Pflichtfelder (Ziel ≥ 90 % auf dem Testset) und die Bearbeitungszeit je Dokument.

## Zielgruppe

| Gruppe | Bedarf |
|---|---|
| **Entwickler:innen** (Primärnutzer der API) | Dokumente per HTTP hochladen, verlässliches JSON zurückbekommen, klare Fehlermeldungen |
| **Fachabteilungen** (Buchhaltung, HR) – indirekt über deren Software | Weniger Abtippen, Fehler erkennen, unsichere Felder prüfen |
| **Betreiber / Admin** | Nutzung, Kosten und Fehlerquoten überwachen, Zugriffe verwalten |

## Wichtigste Anwendungsfälle

1. **Rechnung hochladen** → Lieferant, Rechnungsnummer, Datum, Währung, Summen als JSON (Positionen: spätere Ausbaustufe).
2. **Dokumenttyp automatisch erkennen** (Rechnung / Beleg / Vertrag / Lebenslauf / Sonstiges) – spätere Ausbaustufe.
3. **Eigenes Schema definieren** → Extraktion nur der gewünschten Felder – spätere Ausbaustufe.
4. **Asynchrone Verarbeitung** mit Statusabfrage (Webhook als spätere Ausbaustufe).
5. **Unsichere Felder markieren**, damit ein Mensch gezielt nachprüft – spätere Ausbaustufe.

## MVP in einem Satz

Authentifizierter Upload eines Text-PDFs → asynchrone Verarbeitung (LLM-Extraktion) → Abruf des schema-validierten JSON-Ergebnisses für den Typ **Rechnung** (Kopfdaten und Summen) → Löschen des Dokuments. Enthaltenes und Ausgeschlossenes: [`docs/mvp.md`](docs/mvp.md).

## Grobe Architektur (Skizze, noch keine Technologieentscheidung)

```
Client ──HTTPS──▶ API (Upload, Auth, Status, Ergebnis)
                    │
                    ├─▶ Dateispeicher (Originale, temporär)
                    └─▶ Job-Queue ─▶ Worker: PDF-Text → LLM-Extraktion
                                              → Schema-Validierung
                                              └─▶ Ergebnisspeicher (JSON)
```

Technologieentscheidungen (Sprache, Framework, LLM-Anbieter) sind bewusst **noch offen** und werden nach der Spezifikation getroffen.

## Dokumentation

| Datei | Inhalt |
|---|---|
| [`docs/requirements.md`](docs/requirements.md) | User Stories mit Akzeptanzkriterien, funktionale und nichtfunktionale Anforderungen, Priorisierung, Abhängigkeiten, Konsistenzprüfung |
| [`docs/mvp.md`](docs/mvp.md) | MVP-Anwendungsfall, enthaltene und ausgeschlossene Funktionen, Erfolgskriterien |
| [`docs/ai-usage.md`](docs/ai-usage.md) | KI-Anwendungen, ausgewählte Prompts und Ergebnisse, kritische Bewertung, Reflexion |

## Team

Trajkoski Daniel _(weitere Teammitglieder ergänzen)_
