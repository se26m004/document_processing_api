# AI-Einsatz und Reflexion

## 1. Eingesetzte Werkzeuge

| Werkzeug | Zweck |
|---|---|
| Claude Code (Claude Sonnet 5.5) | Anforderungsanalyse, Entwurf der Deliverables, Abgleich mit der Aufgabenbeschreibung |
| Claude-Subagent (separate Sitzung) | Unabhängige Prüfung auf Widersprüche und Lücken |

## 2. KI-Anwendungen im Überblick

| Nr. | Anwendung | Werkzeug | Ergebnis |
|---|---|---|---|
| A1 | **Generierung** von Problemdefinition, Personas, User Stories, Anforderungen, Priorisierung und MVP aus der groben Idee | Claude Code (Hauptsitzung) | Erstentwurf von `README.md`, `requirements.md` |
| A2 | **Abgleich** des Entwurfs mit den Detail-Vorgaben der Aufgabe (Checkliste: Deliverables, Mindestanzahlen, Qualitätsanforderungen) | Claude Code (Hauptsitzung) | Lücken gefunden: `mvp.md` fehlte, zu viele Must-Anforderungen, keine Abhängigkeiten/Kosten, Messbarkeit unklar → Überarbeitung |
| A3 | **Unabhängige Prüfung** auf Widersprüche, Lücken, Unklarheiten und Realismus durch eine getrennte KI-Instanz ohne Kenntnis der Entstehung | Claude-Subagent (frische Sitzung, nur Lesezugriff) | 12 Befunde, siehe Abschnitt 4 |

A3 wurde bewusst als eigene Instanz ausgeführt, weil die Prüfung des eigenen Entwurfs durch dieselbe Sitzung blinde Flecken hat.

## 3. Ausgewählte Prompts und Ergebnisse

### A1 – Generierung (Prompt wörtlich, Nutzer)

> Ich habe einen Ordner mit github gecloned, wir haben folgendes Thema: Document Processing API – Dokumente hochladen, analysieren und strukturierte Daten extrahieren und wir müssen folgendes machen: *Ziel:* Aus einer groben Idee eine strukturierte und umsetzbare Produktspezifikation entwickeln. *TODOs:* Projektidee und Problemstellung definieren · Zielgruppe und wichtigste Anwendungsfälle identifizieren · AI zur Anforderungsanalyse einsetzen · Mindestens 8–10 User Stories erstellen · Funktionale und nichtfunktionale Anforderungen definieren · Anforderungen priorisieren (Must/Should/Could) · MVP-Scope festlegen · Anforderungen auf Widersprüche und Unklarheiten prüfen. *Deliverables:* README.md, docs/requirements.md, docs/ai-usage.md

**Ergebnis (Kurzfassung):** 15 User Stories, 20 FR, 13 NFR, MVP mit Rechnung und Beleg, 6 Widersprüche, 8 offene Fragen. Der Entwurf hatte 11 Must-Anforderungen und war für ein Semester zu groß.

### A2 – Abgleich mit den Detail-Vorgaben (Prompt wörtlich, Nutzer)

> schaue ob alles davon umgesetzt ist, wenn nicht dann füge den rest hinzu

(zusammen mit dem eingefügten Text der Aufgabenbeschreibung)

**Ergebnis:** `docs/mvp.md` angelegt; MVP von Rechnung+Beleg auf Rechnung reduziert; Beleg, Scans/OCR, Confidence, Webhooks auf Should; Abschnitt „Abhängigkeiten, Kosten und Risiken", Glossar und Beispiel-JSON ergänzt; Prüfmethode je NFR; Akzeptanzkriterien für alle Must-Stories.

### A3 – Unabhängige Prüfung (Prompt wörtlich)

> Du bist ein kritischer Requirements-Engineering-Reviewer. Lies docs/requirements.md und docs/mvp.md (nur lesen, nichts ändern). Es geht um eine "Document Processing API" (Dokumente hochladen, per LLM strukturierte Daten extrahieren), umzusetzen in einem Studiensemester von einem kleinen Team.
>
> Suche gezielt nach: (1) Widersprüchen zwischen Stories, Anforderungen, NFRs und MVP-Dokument (IDs, Prioritäten, Zahlen), (2) nicht überprüfbaren oder mehrdeutigen Formulierungen, (3) fehlenden Anforderungen/Randfällen, (4) Must-Anforderungen, die nicht wirklich unverzichtbar sind oder für ein Semester unrealistisch, (5) Stellen, an denen ein Entwickler Rückfragen hätte. Keine Stilkritik.
>
> Antworte auf Deutsch als nummerierte Liste, je Befund: Fundstelle (Datei, ID), Problem, konkreter Verbesserungsvorschlag. Maximal 12 Befunde, wichtigste zuerst. Keine Einleitung.

**Ergebnis:** 12 Befunde. Der Prompt enthielt bewusst keine eigene Einschätzung des Entwurfs, um kein bestätigendes Ergebnis zu provozieren.

## 4. Kritische Bewertung und Entscheidungen

### 4.1 Umgang mit den 12 Befunden aus A3

