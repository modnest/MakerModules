# Schaltungsentwurf: Ausgangsmodul 1,8 V / 1 A (CC/CV)

Stand: 12.11.2026 · Projekt „Stromversorgung" (modulares USB-PD-Netzteil)

Dieses Dokument fasst den Entwurfsstand des 1,8-V-Ausgangsmoduls zusammen. Die Werte des Linearreglers sind in ngspice simuliert (TI-Makromodell OPA4388, Verhaltensmodell AON7407). Ein LTspice-Lauf der aktuellen Fassung steht noch aus. Bauteilbezeichner im Regelteil (R1318, R_PD usw.) sind die Namen aus der Simulation, nicht die KiCad-Referenzen.

---

## 1. Anforderungen

| Punkt | Vorgabe |
|---|---|
| Ausgangsspannung | 1,8 V fest (CV) |
| Ausgangsstrom | max. 1 A |
| Stromlimit | per 12-Bit-DAC (MCP4728) von der MCU einstellbar, Auflösung mind. 10 mA, besser erwünscht |
| Betriebsarten | CV, solange die Last unter dem Limit liegt; sonst schnelle Begrenzung (CC) |
| Eingang | VBUS 5–48 V je nach USB-PD-Contract, vorgeschalteter Buck auf ca. 2,0 V |
| Überwachung | INA238 am 10-mΩ-Shunt; LM339 für CC_ACTIVE, VOUT_LOW (< 80 % Sollwert) und ALERT-Latch |
| Abschaltung | Overload (VOUT_LOW) nach Wartezeit, INA-Überstrom innerhalb 10 ms; dauerkurzschlussfest wünschenswert |
| Einsatz | Versorgung von FPGA-/MCU-Prototypen mit 1,8 V Vcc |
| Aufbau | Steckmodul (8 Slots), Mainboard mit STM32G0B1VCT6, zwei MCP4728T für je 4 Slots |
| Bezug | Bauteile nach Möglichkeit von JLC/LCSC |
| Nebenbedingung | keine Hilfsspannung über 5 V (bei 10-W-PD-Netzteil nur 5 V VBUS) |

---

## 2. Blockschaltbild

```
 VBUS 5…48 V                         Modul-Leistungspfad
 ──────────► Buck LM706A0 ──► VIN ≈ 2,0 V ──[R_Shunt 10 mΩ]──► VOUT ──► AON7407 ──► VOUTSW 1,8 V ──► Klemme
                ▲  FB                │                   │        (Source)  (Drain)   │
                └────────────────────┼───────────────────┘                  ▲        ├─ C_SW 47 µF
                     (FB-Abgriff hinter dem Shunt)                          │ Gate   └─ R_BLEED 180 Ω
                                     │                                      │
                                     ├──► INA238 ── I²C / ALERT ──► MCU      │ 10 Ω
                                     ▼                                      │
              ┌───────────── A: Differenzverstärker 150×, +0,1 V Offset ────┼──────────────┐
              │                    AOUT = 0,1 V + 1,5 V/A · I               │              │
              │                                │                           GCTL ──470 Ω── GND
              │                                ▼                            ▲
   V_ISET ────┼──────────► B: CC-Integrator ─────────── BAT54C (Dioden-ODER) ┤
   (DAC)      │                                                             │
              │   REF3025 2,5 V ─► Teiler ─► C: Puffer 1,8 V ─► D: CV-PI-Regler ◄── VOUTSW
              │
              └──► LM339: CC_ACTIVE · VOUT_LOW · ALERT-Latch ──► MCU
```

- Beide Regler können das Gate nur **nach oben** ziehen (FET zu). Es gewinnt der Regler, der weniger Strom verlangt.
- Ohne Eingriff zieht R_PD das Gate auf GND, der FET ist dann voll durchgesteuert.
- Der OPA4388 hat vier Kanäle: A (Shunt-Verstärker), B (CC), C (Sollwert-Puffer), D (CV).

---

## 3. Gewählte Bauteile (aktiv)

