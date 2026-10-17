# AGENTS.md

Zweck
-----
Diese Datei dokumentiert, wie die clientseitige Tagesaufzeichnung, Historie und Visualisierung in der Web‑UI (index.html) funktionieren. Sie ist speziell dafür gedacht, dass andere Agents (oder Menschen) schnell verstehen, wie Tage erfasst, wie die Diagramme aufgebaut werden und wo sich die wichtigsten Hooks befinden.

Kurz: Ja — diese Datei darf von anderen Agents gelesen und editiert werden. Wenn ein Agent Änderungen vornimmt, bitte kurz in einer Änderungskommentierung (Commit‑Message / PR‑Beschreibung) erklären, was geändert wurde und warum.

Wo im Repo
---------
- Hauptimplementierung: `index.html` (Client‑side JavaScript)
- Visualisierung: Plotly (via CDN im Kopf der Seite)

Wichtigste DOM‑IDs / UI‑Elemente
------------------------------
- #nextDay — Button, der den Tag abschließt und die Simulation für den nächsten Tag ausführt.
- #reset — Reset/Neustart des Spiels.
- #productFilter — Dropdown zum Auswählen eines Produkts für die Tages‑Totals.
- #showAll — Button, um wieder "Alle Produkte" auszuwählen.
- #inventoryPlot — Container für die Liniengrafik (Bestand am Tagesende über Tage).
- #totalsPlot — Container für die Balkendiagramme (produziert / verkauft / konsumiert pro Tag).

Datenmodell (clientseitig)
--------------------------
Die Seite hält den Zustand komplett im `state` Objekt (keine Server‑Persistenz). Relevante Felder:

- state.day: laufender Tag (Number)
- state.people: Array der Bewohner (name, job, money, stock, needsFulfilled)
- state.log: Array von Strings (Ereignislog)
- state.history: Array mit abgeschlossenen Tagen (siehe unten)

Jeder Eintrag in state.history hat die Form:
{
  day: Number,
  events: [ { idx: Number, event: 'produce'|'sell'|'consume', product: String, qty: Number } ],
  samples: [ { idx: Number, inventory: { <product>: Number, ... } } ],
  totals: { <product>: { produced: Number, sold: Number, consumed: Number }, ... }
}

Temporäre Tagesaufzeichnung (während eines Tages)
-------------------------------------------------
- state._dayEvents: Array wie oben, aber nur für laufenden Tag
- state._daySamples: Array von Snapshots (inventory) nach jedem strukturierten Ereignis

Wichtigste Funktionen (Hooks)
----------------------------
- startDayRecording(): Initialisiert die temporären Arrays und speichert den Tagesanfangs‑Snapshot.
- recordEvent(type, product, qty): Wird an den Stellen aufgerufen, an denen die Simulation Verkäufe/Produktion/Konsum erzeugt. Fügt ein strukturiertes Ereignis hinzu und speichert einen Markt‑Snapshot.
- endDayRecording(): Aggregate berechnen (totals), history pushen und die Graphen neu rendern.
- renderDayGraphs(): Baut die Multi‑Day‑Plots (inventoryPlot und totalsPlot) aus state.history; nutzt Plotly.

Wo recordEvent aktuell aufgerufen wird
-------------------------------------
- Nach erfolgreichem Kauf in buyRawMaterials(): recordEvent('sell', material, 1)
- Nach Produktion in produceGoods(): recordEvent('produce', output, outputAmount)
- Beim Kauf + sofortigem Verbrauch (Kauf→Konsum) in consumeGoods(): recordEvent('sell', product, 1) und recordEvent('consume', product, 1)

Mobile‑Optimierungen
--------------------
- Responsive CSS: Charts stapeln sich vertikal auf kleinen Bildschirmen; Schriftgrößen & Paddings reduziert.
- Plotly Konfiguration: `responsive: true` und `displayModeBar` wird auf mobilen Geräten ausgeblendet.
- Linienbreiten und Font‑Sizes werden für mobile (width <= 600px) reduziert.

Performance‑ und Skalierungshinweise
------------------------------------
- state.history kann bei sehr vielen Tagen groß werden. Vorschläge:
  - Trunkiere ältere Tage (z. B. nur letzte N Tage in `state.history`, ältere in `localStorage` oder exportieren).
  - Aggregiere ältere Tage (Woche/Monat) und zeichne nur aggregierte Werte.
  - Bei vielen Produkten: zeige nur Top‑N Produkte oder nutze Lazy‑Loading / Toggle‑Ebenen.
- Plotly: Bei sehr vielen Punkten/Serien auf WebGL‑Traces (scattergl) umstellen.

Erweiterungsmöglichkeiten
------------------------
- CSV/JSON Export: `state.history` als Datei exportieren (Beispiel: JSON.stringify(state.history)).
- Persistenz: Speichern in localStorage oder Backend‑API (POST /history)
- Drilldown: Klick auf Balken → Liste der Ereignisse des Tages anzeigen (state.history[dayIndex].events)
- Mehrere Dörfer: Falls es in Zukunft mehrere Dörfer gibt, sollte history auf Dorf‑Ebene partitioniert werden.

Testing / Manuelles QA
----------------------
Empfohlene Tests (manuell im Browser, auch auf iPhone):
1. Seite öffnen
2. Mehrmals auf "Nächster Tag" klicken (mindestens 5×)
3. Prüfen: Diagramme zeigen je Tag einen Punkt / Balken; Legende und Achsenbeschriftungen sind lesbar.
4. Produktfilter testen (ein Produkt auswählen, "Alle Produkte" zurücksetzen)
5. Auf kleinerem Bildschirm prüfen: Bedienbarkeit, keine überlappenden Legenden

Berechtigungen / Agents
-----------------------
- Diese Datei darf von anderen Agents gelesen und editiert werden.
- Bitte: Wenn ein Agent die Datei ändert, schreibe eine kurze Zusammenfassung in die Commit‑Message / PR‑Beschreibung mit:
  - Name (Agent oder Person)
  - Datum
  - Kurze Erklärung der Änderung

Kontakt / Hinweise
------------------
Wenn etwas nicht wie erwartet funktioniert (z. B. fehlende Produktnamen oder nicht korrekte Summen), öffne bitte ein Issue oder füge einen Commit mit reproduzierbarem Testfall hinzu.


---
Kleiner Hinweis: Die Implementierung ist bewusst clientseitig gehalten, damit die Seite offline funktioniert. Falls persistente Analysen oder Exporte benötigt werden, empfehle ich ein kleines Backend (z. B. ein JSON‑Store oder GitHub Pages + manual export) zu ergänzen.
