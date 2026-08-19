# Zähler-Blinker

Web-App, die die Konfigurationssequenz eines Landis+Gyr E220 als Lichtimpulse
über die optische Schnittstelle (INFO-DSS) sendet — statt sie von Hand mit
einer Taschenlampe einzublinken.

Sprache der Oberfläche und aller Texte: **Deutsch**.

## Dateien

- `index.html` (bzw. `zaehler-blinker.html`) — die gesamte App: eine Datei,
  kein Build, keine Abhängigkeiten. Inline-CSS, Inline-JS.

## Ziel der App

Am E220 sollen zwei Einstellungen gesetzt werden:

- **InF → on** — schaltet den erweiterten Datensatz auf der optischen
  INFO-Schnittstelle frei. Erst dann liefert der Zähler `1-0:16.7.0`
  (Momentanleistung). Ohne das sieht ein IR-Lesekopf nur Zählerstände.
- **PIn → oFF** — deaktiviert die PIN-Abfrage, damit die Momentanleistung
  dauerhaft sichtbar bleibt.

Hintergrund: Der Zähler wird von einem Zendure Smart Meter D0 (optischer
Lesekopf) ausgelesen, für die Nulleinspeisungsregelung einer Garagen-PV mit
Batterie. Die Momentanleistung ist dabei die Regelgröße.

## Protokoll der optischen Schnittstelle (E220)

Quelle: Landis+Gyr Benutzerhandbuch E220-AMxD, Dokument D000063497,
Abschnitte 2.5.3 und 5.4.1. Ergänzend die Kurz-Bedienungsanleitung
(dort „GIN" statt „PIN") und das volkszaehler-Wiki.

| Größe | Wert |
|---|---|
| Sensorposition | rechte obere Ecke der Frontplatte |
| Mindesthelligkeit | 400 Lux |
| kurzer Impuls | < 2 s |
| langer Impuls | > 5 s |
| Ziffernübernahme | nach ca. 3 s ohne Impuls |
| Rückfall auf Standardanzeige | nach 120 s Untätigkeit |

Ablauf: 1 kurzer Impuls öffnet das Menü (Displaytest), dann vierstellige PIN —
je Ziffer *n* kurze Impulse, danach Ziffernpause. Ziffer 0 = kein Impuls, nur
warten. Bei korrekter PIN zeigt das Display `P`.

Menüreihenfolge ab `P`, je 1 kurzer Impuls weiter:

```
P → E → 1d → 7d → 30d → 365d → HIS → InF → PIn
```

`InF` und `PIn` sind **Umschalter**: Ein langer Impuls kippt den aktuellen
Zustand, er setzt keinen Sollwert.

## Bekannte Unsicherheiten — bitte nicht wegoptimieren

1. **Die Menüreihenfolge ist nicht verifiziert.** Sie stammt aus einem
   Ablaufdiagramm im PDF, dessen Textextraktion lückenhaft ist. Zusätzlich
   hängt es von Firmwareparametern des Messstellenbetreibers ab, welche
   Menüpunkte überhaupt erscheinen — sind die historischen Register
   deaktiviert, entfallen vier Positionen und die Zählung stimmt nicht mehr.
2. **Ein langer Impuls an falscher Stelle ist destruktiv.** In den Registern
   `1d`, `7d`, `30d`, `365d` und bei `HIS` löscht er die dortigen Werte.
   Das sind fünf empfindliche Positionen auf dem Weg zu `InF`.

Deshalb hält die App **vor jedem langen Impuls** an und fragt, was auf dem
Display steht. Diese Haltepunkte sind eine Sicherheitsmaßnahme, keine
Bequemlichkeitslücke. Wer sie entfernen will, muss vorher die Menüreihenfolge
am konkreten Gerät verifiziert haben.

## Lernmodus

Räumt Unsicherheit 1 aus, indem er sie misst statt sie zu glauben. Statt in
der Menüphase blind sieben Impulse zu zählen, sendet er **einen** kurzen
Impuls, hält an und fragt über ein Raster der bekannten Menüpunkte, was
tatsächlich im Display steht — bis der Nutzer `InF` meldet (bzw. `PIn`, wenn
nur die PIN-Abfrage abgeschaltet werden soll).

Tragende Überlegungen, bitte beim Umbauen erhalten:

- **Die Segmentanzeige sagt im Lernmodus nichts vorher** (`- - -`). Zeigt sie
  den erwarteten Menüpunkt, bestätigt der Nutzer die Erwartung statt der
  Realität — dann misst der Modus nichts.
- **Ein kurzer Impuls ist folgenlos.** Deshalb bricht „etwas anderes" hier
  *nicht* ab, sondern wird als `?` protokolliert und weitergezählt. Der
  Lernmodus entscheidet nie über einen langen Impuls; das bleibt der
  bestehenden Abfrage vor `InF`/`PIn` überlassen.
- **„Standardanzeige"** ist eine eigene Antwortmöglichkeit. Nach 120 s ohne
  Impuls fällt der Zähler zurück, und die App kann das nicht selbst merken —
  ohne diesen Knopf wird der Nutzer in eine falsche Antwort gedrängt.