| Ref | Funktion | Typ | Gehäuse | JLC/LCSC | Anmerkung |
|---|---|---|---|---|---|
| Q702 | Längs-FET | AON7407 (AOS) | DFN 3×3, 8 Pins + Pad | C176756 | Nachbau AON7407-ES (C19725104) nicht verwenden |
| U703 | Regler, 4 Kanäle | OPA4388IPWR (TI) | TSSOP-14 | C2058586 | Tube-Variante OPA4388IPW: C1346834; UMW-Nachbau C22368271 nicht verwenden |
| neu | Referenz 2,5 V | REF3025AIDBZR (TI) | SOT-23-3 | C11334 | eine pro Modul |
| U701 | Strommessung | INA238AIDGSR (TI) | VSSOP-10 | C2868250 | |
| U704 | Komparatoren | LM339DR (TI) | SOIC-14 | C7948 | |
| neu | Dioden-ODER Gate | BAT54C | SOT-23 | C2135 | gemeinsame Kathode an GCTL |
| neu | Klemmdiode CV-Regler | 1N4148WS | SOD-323 | C2128 | zwischen den Eingängen von Kanal D |
| U702 | Buck | LM706A0 (TI) | – | aus vorhandenem Plan | Spule L701 15 µH, eigener Shunt R722 39 mΩ |

**Entfallen gegenüber dem ursprünglichen Plan:** OPA2197, Si2365EDS, Treibertransistor MMBT2222A (Q701) mit R702–R706.

---

## 4. Dimensionierung des Linearreglers

### 4.1 Werte

| Block | Bauteil | Wert | Toleranz / Hinweis |
|---|---|---|---|
| Shunt | R_Shunt (R713) | 10 mΩ | 1 % |
| A: Shunt-Verstärker | R1313, R1314 | 1 kΩ | **0,1 %** |
| | R1316, R1317 | 150 kΩ | **0,1 %** |
| | Offset-Teiler R_O1 / R_O2 (aus VREF) | 2,4 kΩ / 100 Ω | 1 %, Fußpunkt von R1316 |
| B: CC-Integrator | R1318 (V_ISET → −IN) | 10 kΩ | |
| | C1303 (−IN → GCTL) | 15 nF | C0G/X7R |
| | R1320 (AOUT → +IN) | 10 kΩ | |
| | R_LG + C_LG (+IN → GND) | 1,1 kΩ + 4,7 nF | Filterglied im Istwertpfad |
| C: Sollwert-Puffer | Teiler aus 2,5 V | 7 kΩ / 18 kΩ (simuliert) | real: **6,98 kΩ / 18,0 kΩ**, 0,1 % |
| D: CV-Regler | R1321 (VOUTSW → +IN) | 100 Ω | |
| | R18C (Sollwert → −IN) | 10 kΩ | |
| | R19C + C_F (−IN → GCTL) | 20 kΩ + 10 nF | |
| | Klemmdiode zwischen −IN und +IN | 1N4148WS | Anode an −IN |
| Gate | R_Gate (GCTL → Gate) | 10 Ω | |
| | R_PD (GCTL → GND) | 470 Ω | |
| Ausgang | C_SW | **47 µF effektiv** | nach DC-Bias-Derating; ESR einige mΩ |
| | R_BLEED | 180 Ω | Grundlast 10 mA |
| Versorgung | OPA4388 | 5 V (VCC_CTRL) | 100 nF + 10 µF |

### 4.2 Shunt und Strommessung

- Spannungsabfall am Shunt bei 1 A: 10 mV, Verlust 10 mW.
- Ausgang Kanal A: AOUT = U_Offset + 150 · I · 10 mΩ = **0,1 V + 1,5 V/A · I**
  - bei 1 A: 1,60 V, bei 1,3 A: 2,05 V
- Offset: 2,5 V · 100 / (2400 + 100) = **0,100 V**. Der OpAmp-Ausgang bleibt damit auch bei 0 A von der unteren Schiene weg.
- Gleichtaktspannung am Verstärker ≈ 2,0 V. Bei 0,1 % Widerstandstoleranz beträgt der Gleichtaktfehler im ungünstigsten Fall etwa 4 · 0,001 · 2,0 V = 8 mV am Ausgang, also rund **5 mA**. Der Innenwiderstand des Offset-Teilers (96 Ω) erhöht die Unsymmetrie zu R1317 um 0,06 % (≈ 1 mA). Beide Fehler sind konstant, weil VIN geregelt ist, und lassen sich per Firmware abgleichen.
- Offset OPA4388: max. 5 µV am Shunt → 0,5 mA.

### 4.3 Stromsollwert (DAC)

