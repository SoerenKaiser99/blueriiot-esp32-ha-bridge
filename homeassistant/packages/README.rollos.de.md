# Rollos per Alexa: „Guten Morgen" / „Gute Nacht" + Hitzeschutz

Diese Anleitung gehört zu [`rollos.yaml`](rollos.yaml). Sie richtet zwei
Sprachbefehle ein und enthält am Ende eine Checkliste, warum eine bestehende
Einrichtung „nicht mehr richtig funktioniert".

| Sprachbefehl | Was passiert |
|---|---|
| „Alexa, guten Morgen" | Terrassentür **immer** ganz auf. Alle anderen Rollos auf – **außer** sie stehen wegen Hitzeschutz auf 50 %, dann bleiben sie stehen. |
| „Alexa, gute Nacht" | Alle Rollos zu. Hitzeschutz-Merker wird zurückgesetzt. |
| (automatisch) | Außentemperatur > 25 °C für 10 min, tagsüber → Hitzeschutz-Rollos auf 50 %, Merker `input_boolean.rollos_hitzeschutz` an. Unter 23 °C für 30 min → Merker aus, tagsüber wieder ganz auf. |

Der Merker ist der entscheidende Baustein: Das Morgen-Skript kann nicht an der
Position 50 % allein erkennen, *warum* ein Rollo dort steht. Deshalb setzt die
Hitzeschutz-Automation einen Merker, und das Morgen-Skript prüft ihn.

---

## 1. Package installieren

1. In `configuration.yaml` (falls noch nicht vorhanden):

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

2. `rollos.yaml` nach `/config/packages/rollos.yaml` kopieren.
3. Die mit `# <-- anpassen` markierten Zeilen ändern:
   - beide Cover-Gruppen (`Rollos Alle`, `Rollos Hitzeschutz`)
   - `terrassentuer:` im Skript `rollos_guten_morgen`
   - `sensor.aussentemperatur` in beiden Automationen
4. **Entwicklerwerkzeuge → YAML → Konfiguration prüfen**, dann HA neu starten
   (Cover-Gruppen werden nur beim Start angelegt).
5. Prüfen, dass diese Entitäten existieren:
   `cover.rollos_alle`, `cover.rollos_hitzeschutz`,
   `input_boolean.rollos_hitzeschutz`, `script.rollos_guten_morgen`,
   `script.rollos_gute_nacht`.

## 2. Skripte ohne Alexa testen

