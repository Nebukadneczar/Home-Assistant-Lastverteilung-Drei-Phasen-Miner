# Home Assistant – Lastverteilung Drei-Phasen-Miner

**Version: 1.1.0**

Blueprint für Home Assistant zur Begrenzung der elektrischen Last einer Miner-Phase auf maximal **9.000 W**.

Die Automation überwacht die Phase, an der der Miner hängt, und passt das Leistungsziel des Miners an. Die Prüfung erfolgt standardmäßig alle **5 Minuten**. Eine zusätzliche Sofortreaktion erfolgt, wenn die Phase die 9.000-W-Grenze überschreitet.

## v1.1.0 – Neue Features

### 🔄 Hysterese gegen Flattern
Neu in v1.1.0: Der Miner wird **erst wieder erhöht**, wenn die Phasenlast deutlich unter dem Grenzwert liegt (konfigurierbar, Standard: 500 W unter dem Limit). Das verhindert, dass das Target bei schwankenden Lasten (z.B. Waschmaschine + Miner) ständig auf- und ab springt.

```
Erhöhung erst wenn: Phase < (Limit - Reserve - Hysterese)
z.B.: 9000 - 100 - 500 = 8400 W
```

### 🛡️ Stabilere Ausführung
- `mode: single` statt `restart` – wartet bis der laufende Durchlauf fertig ist
- `max_exceeded: log` statt `silent` – unnötige Trigger werden geloggt, nicht still ignoriert

### 🎯 Präzise Trigger
- Nur noch Trigger auf der **Miner-Phase** (statt auf allen 3 Phasen)
- Weniger unnötige Automation-Ausführungen

## Funktionsprinzip

```
Phase A ─ aktuelle Leistung ─┐
Phase B ─ aktuelle Leistung ─┼─> Home Assistant ─> Miner Power Target
Phase C ─ aktuelle Leistung ─┘                                │
                                                               └─ maximal 9.000 W je Phase
```

Für den ausgewählten Miner wird die zulässige Leistung aus der Last seiner Phase berechnet.

```
Grundlast = Phasenleistung - aktuelles Miner-Leistungsziel

verfügbare Minerleistung =
    9.000 W - Grundlast - Sicherheitsreserve
```

Das Ergebnis wird anschließend auf den konfigurierten maximalen Minerwert begrenzt.

## Eigenschaften in v1.1.0

| Feature | v1.0.0 | v1.1.0 |
|---------|--------|--------|
| Überwachung Phase A, B, C | ✅ | ✅ |
| Miner-Phase wählbar | ✅ | ✅ |
| Max 9.000 W (konfigurierbar) | ✅ | ✅ |
| Sicherheitsreserve | ✅ | ✅ |
| Max. Minerleistung | ✅ | ✅ |
| Änderungsstufe | ✅ | ✅ |
| **Hysterese gegen Flattern** | ❌ | ✅ |
| **mode: single** | ❌ (restart) | ✅ |
| **max_exceeded: log** | ❌ (silent) | ✅ |
| **Nur Trigger auf Miner-Phase** | ❌ (alle 3) | ✅ |
| Automatische Anpassung | ✅ | ✅ |
| Regelintervall | ✅ | ✅ |
| Sofortreaktion bei Überlast | ✅ | ✅ |
| Wiederverwendbar als Blueprint | ✅ | ✅ |

## Wichtiger Hinweis

v1.1.0 regelt **einen ausgewählten Miner** anhand **seiner zugeordneten Phase**.

Die Phasenleistung wird als Gesamtleistung der jeweiligen Phase betrachtet. Für die Berechnung wird das aktuelle Miner-Power-Target als Anteil des Miners an der Phasenlast verwendet. Ein separater Ist-Leistungssensor des Miners ist nicht erforderlich.

Die Automation kann keine Last einer anderen Phase reduzieren, wenn der Miner nicht an dieser Phase angeschlossen ist. Eine echte Lastverteilung über mehrere Miner und mehrere Phasen ist als mögliche spätere Erweiterung vorgesehen.

## Beispiel

Konfiguration:

- Phase A: `sensor.phase_a_leistung`
- Phase B: `sensor.phase_b_leistung`
- Phase C: `sensor.phase_c_leistung`
- Miner hängt an Phase B
- Miner Power Target: `number.antminer36_power_target`
- maximale Minerleistung: 2.832 W
- Sicherheitsreserve: 100 W
- **Hysterese: 500 W**
- Änderungsstufe: 100 W
- Prüfintervall: 5 Minuten

### Szenario 1: Phase B bei 6.500 W

```
Grundlast = 6.500 W - 2.000 W (aktuelles Target) = 4.500 W
verfügbar = 9.000 W - 4.500 W - 100 W = 4.400 W
→ Miner kann auf 2.832 W (Max) erhöht werden
→ Aber: Hysterese? Phase (6.500 W) < 8.400 W? JA → Erhöhung erlaubt
```

### Szenario 2: Phase B bei 8.800 W (nähert sich Limit)

```
Grundlast = 8.800 W - 2.832 W = 5.968 W
verfügbar = 9.000 W - 5.968 W - 100 W = 2.932 W
→ Miner bleibt bei 2.832 W (unterschied < 100 W)
→ Warnung: "Phase nähert sich dem Grenzwert"
```

### Szenario 3: Phase B bei 9.200 W (über Limit)

```
SOFORTREAKTION!
Grundlast = 9.200 W - 2.832 W = 6.368 W
verfügbar = 9.000 W - 6.368 W - 100 W = 2.532 W
→ Miner wird sofort auf 2.500 W reduziert
→ Benachrichtigung: "Phase ÜBER dem Grenzwert"
```

### Szenario 4: Phase B fällt auf 8.500 W (nach Reduktion)

```
Hysterese-Limit = 9.000 - 100 - 500 = 8.400 W
Phase (8.500 W) > 8.400 W? JA → Erhöhung NICHT erlaubt
→ Miner bleibt reduziert, bis Phase unter 8.400 W fällt
```

## Installation

Die Blueprint-Datei liegt unter:

`blueprints/automation/miner/lastverteilung_drei_phasen.yaml`

In Home Assistant: **Einstellungen → Automatisierungen & Szenen → Blueprints → Blueprint importieren**.

## Konfiguration

| Parameter | Standard | Beschreibung |
|-----------|----------|--------------|
| Phase A/B/C Sensor | – | Leistungssensoren der Phasen |
| Miner-Phase | B | An welcher Phase hängt der Miner? |
| Miner Power Target | – | `number.*` Entity für das Target |
| Maximale Minerleistung | 2.832 W | Obergrenze für den Miner |
| Phasengrenzwert | 9.000 W | Max. zulässige Phasenlast |
| Sicherheitsreserve | 100 W | Reserve unter dem Limit |
| **Hysterese-Schwelle** | **500 W** | **Neu: Verhindert Flattern** |
| Änderungsstufe | 100 W | Schrittweite für Anpassungen |
| Prüfintervall | 5 Min. | Zeit zwischen Prüfungen |
| Benachrichtigungs-ID | – | ID für persistente Notifications |

## Changelog

### v1.1.0 (2026-11-05)
- **Added:** Hysterese-Schwelle gegen Flattern bei schwankenden Lasten
- **Changed:** `mode: single` statt `restart` für stabilere Ausführung
- **Changed:** `max_exceeded: log` statt `silent` für bessere Fehlersichtbarkeit
- **Changed:** Nur noch Trigger auf der Miner-Phase statt auf allen 3 Phasen
- **Improved:** Benachrichtigungen mit Emojis für bessere Übersicht

### v1.0.0
- Initial Release
- Grundlegende Drei-Phasen-Lastverteilung
- Konfigurierbare Grenzwerte und Reserven
- Sofortreaktion bei Überlast

## Repository

Dieses Repository ist bewusst getrennt vom Projekt **Home-Assistant**, damit beide Automationen unabhängig versioniert und gepflegt werden können.

Version **1.1.0** ist die stabilisierte Version mit Hysterese und verbesserter Trigger-Logik.