- MCP4728 mit interner Referenz 2,048 V, Verstärkung 1 → 0,5 mV pro Schritt.
- **V_ISET = 0,1 V + 1,5 V/A · ILIM**, DAC-Code = 200 + 3000 · ILIM[A]
  - 1 A → Code 3200; Endwert Code 4095 → **1,30 A**
  - Auflösung: **0,33 mA pro Schritt**
  - DAC = 0 V → V_ISET liegt unter dem Offset, der FET sperrt sicher (definierter Aus-Zustand, auch bei MCU-Reset).
- Genauigkeit der internen Referenz: 2,007–2,089 V (±2 %), ca. 45 ppm/K → unkalibriert bis ±20 mA bei 1 A. Abgleich über den INA238 vorgesehen.
- Die Grundlast R_BLEED (10 mA) zählt zum gemessenen und begrenzten Strom.

### 4.4 Regler-Zeitkonstanten

| Regler | Größe | Wert |
|---|---|---|
| CC (B) | Integrator R1318 · C1303 | 150 µs |
| | Filterglied im Istwertpfad | Pol ≈ 3,0 kHz (11,1 kΩ · 4,7 nF), Nullstelle ≈ 31 kHz (1,1 kΩ · 4,7 nF) |
| CV (D) | Proportionalanteil R19C / R18C | 2 |
| | Integrator R18C · C_F | 100 µs |
| | PI-Nullstelle 1 / (2π · R19C · C_F) | ≈ 0,8 kHz |

### 4.5 Spannungssollwert und Referenz

- Sollwert 2,5 V · 18,0 k / (6,98 k + 18,0 k) = **1,801 V** (simuliert mit 7 k / 18 k = 1,800 V; 7 kΩ ist kein Normwert).
- Fehlerbudget: REF3025 ±0,2 % (±3,6 mV am Ausgang), Teiler 0,1 % (≈ ±1,5 mV), Drift max. 75 ppm/K → zusammen etwa **±7 mV** (±0,4 %).
- Last der Referenz: Sollwertteiler 0,1 mA + Offset-Teiler 1,0 mA ≈ 1,1 mA (REF3025 liefert bis 25 mA).

### 4.6 Gate-Ansteuerung

Benötigte Source-Gate-Spannung des AON7407 im Begrenzungsbetrieb (Modell, typisches Exemplar, 25 °C, Vds ≈ 2 V):

| Strom | Vsg | Steilheit |
|---|---|---|
| 1 mA | 0,62 V | 0,02 S |
| 10 mA | 0,73 V | 0,2 S |
| 100 mA | 0,85 V | 2 S |
| 500 mA | 0,92 V | 15 S |
| 1 A | 0,95 V | 20 S |

- Schwellstreuung laut Datenblatt 0,3–0,9 V → Gate-Spannung bei 1 A und 2,0 V Buck zwischen ca. 0,7 und 1,3 V. Für den OPA4388 unkritisch.
- Strom aus dem OpAmp in R_PD: bei 1,0–1,3 V Gate-Spannung ca. 2–3 mA.
- Voll durchgesteuert (Gate ≈ 0 V, Vsg ≈ 2 V): RDS(on) ≈ 11 mΩ → 11 mV Abfall bei 1 A.

### 4.7 Verlustleistung und Thermik (AON7407)

Datenblatt: RthJA 30–40 K/W für t ≤ 10 s, 60–75 K/W im Dauerbetrieb (Montage laut Datenblatt).

| Betriebsfall (Buck 2,0 V) | Vds | Verlust | Erwärmung |
|---|---|---|---|
| CV, 1 A | 0,2 V | 0,2 W | +12…15 K |
| CC, Ausgang knapp über 80 % (1,44 V), 1 A | 0,56 V | 0,56 W | +34…42 K |
| Kurzschluss 1 A, bis 10 s | ≈ 1,9 V | 1,9 W (sim.) | +60…80 K |
| Kurzschluss 1 A, dauerhaft | ≈ 1,9 V | 1,9 W | +115…145 K → **nicht dauerfest** |
| nach Foldback auf 0,25 A, dauerhaft | ≈ 2,0 V | 0,5 W | +30…38 K |
| Kurzschluss bei Buck-Maximum 2,05 V | 2,05 V | 2,05 W | |

- Überlast an 1,5 Ω (1 A): 0,5 W.
- Stromspitze beim harten Kurzschluss: 14–19 A für ca. 10 µs (≈ 0,3 mJ), solange der CV-Regler noch nicht abgegeben hat.
- Weitere Verluste: Shunt 10 mW, R_BLEED 18 mW.