**Entwicklerwerkzeuge → Aktionen** → `script.rollos_guten_morgen` → Ausführen.
Danach **Einstellungen → Automationen & Szenen → Skripte → ⋮ → Ablaufverfolgung**
öffnen. Dort sieht man pro Schritt, welche Rollos angesprochen wurden.
Das Skript schreibt außerdem eine Zeile ins Logbuch („Hitzeschutz aktiv – …"
bzw. „Alle Rollos geöffnet").

Hitzeschutz simulieren: `input_boolean.rollos_hitzeschutz` einschalten, dann
das Morgen-Skript starten → nur die Terrassentür und die Nicht-Hitzeschutz-Rollos
dürfen fahren.

Erst wenn das hier funktioniert, weiter mit Alexa. Sonst sucht man den Fehler an
der falschen Stelle.

## 3. Für Alexa freigeben

Voraussetzung: Home Assistant Cloud (Nabu Casa) oder ein selbst gehosteter
Alexa-Smart-Home-Skill.

1. **Einstellungen → Sprachassistenten → Alexa → Entitäten freigeben**:
   `script.rollos_guten_morgen` und `script.rollos_gute_nacht` freigeben.
   (Alternativ in YAML: `alexa: → filter: → include_entities:`.)
2. „Alexa, suche neue Geräte" sagen oder in der Alexa-App unter *Geräte* die
   Suche anstoßen. Die beiden Skripte tauchen in Alexa als **Szenen** auf.
3. Alexa-App → *Mehr → Routinen → +*:
   - **Wenn:** Sprache → `guten Morgen`
   - **Aktion:** Smart Home → *Szenen* → „Rollos: Guten Morgen"
4. Dasselbe für `gute Nacht` → „Rollos: Gute Nacht".

> **Hinweis:** „Guten Morgen" und „Gute Nacht" sind auch eingebaute
> Alexa-Phrasen. Eine eigene Routine mit exakt diesem Auslöser hat Vorrang – aber
> nur, wenn der Auslöser wirklich wortgleich ist (ohne „Alexa," davor).

---

## 4. Debug-Checkliste: „Es hat mal funktioniert, jetzt nicht mehr"

In dieser Reihenfolge prüfen – die ersten drei Punkte sind mit Abstand am häufigsten.

1. **Alexa zeigt auf eine Entität, die es nicht mehr gibt.**
   Wurde ein Skript/eine Szene umbenannt, gelöscht oder neu angelegt, hält
   Alexa oft einen „toten" Eintrag. Symptom: Alexa sagt „OK", nichts passiert,
   in HA gibt es **keine** Ablaufverfolgung. Lösung: In der Alexa-App die alte
   Szene löschen, „Alexa, suche neue Geräte", Routine neu auf die Szene zeigen.

2. **Das Skript wird gar nicht ausgelöst.**
   Skript → Ablaufverfolgung: Gibt es einen Eintrag zur Uhrzeit des Befehls?
   - Nein → Problem liegt bei Alexa/Freigabe (Punkt 1, Punkt 3).
   - Ja, aber Fehler → Punkt 4.

3. **Entität nicht (mehr) freigegeben.**
   Nach einem HA-Update oder Umstellung auf die neue „Sprachassistenten"-Seite
   wird eine YAML-Filterliste teilweise ignoriert. Unter *Einstellungen →
   Sprachassistenten → Alexa* nachsehen, ob die Skripte wirklich auf „freigegeben" stehen.

4. **Cover-Entity-IDs haben sich geändert.**
   Typisch nach Neu-Anlernen eines Shelly/Zigbee-Aktors oder Umbenennen im
   Geräte-Dialog. Ablaufverfolgung zeigt dann „entity not found" oder das
   Rollo fehlt einfach. Lösung: IDs in den beiden Cover-Gruppen korrigieren.
   Vorteil des Packages: Die IDs stehen nur noch an **einer** Stelle.

5. **Hitzeschutz-Merker hängt auf „an".**
   Wenn der Merker nie zurückgesetzt wird (z. B. weil die alte Version keinen
   Reset hatte), bleiben die Rollos jeden Morgen stehen. In dieser Version
   setzen ihn sowohl „Gute Nacht" als auch die Abkühl-Automation zurück.
   Manuell: `input_boolean.rollos_hitzeschutz` ausschalten.

6. **Skript im Modus `single` läuft noch.**
   Bleibt ein Rollo-Befehl hängen (Aktor offline), kann ein Skript mit
   `mode: single` beim nächsten Aufruf „already running" melden. Im Protokoll
   (*Einstellungen → System → Protokolle*) nach `rollos_` suchen.

7. **Temperatursensor liefert `unavailable`/`unknown`.**
   Dann feuert die Hitzeschutz-Automation nie. Sensor im Verlauf prüfen.

8. **Alexa versteht eine Variante.**
   „Guten Morgen Alexa", „Alexa, einen guten Morgen" … treffen den Auslöser
   nicht. In der Routine können mehrere Auslöser-Phrasen hinterlegt werden.

---

## Anpassungen

- **Alle Rollos bei Hitze halb runter:** In `Rollos Hitzeschutz` dieselbe Liste
  wie in `Rollos Alle` eintragen.
- **Andere Schwellen:** `above: 25` / `below: 23` und die `for:`-Zeiten ändern.
  Die Hysterese (25 / 23) verhindert Hin-und-her-Fahren um 25 °C herum.
- **Nachts kein Hitzeschutz:** Ist bereits so (`sun.sun` muss `above_horizon` sein).
- **Terrassentür bei Hitze ganz oben lassen:** Sie einfach aus
  `Rollos Hitzeschutz` entfernen.
