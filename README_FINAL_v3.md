# Wallbox-Steuerung – Home Assistant

Eigene PV-Regelung für eine **dé/Tuya-Wallbox (11 kW, dreiphasig)** in Home Assistant mit **Shelly Pro 3EM** und **BYD Atto 3 EVO**.

Die Lösung nutzt die Integration [lachand/EV_charger](https://github.com/lachand/EV_charger) für die lokale Kommunikation mit der Wallbox, übernimmt die eigentliche PV-Regelung aber bewusst selbst in Home Assistant.

Geprüft gegen **Tuya EV Charger Local v2.25.0** und Home Assistant **2026.9.x**.

---

## Ziel des Projekts

Die Regelung soll:

- PV-Überschuss möglichst gut fürs Auto nutzen,
- bei Bedarf bewusst einen begrenzten Netzanteil zulassen,
- dreiphasig korrekt rechnen,
- nicht bei jeder Wolke hoch und runter springen,
- Sofortladen mit Vorrang ermöglichen,
- eine PV-Pause unterstützen,
- das Ladeverhalten nachvollziehbar protokollieren,
- eine eigene AC-Ladekurve aufzeichnen,
- erkennen, wenn das Auto zwar angeschlossen ist, aber keine Ladung mehr annimmt,
- und verhindern, dass bei vollem Auto immer wieder neue PV-Startversuche ausgelöst werden.

---

## Inhalt

1. [Dateien](#1-dateien)
2. [Systemarchitektur](#2-systemarchitektur)
3. [Voraussetzungen](#3-voraussetzungen)
4. [Installation](#4-installation)
5. [Wichtige Entities](#5-wichtige-entities)
6. [Wie die PV-Regelung funktioniert](#6-wie-die-pv-regelung-funktioniert)
7. [Dreiphasige Stromberechnung](#7-dreiphasige-stromberechnung)
8. [PV-Überschuss richtig berechnen](#8-pv-überschuss-richtig-berechnen)
9. [Start-, Hoch-, Runter- und Stopplogik](#9-start--hoch--runter--und-stopplogik)
10. [PV-Schwellen und bewusster Netzbezug](#10-pv-schwellen-und-bewusster-netzbezug)
11. [Sofortladen und PV-Pause](#11-sofortladen-und-pv-pause)
12. [Auto voll / keine Ladeannahme](#12-auto-voll--keine-ladeannahme)
13. [AC-Ladekurve](#13-ac-ladekurve)
14. [Regelungslog und Diagnose](#14-regelungslog-und-diagnose)
15. [BYD-Verhalten und Plausibilitätsprüfung](#15-byd-verhalten-und-plausibilitätsprüfung)
16. [Tuya-DPs und technische Erkenntnisse](#16-tuya-dps-und-technische-erkenntnisse)
17. [Bekannte Einschränkungen](#17-bekannte-einschränkungen)
18. [Fehlersuche](#18-fehlersuche)
19. [Lessons Learned](#19-lessons-learned)

---

## 1. Dateien

| Datei | Ziel | Zweck |
|---|---|---|
| `wallbox_package_FINAL_v3_mit_logbuch_ac_ladekurve_und_vollerkennung.yaml` | `/config/packages/wallbox.yaml` | Sensoren, Helper, Scripts, Timer, Automationen, Logging |
| `zuhause_dashboard_FINAL_v3_mit_logbuch_ac_ladekurve_und_vollerkennung.yaml` | Dashboard → Rohdaten-Konfigurationseditor | Komplettes Dashboard „Zuhause“ |
| `README_FINAL_v3.md` | beliebig | Dokumentation dieser Lösung |

**Wichtig:** Package-YAML und Dashboard-YAML sind zwei unterschiedliche Dateien. Die Package-Datei darf nicht in den Dashboard-Rohdateneditor kopiert werden.

---

## 2. Systemarchitektur

```text
PV-Anlage
   │
   ▼
Hausnetz ───────────────► Verbraucher
   │
   ├── Shelly Pro 3EM  ─────► Home Assistant
   │                           │
   │                           ├── berechnet verfügbaren PV-Überschuss
   │                           ├── berechnet Zielstrom 3-phasig
   │                           ├── entscheidet Start / Hoch / Runter / Stop
   │                           ├── protokolliert Regelentscheidungen
   │                           └── steuert Tuya-Wallbox lokal
   │
   └────────────────────────► Tuya-Wallbox ─────► BYD Atto 3 EVO

Netz:
+ = Bezug
- = Einspeisung
```

Grundprinzip: **Home Assistant ist die einzige führende PV-Regelinstanz.**

Der interne PV-Regler der Tuya-Integration bleibt deshalb ausgeschaltet:

```text
switch.tuya_ev_charger_solar_surplus_mode = off
```

So vermeiden wir zwei gleichzeitig arbeitende Regler.

---

## 3. Voraussetzungen

### Hardware

- dreiphasige 11-kW-Tuya-Wallbox
- Shelly Pro 3EM am relevanten Netz-/Messpunkt
- PV-Anlage
- Home Assistant
- getestetes Fahrzeug: BYD Atto 3 EVO

### Software

- Home Assistant mit Packages
- Integration **Tuya EV Charger Local v2.25.0**
- optional für das Dashboard: Mushroom / card-mod / Bubble Card, soweit im bestehenden Dashboard verwendet

### Relevante Integrations-Entities

Mindestens:

```text
sensor.tuya_ev_charger_status
sensor.tuya_ev_charger_total_power
sensor.tuya_ev_charger_session_energy
sensor.tuya_ev_charger_alarm
number.tuya_ev_charger_current_setpoint
number.tuya_ev_charger_surplus_start_threshold
number.tuya_ev_charger_surplus_stop_threshold
switch.tuya_ev_charger_solar_surplus_mode
switch.tuya_ev_charger_charging_session
switch.garage_tuya_ev_charger_force_charge
```

Netzmessung:

```text
sensor.shellypro3em_38182bdee0e0_leistung
```

Vorzeichen des Shelly in diesem Setup:

```text
positiv  = Netzbezug
negativ  = Einspeisung
```

---

## 4. Installation

### 4.1 Packages aktivieren

In `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 4.2 Package kopieren

Die Datei

```text
wallbox_package_FINAL_v3_mit_logbuch_ac_ladekurve_und_vollerkennung.yaml
```

als

```text
/config/packages/wallbox.yaml
```

speichern.

### 4.3 Konfiguration prüfen

Home Assistant:

```text
Entwicklerwerkzeuge → YAML → Konfiguration prüfen
```

Danach Home Assistant **vollständig neu starten**.

### 4.4 Dashboard einspielen

Nur die Datei

```text
zuhause_dashboard_FINAL_v3_mit_logbuch_ac_ladekurve_und_vollerkennung.yaml
```

in:

```text
Dashboard → Bearbeiten → Rohdaten-Konfigurationseditor
```

kopieren.

### 4.5 Nach dem Neustart prüfen

Unter `Entwicklerwerkzeuge → Zustände` sollten unter anderem existieren:

```text
input_boolean.wallbox_pv_automatik
input_boolean.wallbox_auto_nimmt_keine_ladung_an
timer.wallbox_pv_pause
timer.wallbox_ladeannahme_test
sensor.wallbox_pv_ueberschuss_fuer_auto
sensor.wallbox_pv_zielstrom_3_phasig
sensor.wallbox_ac_ladekurve_leistung
sensor.wallbox_ac_ladekurve_pv_verfuegbar
sensor.wallbox_ac_ladekurve_netzfluss
```

---

## 5. Wichtige Entities

### Status und Regelung

| Entity | Bedeutung |
|---|---|
| `binary_sensor.wallbox_auto_angeschlossen` | Auto steckt an der Wallbox |
| `binary_sensor.wallbox_ladung_aktiv` | reale Ladeleistung ≥ 0,1 kW |
| `binary_sensor.wallbox_pv_ladung_aktiv` | Ladung läuft über eigene PV-Automatik |
| `binary_sensor.wallbox_pv_pausiert` | PV-Pause aktiv |
| `binary_sensor.wallbox_fehler` | Fehlerstatus / Alarm |
| `sensor.wallbox_status` | übersetzter Wallbox-Status |
| `input_boolean.wallbox_pv_automatik` | eigene PV-Regelung Ein/Aus |
| `input_boolean.wallbox_auto_nimmt_keine_ladung_an` | Sperre nach fehlgeschlagener Ladeannahme |
| `timer.wallbox_pv_pause` | temporäre PV-Pause |
| `timer.wallbox_ladeannahme_test` | 90-s-Test nach PV-Start |

### Leistung und Zielstrom

| Entity | Bedeutung |
|---|---|
| `sensor.wallbox_pv_einspeisung` | aktuelle Einspeisung in W |
| `sensor.wallbox_pv_ueberschuss_fuer_auto` | für das Auto verfügbarer PV-Überschuss |
| `sensor.wallbox_pv_zielstrom_3_phasig` | eigener Zielstrom in A |
| `number.tuya_ev_charger_current_setpoint` | tatsächlich an Wallbox gesetzter Strom |

### Logging

| Entity | Bedeutung |
|---|---|
| `input_text.wallbox_letzte_regelaktion` | letzte Regelaktion mit Zeitstempel |
| `input_text.wallbox_letzte_regelwerte` | Messwerte zur letzten Regelaktion |
| `script.wallbox_regelung_log` | schreibt eigene Logbook-Einträge |

### AC-Ladekurve

| Entity | Bedeutung |
|---|---|
| `sensor.wallbox_ac_ladekurve_leistung` | tatsächliche AC-Leistung der Wallbox |
| `sensor.wallbox_ac_ladekurve_pv_verfuegbar` | verfügbarer PV-Anteil |
| `sensor.wallbox_ac_ladekurve_netzfluss` | Netzbezug (+) / Einspeisung (-) |
| `sensor.wallbox_ac_ladekurve_zielstrom` | berechneter Zielstrom |
| `sensor.wallbox_ac_ladekurve_sollstrom` | tatsächlich gesetzter Strom |

---

## 6. Wie die PV-Regelung funktioniert

Die interne PV-Regelung der Integration wird **nicht** verwendet.

Stattdessen entscheidet Home Assistant selbst:

1. Ist die eigene PV-Automatik eingeschaltet?
2. Ist das Auto angeschlossen?
3. Läuft kein Sofortladen?
4. Ist keine PV-Pause aktiv?
5. Ist das Fahrzeug nicht als „nimmt keine Ladung an“ gesperrt?
6. Ist genügend verfügbarer Überschuss vorhanden?
7. Welcher dreiphasige Zielstrom passt zum verfügbaren Überschuss?
8. Muss gestartet, hochgeregelt, runtergeregelt oder gestoppt werden?

Damit liegt die Regelhoheit vollständig in Home Assistant.

---

## 7. Dreiphasige Stromberechnung

Für dreiphasiges Laden gilt näherungsweise:

```text
P = U × I × 3
```

mit ca. 230 V pro Phase:

```text
P ≈ 230 × I × 3
P ≈ 690 × I
```

Daraus folgt:

```text
I ≈ P / 690
```

Beispiele:

| Verfügbarer Überschuss | Rechnerisch | Zielstrom |
|---:|---:|---:|
| 4.140 W | 6,0 A | 6 A |
| 4.830 W | 7,0 A | 7 A |
| 5.520 W | 8,0 A | 8 A |
| 6.210 W | 9,0 A | 9 A |
| 6.900 W | 10,0 A | 10 A |
| 11.040 W | 16,0 A | 16 A |

Die Regelung verwendet **1-A-Schritte zwischen 6 und 16 A**.

### Warum nicht nur 6 / 8 / 10 / 13 / 16 A?

DP 107 der Wallbox meldet:

```text
[6, 8, 10, 13, 16]
```

Das sind die Schnellwahlwerte der Tuya-App. Die Integration v2.25.0 dokumentiert jedoch, dass bei `continuous_current` auch ganzzahlige Zwischenwerte akzeptiert werden.

Deshalb nutzt die finale Regelung:

```text
6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16 A
```

---

## 8. PV-Überschuss richtig berechnen

Ein einfacher Blick auf die aktuelle Einspeisung reicht während des Ladens nicht aus.

Beispiel:

```text
PV-Erzeugung:             8,0 kW
sonstiger Hausverbrauch:  0,8 kW
Wallbox:                   5,5 kW
Netzeinspeisung:           1,7 kW
```

Würde man nur die 1,7 kW Einspeisung betrachten, würde die Regelung glauben, es sei kaum PV vorhanden und herunterregeln.

Deshalb wird die aktuell von der Wallbox aufgenommene Leistung wieder addiert.

Im Package:

```text
verfügbarer Überschuss = Wallbox-Leistung - Netzleistung
```

bei der verwendeten Vorzeichenkonvention:

```text
Netzleistung positiv = Bezug
Netzleistung negativ = Einspeisung
```

Das ist äquivalent zu:

```text
verfügbarer Überschuss = Einspeisung + aktuelle Wallbox-Leistung
```

Dadurch regelt sich die Wallbox nicht selbst herunter.

---

## 9. Start-, Hoch-, Runter- und Stopplogik

Die Regelung arbeitet bewusst nicht hektisch.

| Aktion | Verzögerung | Zweck |
|---|---:|---|
| Start | 60 s | nur starten, wenn der Überschuss stabil ist |
| Hochregeln | 60 s | kurze Sonnenlöcher nicht sofort ausnutzen |
| Runterregeln | 30 s | Netzbezug schneller reduzieren |
| Stop | 120 s | kurze Wolkenphasen überbrücken |

Grundprinzip:

```text
langsam hoch
schneller runter
noch langsamer komplett stoppen
```

Das verhindert unnötiges Takten und ständige Stromänderungen.

---

## 10. PV-Schwellen und bewusster Netzbezug

Die Start- und Stoppschwelle bleiben über die vorhandenen Integrations-Number-Entities einstellbar:

```text
number.tuya_ev_charger_surplus_start_threshold
number.tuya_ev_charger_surplus_stop_threshold
```

### Wichtig: Start unter 4,14 kW ist erlaubt

Die Wallbox benötigt bei 6 A dreiphasig ungefähr:

```text
230 V × 6 A × 3 ≈ 4.140 W
```

Wenn die Startschwelle z. B. auf 2.000 W steht, darf die Regelung trotzdem mit 6 A starten.

Dann gilt ungefähr:

```text
4.140 W Mindestladeleistung
-2.000 W PV
=2.140 W Netzanteil
```

Das ist in diesem Projekt **bewusst erlaubt**.

Ziel ist nicht zwingend „0 W Netzbezug“, sondern eine flexible Mischung aus PV und Netz.

Die Startschwelle kann deshalb im Dashboard frei angepasst werden.

---

## 11. Sofortladen und PV-Pause

### Sofortladen

Sofortladen hat Vorrang vor der PV-Regelung.

Auswahl:

```text
1 Stunde
3 Stunden
6 Stunden
Bis voll
```

Schnellwahl-Strom:

```text
6 A
8 A
10 A
13 A
16 A
```

Während Sofortladen greift die eigene PV-Regelung nicht ein.

Nach Ende des Sofortladens übernimmt die PV-Automatik wieder, sofern sie eingeschaltet ist.

### Stopp

- Tippen: Ladung stoppen
- Gedrückt halten: stoppen + PV-Pause für 2 Stunden

Die Pause wird über

```text
timer.wallbox_pv_pause
```

realisiert.

---

## 12. Auto voll / keine Ladeannahme

Ein wichtiger Sonderfall entsteht bei 100 % SoC:

```text
Auto steckt
PV ist vorhanden
Wallbox bietet Strom an
Auto nimmt aber keine Leistung mehr auf
```

Ohne Zusatzlogik würde Home Assistant immer wieder neue PV-Startversuche machen.

### Finale Lösung

Nach jedem echten PV-Start wird gestartet:

```text
timer.wallbox_ladeannahme_test
```

Dauer:

```text
90 Sekunden
```

Wenn danach weiterhin gilt:

- Auto ist angeschlossen,
- PV-Automatik ist aktiv,
- ausreichend Überschuss vorhanden,
- reale Ladeleistung liegt unter 300 W,

wird gesetzt:

```text
input_boolean.wallbox_auto_nimmt_keine_ladung_an = on
```

Folge:

- weitere PV-Starts werden gesperrt,
- Zielstrom geht auf 0 A,
- Dashboard zeigt „Auto voll / nimmt keine Ladung an“,
- Logbook dokumentiert den Vorgang.

Die Sperre wird beim Abstecken zurückgesetzt.

### Verifizierter Testablauf

Der reale Test zeigte:

```text
PV-Automatik Ein
→ nach 60 s PV-Start
→ 90-s-Ladeannahme-Test aktiv
→ AC-Leistung bleibt 0 kW
→ Timer läuft ab
→ Auto nimmt keine Ladung an = Ein
→ weitere PV-Starts gesperrt
```

Damit ist der 100-%-Fall sauber abgedeckt.

---

## 13. AC-Ladekurve

Home Assistant zeichnet eine eigene AC-Ladekurve auf.

Erfasst werden:

- tatsächliche Ladeleistung,
- verfügbarer PV-Überschuss,
- Netzbezug / Einspeisung,
- berechneter Zielstrom,
- tatsächlich gesetzter Strom.

Dadurch kann später nachvollzogen werden:

```text
Wie viel hat die Wallbox angeboten?
Wie viel hat das Auto tatsächlich aufgenommen?
Wann hat der BYD bei hohem SoC reduziert?
Wie viel PV war verfügbar?
Gab es gleichzeitig Netzbezug oder Einspeisung?
```

Die Kurve eignet sich insbesondere zur Beobachtung des Ladeendes bei 99–100 % SoC.

### Wichtig

Die Wallbox misst die AC-Leistung vor dem Fahrzeug.

Der BYD zeigt im Fahrerdisplay während des Ladens zusätzlich eine eigene Ladeleistung an. Diese kann etwas niedriger liegen, weil Fahrzeugverbrauch und Onboard-Charger-Verluste dazwischen liegen.

---

## 14. Regelungslog und Diagnose

Jede wichtige Regelentscheidung wird im Home-Assistant-Logbook dokumentiert.

Beispiele:

```text
PV-Start
Hochregeln
Runterregeln
Stop
Auto voll / keine Ladeannahme
```

Mitprotokolliert werden unter anderem:

```text
PV-Überschuss
Zielstrom
gesetzter Strom
AC-Leistung
Netzfluss
Session-Energie
```

Zusätzlich stehen die letzten Werte dauerhaft in:

```text
input_text.wallbox_letzte_regelaktion
input_text.wallbox_letzte_regelwerte
```

Das Dashboard zeigt sowohl die letzte Regelentscheidung als auch einen eigenen Regelungsverlauf.

---

## 15. BYD-Verhalten und Plausibilitätsprüfung

### Die Wallbox bietet nur an

Wenn Home Assistant 10 A setzt, bedeutet das:

```text
Das Auto darf maximal 10 A aufnehmen.
```

Es bedeutet nicht:

```text
Das Auto muss 10 A aufnehmen.
```

Der BYD entscheidet selbst über seine reale Leistungsaufnahme.

### Bei hohem SoC

Bei 99–100 % kann das Fahrzeug die Leistung reduzieren oder komplett beenden.

### BYD-App vs. reale Messung

Während der Tests zeigte die BYD-App zeitweise eine deutlich niedrigere Leistung als die Wallbox.

Die plausiblere Gegenprobe war:

```text
PV-Erzeugung
- Hausverbrauch
= Netzfluss
```

Diese Bilanz stimmte mit Shelly und Zweirichtungszähler überein.

### Empfohlene Referenzen

Für die Diagnose gleichzeitig beobachten:

1. Wallbox-Leistung in Home Assistant
2. Shelly-Netzfluss
3. physischer Zweirichtungszähler
4. BYD-Ladeleistung im Fahrerdisplay

Die BYD-App allein ist keine zuverlässige Echtzeitreferenz.

---

## 16. Tuya-DPs und technische Erkenntnisse

Ein lokaler TinyTuya-Statusdump zeigte unter anderem:

```text
DP 101
DP 102
DP 106
DP 107
DP 109
DP 150
DP 151
DP 152
DP 155
DP 156
DP 157
DP 188
DP 189
```

Beispiele:

```text
DP 107 = [6, 8, 10, 13, 16]
DP 109 = SLEEP
DP 150 = aktueller Stromwert
DP 152 = Maximalstrom
```

### Display-Standby

Es wurde kein separater DP für

```text
Display
Backlight
Brightness
Screen Off
Standby
```

gefunden.

`DP 109 = SLEEP` beschreibt den Betriebszustand der Wallbox, nicht das Ausschalten des Displays.

Damit ist ein Display-Standby über die aktuell sichtbaren Tuya-DPs nicht verfügbar.

### Sicherheitsprinzip

Unbekannte DPs wurden nicht blind beschrieben.

Grundsatz:

```text
Erst lesen und verstehen, dann schreiben.
```

Gerade bei einer Wallbox ist das wichtiger als experimentelles Ausprobieren unbekannter Steuerwerte.

---

## 17. Bekannte Einschränkungen

### 17.1 Neustart von Home Assistant

Ein bereits wahrer Template-Zustand erzeugt nach einem Neustart nicht zwingend sofort einen neuen Trigger.

Beispiel:

```text
Auto steckt bereits
PV-Automatik ist bereits an
genug Sonne ist bereits vorhanden
HA startet neu
```

Dann kann es vorkommen, dass die Startautomation erst wieder reagiert, nachdem sich einer der relevanten Zustände ändert.

Im Test konnte dies durch kurzes Aus-/Einschalten der PV-Automatik ausgelöst werden.

**Noch offene Verbesserung:** Nach Home-Assistant-Start automatisch eine Neubewertung der PV-Startbedingungen durchführen.

### 17.2 BYD-SoC nicht direkt in Home Assistant

Die Regelung kennt aktuell keinen direkten Fahrzeug-SoC.

Die Vollerkennung arbeitet deshalb bewusst über das reale Verhalten:

```text
Startversuch + 90 s + <300 W = keine Ladeannahme
```

### 17.3 Messwerte aktualisieren nicht exakt gleichzeitig

Shelly, Wallbox und BYD aktualisieren in unterschiedlichen Intervallen. Kleine Momentabweichungen sind normal.

### 17.4 Spannung ist nicht exakt 230 V

Die Formel `/690` ist eine praxisnahe Näherung. Die reale Spannung schwankt leicht, deshalb kann die tatsächliche kW-Leistung von `690 × A` abweichen.

---

## 18. Fehlersuche

| Problem | Prüfung / Lösung |
|---|---|
| Dashboard meldet „Entität nicht verfügbar“ | Package installiert? HA vollständig neu gestartet? |
| `input_boolean.wallbox_auto_nimmt_keine_ladung_an` fehlt | richtige V3-Package-Datei unter `/config/packages/wallbox.yaml`? |
| `timer.wallbox_ladeannahme_test` fehlt | Package nicht geladen / falsche Version |
| Auto steckt, PV reicht, aber nach HA-Neustart passiert nichts | bekannte Neustart-Einschränkung; PV-Automatik einmal aus/ein |
| Wallbox nicht erreichbar | prüfen, ob TinyTuya noch verbunden ist; Integration neu laden |
| Nach `release_connection` keine Daten | TinyTuya schließen, Freigabezeit abwarten oder Integration neu laden |
| Zielstrom passt nicht | `sensor.wallbox_pv_ueberschuss_fuer_auto / 690` gegenprüfen |
| Auto lädt mit mehr Netz als erwartet | Start-/Stoppschwelle und Zielstrom prüfen |
| Zielstrom = 0 trotz Sonne | Sperre `wallbox_auto_nimmt_keine_ladung_an` oder PV-Pause prüfen |
| Auto voll, aber neue Startversuche | Ladeannahme-Timer und Vollerkennungs-Automation prüfen |
| BYD-App zeigt andere kW als HA | Wallbox + Shelly + physikalischen Zähler vergleichen |
| Interner Tuya-PV-Regler ist an | ausschalten; nur eine Regelinstanz verwenden |

### Hilfreiche Diagnose-Entities

```text
sensor.wallbox_pv_ueberschuss_fuer_auto
sensor.wallbox_pv_zielstrom_3_phasig
number.tuya_ev_charger_current_setpoint
sensor.tuya_ev_charger_total_power
binary_sensor.wallbox_ladung_aktiv
input_boolean.wallbox_auto_nimmt_keine_ladung_an
timer.wallbox_ladeannahme_test
input_text.wallbox_letzte_regelaktion
input_text.wallbox_letzte_regelwerte
```

---

## 19. Lessons Learned

### 1. Eine Regelung braucht genau eine führende Instanz

Interner Tuya-Regler und Home Assistant dürfen nicht gleichzeitig versuchen, denselben Strom zu regeln.

**Lösung:** interne PV-Regelung aus, Home Assistant übernimmt vollständig.

### 2. Die Messgröße muss physikalisch korrekt sein

Nur die aktuelle Einspeisung zu betrachten reicht während einer laufenden EV-Ladung nicht.

Die eigene Wallbox-Leistung muss in den verfügbaren Überschuss zurückgerechnet werden.

### 3. Dreiphasig muss auch dreiphasig gerechnet werden

Für eine dreiphasige 11-kW-Wallbox ist `/230` falsch.

Die korrekte Näherung lautet:

```text
W / 690
```

### 4. Geräte-DPs nicht vorschnell als Hardwaregrenzen interpretieren

`[6, 8, 10, 13, 16]` sah zunächst wie eine feste Stromstufung aus.

Tatsächlich sind 1-A-Schritte möglich.

### 5. Asymmetrische Verzögerungen machen die Regelung stabil

Nicht jede Änderung muss gleich schnell behandelt werden.

```text
Start 60 s
Hoch 60 s
Runter 30 s
Stop 120 s
```

ist bewusst asymmetrisch.

### 6. Ein bisschen Netzbezug kann Teil der Strategie sein

Eine Startschwelle unter 4,14 kW ist nicht automatisch falsch.

Hier ist sie bewusst erlaubt, damit schon bei kleinerer PV-Leistung geladen werden kann.

### 7. Das Auto entscheidet über die reale Leistungsaufnahme

Die Wallbox gibt nur eine Obergrenze vor.

Der BYD kann bei hohem SoC weniger aufnehmen.

### 8. App-Anzeigen sind nicht automatisch die Wahrheit

Wallbox, Shelly und physischer Zähler ergaben eine konsistente Energiebilanz, während die BYD-App zeitweise andere Werte zeigte.

### 9. Der physische Zähler ist ein wertvoller Plausibilitätscheck

Softwarewerte sollten bei Unsicherheit immer mit einer unabhängigen Messung gegengeprüft werden.

### 10. „Auto voll“ ist ein eigener Regelzustand

„Angeschlossen, aber 0 kW“ ist nicht dasselbe wie „wartet auf Sonne“.

Die 90-s-Ladeannahmeprüfung verhindert unnötige Startschleifen.

### 11. Logging gehört von Anfang an zur Regelung

Ohne Zeitstempel und Messwerte wird Fehlersuche schnell zum Raten.

Das Logbook hätte im Projekt mehrere Stunden Trial-and-Error sparen können, wenn es von Beginn an vorhanden gewesen wäre.

### 12. Dashboard und Package klar trennen

Beide sind YAML, aber für völlig unterschiedliche Home-Assistant-Bereiche.

Dateien sollten deshalb eindeutig benannt und im Header mit ihrem Zielpfad dokumentiert sein.

### 13. Neustartverhalten muss aktiv mitgedacht werden

Automationen funktionieren im normalen Betrieb anders als unmittelbar nach einem HA-Neustart.

Ein System ist erst robust, wenn auch Wiederanlauf und bestehende Zustände berücksichtigt werden.

### 14. Erst beobachten, dann schreiben

Bei Tuya-DPs zuerst Status und Verhalten erfassen.

Unbekannte Steuer-DPs nicht blind beschreiben.

### 15. Kleine Diagnose-Sensoren sparen große Debug-Zeit

Einige wenige transparente Zwischenwerte waren entscheidend:

```text
verfügbarer PV-Überschuss
Zielstrom
gesetzter Strom
AC-Leistung
Netzfluss
Ladeannahme-Timer
Vollerkennungs-Flag
```

Je klarer diese Werte sichtbar sind, desto schneller lässt sich die Regelung verstehen.

---

## Fazit

Die finale Lösung ist bewusst nicht auf maximale Komplexität optimiert, sondern auf **Nachvollziehbarkeit und Stabilität**.

Die wichtigsten Prinzipien sind:

```text
eine Regelinstanz
korrekte physikalische Messgrößen
dreiphasige Berechnung
ruhige Zeitlogik
flexibler Netzanteil
transparente Diagnose
sauberes Logging
explizite Vollerkennung
```

Damit ist die Wallbox-Regelung für den täglichen Betrieb deutlich robuster und gleichzeitig gut nachvollziehbar.