### 4.8 Masseversatz über den Modulstecker

Rückstrom über die drei Massekontakte = Eingangsstrom des Bucks. Angenommen 10 mΩ für Kontakte und Leiterbahn (nicht gemessen).

| Modul | Fall | Massestrom | Versatz | Fehler Stromlimit |
|---|---|---|---|---|
| 1,8 V / 1 A | VBUS 48 V | ≈ 0,07 A | 0,7 mV | ≈ 0,5 mA |
| 1,8 V / 1 A | VBUS 5 V | ≈ 0,7 A | 7 mV | ≈ 5 mA |
| 3,3 V / 4 A | VBUS 9 V | ≈ 1,8 A | 18 mV | ≈ 50 mA |
| 24 V / 3 A | VBUS 20 V | ≈ 4 A | 40 mV | ≈ 80 mA |

Der Fehler geht in die sichere Richtung (Modul begrenzt früher).

---

## 5. Simulationsergebnisse (ngspice)

Lastprofil: Leerlauf mit 10 mA Grundlast → 0,5 A (2–3 ms) → Kurzschluss 0,1 Ω (4–6 ms) → 1,5 Ω (7–9 ms), ILIM = 1 A.

| Größe | Steife 2,0-V-Quelle | Mit Buck-Modell |
|---|---|---|
| Ausgang im Leerlauf | 1,808 V | 1,797 V |
| Einbruch bei Sprung auf 0,5 A | 1,744 V | 1,684 V |
| Überschwingen nach Lastabwurf 0,5 A | 1,849 V | 1,926 V |
| Strom im Kurzschluss | 0,9987 A | 1,0031 A |
| Stromspitze Kurzschluss | 19 A | 14 A |
| FET-Verlust im Kurzschluss | 1,90 W | 1,90 W |

Weitere Fälle (steife Quelle):

| Variante | Ergebnis |
|---|---|
| FET-Schwelle min./max. | praktisch keine Änderung |
| ILIM 20 mA | 19,0 mA |
| ILIM 100 mA | 99,0 mA an 1,5 Ω; im Kurzschluss 96,2 mA mit 4,9 mA Welligkeit |
| ILIM 600 mA | 598,6 mA |
| C_SW 22 µF | Restwelligkeit ≈ 10 mV bzw. 9 mA → zu klein |
| C_SW 47 µF … 1000 µF | stabil; bei 1000 µF verlängert sich der Anlauf (Laden mit ILIM) |

Das Buck-Modell ist ein gemittelter PI-Regler (≈ 3,5 kHz Durchtritt, 66 µF, Stromgrenze 1,4 A als Annahme). Es schwingt nach Lastabwurf bis 2,55 V über und ist vermutlich zu pessimistisch.

Dateien: `CCCV_1V8.asc` (LTspice), `CCCV_1V8_ngspice.cir` (geprüfte Netzliste), `AON7407_behav.lib` (FET-Modell), `OPA4388.LIB` (TI).

---

## 6. Entscheidungen und Begründung