| # | Befund (Kurz) | Entscheidung | Begründung / Abweichung |
|---|---|---|---|
| 1 | 90 %-Metrik uneinheitlich, Testset zu klein | **Übernommen** | Mikro-Mittel ≥ 90 % plus ≥ 80 % je Feld, Testset ≥ 30; Abnahme = dokumentierte Messung. Der Hinweis, Währung sei trivial, wurde nicht umgesetzt, weil das je-Feld-Minimum das Problem abfängt |
| 2 | Summenprüfung an drei Stellen unterschiedlich eingeordnet | **Übernommen** | Eigene Anforderung FR-24 (Must); Fallback-Satz im MVP gestrichen |
| 3 | Pflichtfelder vs. Netto/Steuer unklar | **Übernommen** | Netto/Steuer optional, Summenprüfung nur wenn alle drei Beträge vorhanden |
| 4 | Verhalten bei `failed`, Retry-Semantik | **Übernommen** | `/result` liefert bei `failed` `409`; Korrekturversuch (1×) und technische Wiederholung (2×) getrennt |
| 5 | Persistenz der Queue, hängende Jobs | **Übernommen** | Jobs persistent in DB, Timeout 120 s |
| 6 | NFR-01 mit p95 bei 20 Werten nicht prüfbar; Mehrseiter nicht durchgesetzt | **Übernommen** | Max. 60 s / Median 30 s über das Testset; Seitenlimit 5 (`too_many_pages`) |
| 7 | Upload-Randfälle | **Teilweise** | Magic Bytes, `no_text_layer`, `400`, Rate Limit ausgeschlossen. Duplikate und ID-Format bewusst nicht geregelt (für den Demo-Fall irrelevant) |
| 8 | Key-Verwaltung, TLS, Log-Test | **Übernommen** | Keys gehasht, per Skript; TLS über Reverse Proxy außerhalb des MVP-Codes; Log-Test mit Marker-Text |
| 9 | Löschumfang undefiniert | **Übernommen** | Löschumfang aufgelistet, Verhalten bei `processing`, Hinweis auf LLM-Anbieter |
| 10 | Traceability, NFR-12/13 | **Teilweise** | NFR-13 → Must. `model_version` aus US-02 entfernt und als NFR-12 (Should) belassen, statt NFR-12 komplett auf Must zu heben (sonst mehr Must) |
| 11 | CI/Tests, Quickstart-Test | **Übernommen** | NFR-08 → Must mit gemocktem LLM; Messlauf separat; Quickstart-Voraussetzungen beschrieben |
| 12 | Kosten/Anbieter offen, Injection, Timeout | **Teilweise** | NFR-14 (Should) für Timeout, Temperature 0, Injection-Test. Kostenlimit bleibt Should; U-6 als „vor Projektstart zu klären" markiert. Entscheidung über Anbieter und Budget liegt beim Team, nicht bei der KI |

Beobachtung zur Qualität von A3: Die Befunde waren überwiegend konkret und belegbar. In Befund 11 enthielt der Text allerdings eine selbstkorrigierende, verwirrende Zählpassage („5 Must-Stories … ergibt 6 – korrekt"); der Teil wurde nicht verwendet und die Zählung manuell geprüft (6 Must-Stories).

### 4.2 Annahmen der KI, die das Team bestätigen muss

| Annahme | Risiko |
|---|---|
| Zielwerte (90 % / 80 %, 60 s, 10 MB, 5 Seiten, 120 s Timeout, 30 Rechnungen) | Frei gewählt, ohne Messung oder Kundengespräch |
| MVP = Rechnung, Text-PDF, nur Kopfdaten und Summen | Plausibel, aber nicht aus Nutzerdaten abgeleitet |
| Zielgruppe „Entwickler:innen" als Primärnutzer | Annahme zu B2B-Integration |
| LLM-Anbieter, Budget, Hosting | Offen (U-6) |

### 4.3 Allgemeine Schwächen KI-gestützter Anforderungsanalyse

- **Plausibel statt geprüft:** Die KI kennt weder Budget, Team noch Kunden; Zahlen und Scope sind Vorschläge.
- **Vollständigkeitsillusion:** Eine lange Liste wirkt vollständig. Die erste Fassung hatte trotz Länge zu viele Must-Anforderungen und kein `mvp.md`; erst der Abgleich mit der Aufgabenbeschreibung (A2) hat das sichtbar gemacht.
- **Gleiche Familie von Modellen:** A3 war eine unabhängige Sitzung, aber dasselbe Modell wie A1/A2. Gemeinsame blinde Flecken bleiben möglich; ein Review durch das Team ist weiterhin nötig.

## 5. Reflexion

1. **Übernommen / geändert / verworfen:** Übernommen wurde der Gesamtaufbau und der Großteil der Befunde der unabhängigen Prüfung (9 von 12 vollständig, 3 teilweise, siehe 4.1). Geändert wurde vor allem der Umfang: Der Erstentwurf war für ein Semester zu groß und wurde auf einen Rechnungs-Anwendungsfall reduziert.
2. **Gefundene und übersehene Lücken:** Die unabhängige Prüfung fand konkrete Lücken, die im Erstentwurf fehlten, etwa die widersprüchliche Einordnung der Summenprüfung, das undefinierte Verhalten bei `failed`-Jobs und einen unklaren Löschumfang. Der Abgleich mit der Aufgabenbeschreibung deckte ein fehlendes Deliverable (`mvp.md`) und zu viele Must-Anforderungen auf.
3. **Hilfreich / generisch:** Hilfreich war die schnelle Struktur und die Formulierung testbarer Akzeptanzkriterien. Generisch sind Teile der nichtfunktionalen Anforderungen .
4. **Wirkung präziserer Prompts:** Der erste Prompt war die Aufgabenstellung und lieferte einen breiten, aber zu großen Entwurf. Der Prüf-Prompt (A3) war enger formuliert und lieferte deutlich konkretere, umsetzbare Befunde.
5. **Beim nächsten Mal anders:** Erst Kontext liefern, bevor die KI Zahlen vorschlägt.