- Die 120 s laufen **zwischen zwei Impulsen**, nicht kumulativ; jeder
  Haltepunkt hat sein eigenes Budget. Ein Zähler in der Ablaufansicht zeigt
  den Abstand, ab 90 s mit Warnton.
- **Zurück gibt es nicht** — das Menü kennt nur Vorwärtsimpulse. Korrigiert
  wird vorwärts oder durch Neustart nach 120 s.
- Deckel bei 20 Schritten. Erscheint `InF` bis dahin nicht, hat der
  Messstellenbetreiber den Punkt vermutlich nicht freigegeben. Auch das ist
  ein Ergebnis, kein Fehler.

Das Ergebnis landet als Pfad-Array unter dem localStorage-Schlüssel
`zb.menupfad.v1` (nur die Menüreihenfolge — **die PIN nie**). Ab dem zweiten
Lauf ist der Lernmodus vorbelegt aus, der blinde Durchlauf nutzt die gelernte
Schrittzahl und beschriftet die Zwischenschritte damit; der Textplan ebenso.
„Gelerntes vergessen" in der Einrichtung setzt zurück.

Ebenfalls unklar: ob die Einstellungen einen Stromausfall überleben. Der
Firmwareparameter „Endnutzereinstellungen Stromausfall" wird vom
Messstellenbetreiber gesetzt und ist nicht änderbar. Die Kurzanleitung sagt,
nach einem Spannungsausfall sei die Eingabe grundsätzlich zu wiederholen.

## Lichtquelle: der harte Teil

Die App kann auf drei Wegen blinken:

- **Bildschirm** — Vollflächiges Weiß, funktioniert überall, braucht keine
  Berechtigungen. Physikalisch unkritisch: eine aufliegende Leuchtfläche
  liefert etwa π × Leuchtdichte, beim pOLED also grob vierstellig Lux
  gegen 400 gefordert, und praktisch abstandsunabhängig.
- **Kamera-Blitz (Torch)** — über `getUserMedia` +
  `applyConstraints({advanced:[{torch:true}]})`.
- **Ablaufplan** — reine Textausgabe zum Abarbeiten mit echter Taschenlampe.

### Torch braucht einen echten sicheren Kontext

Verifiziert auf einem Motorola Razr 50 Ultra, Chrome für Android:

| Herkunft | Ergebnis |
|---|---|
| `file://` | keine Nachfrage, `mediaDevices` nicht nutzbar |
| Claude-Artifact-Iframe (`about://`) | `NotAllowedError: Permission denied`, keine Nachfrage — dem Iframe fehlt `allow="camera"` |
| veröffentlichtes Claude-Artifact | identisch, ebenfalls Iframe |
| Netlify Drop (`https://…netlify.app`) | **funktioniert**, Kameraabfrage erscheint, Blitz schaltet |

Reines `http://` im LAN reicht nicht, `localhost` dagegen schon.
Ein selbstsigniertes Zertifikat reicht ebenfalls **nicht**.

### Implementierungsdetails, die nötig waren

- Der Stream hängt an einem versteckten `<video>`, das tatsächlich abspielt.
  Ohne das bleibt der Blitz auf etlichen Android-Geräten dunkel, auch wenn
  `applyConstraints` Erfolg meldet.
- `facingMode: environment` liefert bei Multikamera-Geräten oft eine Linse
  ohne Torch-Fähigkeit. Die App prüft `getCapabilities().torch` und geht bei
  Fehlanzeige alle `videoinput`-Geräte einzeln durch.
- Der Knopf „Blitz testen" gibt Herkunft, `isSecureContext`, Verfügbarkeit von
  `mediaDevices` und Iframe-Status aus. Das ist das Diagnosewerkzeug — bei
  Torch-Problemen zuerst dorthin schauen.

## Weitere Bausteine

- **7-Segment-Nachbau** in CSS (kein Font), zeigt an, was der Zähler im
  jeweiligen Schritt anzeigen sollte. Segmentkarte in der Konstanten `SEG`.
- **Ansagen und Signaltöne** über WebAudio und `speechSynthesis` (`de-DE`),
  weil das Display beim Bildschirmmodus auf dem Sensor liegt und nicht
  lesbar ist.
- **Wake Lock**, damit der Bildschirm während des Ablaufs anbleibt.
- Alle Zeiten sind in der Oberfläche einstellbar.

Die PIN wird **nicht** gespeichert und steht nirgends im Quelltext — sie wird
bei jedem Lauf eingetippt. Das soll so bleiben; die App darf öffentlich
gehostet werden.

## Offene Punkte

- [x] Lernmodus — gebaut, am Gerät noch nicht erprobt.
- [ ] Bei „Etwas anderes" an den **Umschalt**-Haltepunkten (`InF`, `PIn`) nicht
      abbrechen, sondern korrigieren lassen. In der Menüphase erledigt der
      Lernmodus das; dort ist Weiterzählen gefahrlos.
- [ ] Fallback über die `ImageCapture`-API, falls `applyConstraints` auf einem
      Gerät scheitert.
- [ ] Praxistest am Zähler steht noch aus. Erfolgskontrolle ist nicht das
      Display, sondern ob der Zendure-Lesekopf eine Wattzahl liefert.