| Nr. | Entscheidung | Begründung |
|---|---|---|
| E1 | Buck vor dem Linearregler, Ausgang ≈ 2,0 V, FB hinter dem Shunt | Verlust im FET klein halten; 0,2 V Reserve für Buck-Toleranz (±2,48 %) und Einbrüche bei Lastsprüngen |
| E2 | P-Kanal-FET statt N-Kanal-Sourcefolger | keine Hilfsspannung über 5 V; bei 10-W-PD-Netzteil liegen nur 5 V an |
| E3 | AON7407 statt Si2365EDS | SOT-23 thermisch zu schwach (bis 166 K/W, 2 W im Kurzschluss nicht haltbar); DFN 3×3 mit Pad; gut verfügbar (FDMC510P nur geringe Stückzahl) |
| E4 | OPA4388 statt OPA2197 | 5-V-Betrieb am Rand der OPA2197-Spezifikation; Eingangs-Übergangsbereich des OPA2197 (2,0–3,5 V bei 5 V) liegt genau an der Gleichtaktspannung; Offset 5 µV statt 100 µV |
| E5 | TI-Originale (OPA4388, AON7407 von AOS) | Nachbauten mit eigenem, ungeprüftem Datenblatt; SPICE-Modelle gelten nur fürs Original |
| E6 | Gate direkt vom OpAmp, kein Treibertransistor | Ergebnis der Simulationsreihe: NPN-Treiber mit Miller-Kompensation war entweder instabil oder zu langsam (ms statt µs) |
| E7 | Kein Inverter-Kanal | Polarität über die Eingangszuordnung (Istwert an +IN, Sollwert an −IN); invertierter DAC-Code entfällt |
| E8 | Dioden-ODER (BAT54C) + R_PD | beide Regler arbeiten auf denselben Knoten; der nicht aktive Regler sperrt seine Diode (Anti-Windup); Übergang CV↔CC ohne Umschaltlogik |
| E9 | 0,1 V Offset am Shunt-Verstärker | OpAmp-Ausgang nie an der unteren Schiene; DAC = 0 V schaltet sicher ab |
| E10 | Sense-Verstärkung 1,5 V/A | passt zur internen 2,048-V-Referenz des MCP4728: 1,30 A Endwert, 0,33 mA/Schritt |
| E11 | MCP4728 mit interner Referenz statt VDD | 3,3-V-MCU-Versorgung ungenauer und gestört; Restfehler ±2 % per INA238 abgleichen |
| E12 | REF3025 auf jedem Modul | zentrale Referenz würde lastabhängigen Masseversatz des Steckers übernehmen; Modul bleibt eigenständig testbar |
| E13 | C_SW ≥ 47 µF effektiv + 180 Ω Grundlast | CV-Schleife braucht den Ausgangspol; mit 22 µF Restwelligkeit; Grundlast definiert den Arbeitspunkt im Leerlauf |
| E14 | Ausgangsklemmen und Ausgangsmasse auf dem Modul | Laststrom fließt nicht über den Stecker; nur Buck-Eingangsstrom erzeugt Masseversatz |
| E15 | Abschaltstrategie: 100 ms tolerieren, dann Foldback auf ca. 25 %, periodischer Wiederanlauf (oder Abschalten nach 1 s) | Einschaltströme kapazitiver Lasten zulassen (1 A lädt 10 000 µF in ≈ 18 ms); Kurzschluss im SOT/DFN thermisch nur kurz haltbar; damit dauerkurzschlussfest |
| E16 | Modulerkennung per ADC-Abfrage statt Interrupt | Teilerwerte bewusst weit gespreizt; ab Code 4 kein sicherer Logikpegel für einen Interrupt (Schwelle STM32G0 ≈ 0,3 · VDD) |

---

## 7. Steckerbelegung (LB225-Y20-B0R03, Sicht Mainboard)

| Pin | Signal | Pin | Signal |
|---|---|---|---|
| 1, 3, 5 | GND | 2, 4, 6 | VBUS |
| 7 | 5 V | 8 | 3,3 V |
| 9 | SCL | 10 | SDA |
| 11 | A0 (INA-Adresse) | 12 | A1 (INA-Adresse) |
| 13 | CC_ACTIVE | 14 | ENABLE (Buck) |
| 15 | VOUT_LOW | 16 | ALERT (INA) |
| 17 | Modultyp (Spannungsteiler) | 18 | ISET (DAC) |
| 19 | EN Boost (nur 24-V-Modul) | 20 | PRESENCE (geplant: Bezugsmasse ISET) |

Kontaktbelastbarkeit 2 A je Kontakt → 6 A für VBUS bzw. GND. Ungünstigster Fall 24-V-Modul an 20 V: ≈ 4 A (≈ 1,3 A je Kontakt).

Modultyp-Codierung (10 kΩ Pull-up an 3,3 V, nachgerechnet):

| Code | R_Modul | U_ADC |
|---|---|---|
| 0 | offen | 3,30 V |
| 1 | 1,0 kΩ | 0,30 V |
| 2 | 2,2 kΩ | 0,60 V |
| 3 | 3,9 kΩ | 0,93 V |
| 4 | 6,8 kΩ | **1,34 V** (in alter Tabelle 1,47 V) |
| 5 | 10 kΩ | 1,65 V |
| 6 | 15 kΩ | 1,98 V |
| 7 | 22 kΩ | 2,27 V (in alter Tabelle 2,34 V) |
| 8–11 | 33 / 47 / 68 / 100 kΩ | 2,53 / 2,72 / 2,88 / 3,00 V |

