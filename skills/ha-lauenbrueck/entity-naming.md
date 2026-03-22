# Entity Naming Convention — LB (Lauenbrück)

> **Scope:** LB-Installation only. Opinionated user-space convention —
> nicht für homeassistant-ai/skills geeignet (kein offizielles HA-Verhalten).
> Gehört in persönlichen Skill, nicht upstream.

---

## Format

```
<domain>.<location>_<device>_<measurement>
```

**Korrekt:**
- `sensor.kueche_steckdose_spuelmaschine_power`
- `sensor.wohnzimmer_shelly_temperatur`
- `switch.schlafzimmer_licht`
- `binary_sensor.buero_fenster_kontakt`

**Falsch:**

| Beispiel | Fehler |
|---|---|
| `sensor.kitchen_dishwasher_power` | Englisch statt Deutsch |
| `sensor.kitchen_power_dishwasher` | Measurement vor Device — Typ-vor-Gerät-Regel verletzt |
| `sensor.sensor_wohnzimmer_temp` | Domain-Redundanz im Namen |
| `sensor.shelly_wohnzimmer_temp` | Brand zuerst statt Location |

---

## Räume-Glossar (LB-kanonisch)

| Raum | Entity-ID-Kürzel |
|---|---|
| Wohnzimmer | `wohnzimmer` |
| Schlafzimmer | `schlafzimmer` |
| Badezimmer | `badezimmer` |
| Küche | `kueche` |
| Schuppen | `schuppen` |
| Garten | `garten` |
| Flur | `flur` |
| Büro | `buero` |

---

## Scripts

```
script.[<location>_]<function>
```

Location optional — nur wenn Funktion raumspezifisch:
- `script.wohnzimmer_licht_dimmen`
- `script.nachtmodus_aktivieren`

---

## Automationen

```
automation.<bereich>_<ereignis>
```

- `automation.kueche_wandschalter_led`
- `automation.abwesenheit_aktivieren`
- `automation.global_abwesenheit_deaktivieren`

---

## Helfer (Helper)

```
input_boolean.<kontext>_<funktion>
input_select.<kontext>_<funktion>
```

- `input_boolean.abwesenheit_aktiv`
- `input_boolean.nachtmodus`
- `input_select.heizung_preset`

---

## Allgemeine Regeln

- Entity-IDs: ausschließlich **lowercase + Unterstriche**, keine Bindestriche
- Friendly names: **Deutsch**, Großschreibung wie im Deutschen üblich
- Kein Brand/Hersteller im Entity-ID-Präfix (`shelly_`, `aqara_`, `sonoff_` etc. vermeiden)
- Einheitliche Measurement-Suffixe: `_power`, `_energy`, `_temperatur`, `_feuchte`, `_lux`, `_co2`

---

## Umbenennung bestehender Entities

Vor jeder Umbenennung: `references/safe-refactoring.md`-Workflow (→ best-practices Skill):
- Config-Entry-Data prüfen (Better Thermostat, Generic Thermostat, Min/Max, Threshold)
- Storage-Mode-Dashboards patchen (WebSocket API, kein Restart nötig)
- Automationen, Scripts, Szenen scannen

Verifiziert LB 21.03.2026.