Das 1,8-V-Modul trägt Code 1 (R748 = 1 kΩ). Firmware-Schwellen auf die Mitten der gerechneten Werte legen.

---

## 8. Offene Punkte

### Hardware

1. **Buck-Ausgangsspannung:** FB-Teiler im Plan 13 k / 10 k. Bei 0,8 V Referenz des LM706A0 ergibt das 1,84 V statt 2,0 V. Referenzwert im Datenblatt prüfen, ggf. 15 k / 10 k.
2. **Buck-Ausgangskapazität:** Plan zeigt C701 = 100 µF, zuvor genannt 3 × 22 µF. Festlegen und Simulationsmodell anpassen.
3. **Buck-Stromgrenze und Lastsprungverhalten:** Stromgrenze im Modell (1,4 A) ist eine Annahme. WEBENCH-Transientenkurve fehlt (WEBENCH nennt 36 mV, Modell zeigt bis 0,55 V Überschwingen).
4. **Überschwingen am Ausgang:** mit Buck-Modell bis 2,02 V nach Kurzschlussende. Grenzwerte der Ziel-FPGAs prüfen (oft ≈ 2,0 V absolut).
5. **LM339-Beschaltung neu auslegen:**
   - CC_ACTIVE: neue Quelle nötig (bisheriges CTRL entfällt). Vorschlag: Ausgang Kanal B gegen feste Schwelle (CV ≈ 0 V, CC ≈ 1,4 V).
   - VOUT_LOW: Schwelle 80 % = 1,44 V.
   - ALERT-Latch übernehmen.
6. **Pin 20:** PRESENCE als Bezugsmasse für ISET umwidmen (Empfehlung, noch nicht bestätigt). Für das 1-A-Modul kann Pin 20 an der Modulmasse liegen; nötig vor allem für Module ab 3 A.
7. **RC-Filter am ISET-Eingang:** ca. 1 kΩ / 100 nF vorsehen.
8. **Ausgangsschutz:** TVS-Typ wählen; Rückstrom über die Body-Diode bei Fremdspannung am Ausgang (zweiter FET oder Ideal-Dioden-Baustein, falls relevant).
9. **Fail-safe:** Fällt die 5-V-Versorgung des OpAmps aus, zieht R_PD das Gate auf GND → FET voll an. Abhilfe prüfen (z. B. Buck-ENABLE an 5 V koppeln).
10. **Pulsfestigkeit:** Kurzschlussspitze 14–19 A / 10 µs gegen das SOA-Diagramm des AON7407 prüfen.
11. **Thermik:** Kupferfläche unter dem FET-Pad festlegen; RthJA des eigenen Layouts weicht vom Datenblatt ab.
12. **Steckerbelegung Modulseite:** im Modul-Schaltplan sind gerade/ungerade Pins gegenüber der Mainboard-Liste durchgehend vertauscht – vermutlich gewollte Spiegelung, gegen die Footprints prüfen.
13. **Widerstandsnormwerte:** 7 kΩ (Simulation) durch 6,98 kΩ ersetzen; 0,1 %-Typen für Shunt-Verstärker und Sollwertteiler bei LCSC auswählen.

### Simulation / Modelle

14. **LTspice-Lauf** der aktuellen Fassung (`CCCV_1V8.asc`) steht aus.
15. **AON7407-Modell:** unter 100 mA extrapoliert, Temperaturgang nicht angepasst, Cgd konstant 3 nF.
16. **Kleiner Strom im Kurzschluss:** bei ILIM = 100 mA −3,8 mA Abweichung mit 4,9 mA Welligkeit (geringe FET-Steilheit). Bei 10 mA Anforderung unkritisch, für 1-mA-Auflösung prüfen.

### Firmware

17. DAC-Formel: Code = 200 + 3000 · ILIM[A]; Abgleich von Offset und Skalenfehler über den INA238 (Nachregelung bei aktivem CC_ACTIVE).
18. Abschaltstrategie (E15) umsetzen: Wartezeit, Foldback, Wiederanlauf.
19. Startverhalten: mit kleinem ILIM (100–300 mA) hochfahren, Lastkapazität aus dU/dt abschätzen, ab ca. 470 µF warnen oder abschalten.
20. Modulerkennung: ADC-Scan alle 10–50 ms per Timer/DMA, 2–3 gleiche Ergebnisse vor Übernahme; Ziehen zusätzlich über fehlende INA-Antwort erkennen.
