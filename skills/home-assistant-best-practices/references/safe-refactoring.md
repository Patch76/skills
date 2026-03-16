# Safe Refactoring Workflow

Follow this workflow whenever you modify existing Home Assistant configuration: renaming entities, replacing template sensors with helpers, converting device triggers, or restructuring automations.

**Core rule:** Search all consumers BEFORE changing anything. Verify zero stale references AFTER.

---

## Universal Workflow

### Step 1: Identify the full scope of change

Answer three questions before touching anything:

1. **What changes?** Entity ID, automation structure, sensor type, or trigger semantics.
2. **What sibling entities share the same device?** Query the device to list every entity it owns (battery sensor, update entity, diagnostic button). Plan changes for all siblings together.
   - *Tool hint:* `ha_get_device(entity_id="...")` if available, or inspect Settings > Devices.
3. **Rename one entity or all device entities?** Devices bundle 2-6 entities. Renaming the primary but leaving siblings with the old naming scheme creates inconsistency.

### Step 2: Search ALL consumers

Search every component type that references entity IDs. Do not limit searches to the component you are editing.

| Component | How to search |
|-----------|---------------|
| Automations | `ha_deep_search(query="entity_id")` or grep `automations.yaml` |
| Dashboards | `ha_dashboard_find_card(entity_id="...")` or grep `.storage/lovelace*`, `ui-lovelace.yaml` |
| Scripts | grep `scripts.yaml` |
| Scenes | grep `scenes.yaml` |
| **Config-Entry data** | **grep `.storage/core.config_entries` — see §Config-Entry-Data below** |
| **Storage Dashboards** | **grep `.storage/lovelace.*` directly — ha_rename_entity does NOT update these** |
| Other | Check AppDaemon apps, Node-RED flows, Pyscript scripts, or any custom integration that references entity IDs |

Record every location found. This list becomes your update checklist for Step 4.

### Step 3: Make the change

Rename the entity, replace the template sensor, or restructure the automation.

### Step 4: Update every consumer

Work through each location from your Step 2 checklist. Update every reference to the new entity ID, helper entity, or automation structure.

### Step 5: Verify

1. **Search for the OLD identifier** across all component types. Expect zero results.
   - *Tool hint:* `ha_search_entities(query="old_name")` or grep all config files.
2. **Search for the NEW identifier** to confirm all expected locations reference it.
3. **Reload or check dashboards** if entity IDs changed.
4. **If stale references remain that you cannot update**, rename the entity back to its original ID to restore functionality, then report the blocking locations to the user.

---

## Entity Renames

Additional requirements beyond the universal workflow:

**Device-sibling discovery (Step 1):**
HA devices bundle multiple entities. A smart plug might expose `switch.*`, `sensor.*_energy`, and `update.*`. A multi-sensor exposes motion, temperature, illuminance, and battery entities. Rename all siblings to match.

Example — renaming a smart plug's entities from manufacturer defaults to room-based names:

| Domain | Old entity ID | New entity ID |
|---|---|---|
| switch | `switch.shellyplug_s_a1b2c3d4e5f6` | `switch.office_heater` |
| sensor | `sensor.shellyplug_s_a1b2c3d4e5f6_energy` | `sensor.office_heater_energy` |
| update | `update.shellyplug_s_a1b2c3d4e5f6` | `update.office_heater` |

**Dashboard reference locations (Step 2):**
Dashboard cards reference entities in multiple places. Search all of these:

- `entity:` field
- `tap_action` and `hold_action` targets
- Conditional card conditions
- Template card Jinja2 blocks

---

## Config-Entry Data — Blindes Terrain für ha_rename_entity

**`ha_rename_entity` aktualisiert ausschließlich die Entity-Registry.** Folgende Speicherorte bleiben unberührt und müssen manuell migriert werden:

### Storage-Mode-Dashboards (`.storage/lovelace.*`)

**Symptom:** Spook meldet „Unbekannte Entitäten verwendet in: [Dashboard-Name]" nach Migration.

**Ursache:** Lovelace-Storage-Dashboards halten entity_ids im `.storage`-Ordner. HA lädt diese **nur beim Start** ein — kein `reload_all`-Äquivalent existiert.

**Fix-Reihenfolge:**
1. Datei laden via `shell_command/read_file /config/.storage/lovelace.<dashboard_id>`
2. String-Replace aller alten → neuen IDs auf dem rohen JSON
3. Zurückschreiben via `write_file` mit Pfad `.storage/lovelace.<dashboard_id>`
4. **HA-Neustart** — zwingend, damit HA den neuen Storage einliest

**Größenbeschränkung:** `write_file` liefert HTTP 500 bei Dateien >~40KB. Lösung:
```python
json.dumps(data, separators=(',', ':'))  # komprimiert ~153KB → ~62KB
```

**`ha_config_set_dashboard(python_transform=...)` ist unbrauchbar für Bulk-Replace** — die Sandbox verbietet `import`, `def`, `isinstance` und `str.replace()`. Nur `split()`+`join()` als replace-Ersatz erlaubt, aber `isinstance` fehlt für rekursives Traversal.

### Config-Entry-`data`-Felder (`.storage/core.config_entries`)

**Betroffen:** Jede Integration, die im Setup-Flow entity_ids abfragt, speichert diese im `data`-Feld des Config-Entry — **nicht** in `options` und **nicht** in YAML.

**Bekannte betroffene Integrationen:**

| Integration | Felder in `data` mit entity_ids |
|---|---|
| **Better Thermostat** | `temperature_sensor`, `humidity_sensor`, `outdoor_sensor`, `window_sensors` |
| Generic Thermostat | `heater`, `target_sensor` |
| Generic Hygrostat | `humidifier`, `target_sensor` |
| Threshold Helper | `entity_id` |
| Min/Max Helper | `entity_ids` |

**Symptom:** Integration meldet nach Neustart „zugehörige Entität fehlt" (Better Thermostat) oder verhält sich falsch.

**Timing-Kritisch:** Config-Entry-`data`-Felder **vor dem HA-Neustart** korrigieren. Eine Integration, die mit veralteten entity_ids startet, kann den Neustart verlängern (BT wartet auf nicht-existente Entities).

**Scan-Snippet:**
```bash
# core.config_entries auf alte entity_ids prüfen
python3 -c "
import json
data = json.load(open('/config/.storage/core.config_entries'))
old_ids = ['old.entity_id_1', 'old.entity_id_2']  # eigene Liste
for entry in data['data']['entries']:
    text = json.dumps(entry.get('data', {}))
    hits = [o for o in old_ids if o in text]
    if hits:
        print(entry['domain'], entry['title'], hits)
"
```

**Fix via Options-Flow:**
```python
# Schritt 1: Flow initiieren
POST /api/config/config_entries/options/flow
body: {"handler": "<entry_id>"}

# Schritt 2: Formular mit neuen Werten abschicken
POST /api/config/config_entries/options/flow/<flow_id>
body: {<felder mit neuen entity_ids>}
```

**Better Thermostat spezifisch — mehrstufiger Flow:**
- Schritt 1 enthält: `temperature_sensor`, `humidity_sensor`, `outdoor_sensor`, `window_sensors`, `off_temperature`, `tolerance`, `target_temp_step`, `window_off_delay`, `window_off_delay_after`
- `weather`-Feld **weglassen** wenn nicht konfiguriert (nicht `null` übergeben → Validierungsfehler)
- `window_off_delay` als Dict: `{"hours": 0, "minutes": 2, "seconds": 0}`
- `target_temp_step` als String: `"0.0"`
- Schritt 2 enthält: `calibration`, `calibration_mode`, `protect_overheating`, `no_off_system_mode`, `heat_auto_swapped`, `child_lock`, `homematicip`

### Config-Entry-Groups (Gruppen-Member-IDs)

**Betroffen:** Alle Gruppen, deren Member-Entities umbenannt wurden (binary_sensor-Gruppen, cover-Gruppen, light-Gruppen).

**Symptom:** Spook „Unbekannte Gruppenmitglieder in: [Gruppenname]"

**Ursache:** Config-Entry-Groups speichern Member-IDs im Options-Feld. `ha_rename_entity` aktualisiert dieses Feld nicht.

**Fix:**
```python
# Option A: ha_config_set_group (für YAML-Gruppen oder group.set-Gruppen)
ha_config_set_group("rollogruppe", entities=["cover.neu_1", "cover.neu_2"])

# Option B: Options-Flow (für Config-Entry-Groups)
POST /api/config/config_entries/options/flow
body: {"handler": "<entry_id>"}
# → neue Member-IDs übergeben (Step 2)
POST /api/config/config_entries/options/flow/<flow_id>
body: {"entities": ["new.entity_1", "new.entity_2"], "hide_members": false, "all": false}
```

**Fallstricke beim Options-Flow für Gruppen:**

| Feld | Erlaubt | Verboten |
|---|---|---|
| `entities` | ✅ | |
| `hide_members` | ✅ | |
| `all` | ✅ (nur binary_sensor/sensor-Gruppen) | |
| `group_type` | ❌ → HTTP 400 | extra key not allowed |
| `name` | ❌ → HTTP 400 | extra key not allowed |

**Schema vorab prüfen:**
```python
# Erlaubte Felder live abfragen
GET /api/config/config_entries/options/flow/<flow_id>
# → data_schema zeigt exakt welche Felder akzeptiert werden
```

Cover-Gruppen haben kein `all`-Feld — weglassen. Binary-sensor-Gruppen haben es.

---

## Erweiterte Migrationscheckliste (Gesamtreihenfolge)

Für Migrationen mit >5 Entity-Renames:

```
1. YAML-Bulk-Replace
   → automations.yaml, scripts.yaml, template.yaml
   → Alle alten IDs als String ersetzen, auf Vollständigkeit prüfen

2. core.config_entries scannen und patchen
   → Auf alle alten IDs prüfen (Scan-Snippet oben)
   → Betroffene Config-Entries via Options-Flow aktualisieren
   → Timing: VOR dem nächsten HA-Neustart

3. Registry-Renames (ha_rename_entity)
   → Alle Domains nacheinander

4. Config-Entry-Groups reparieren
   → binary_sensor-, cover-, light-Gruppen prüfen

5. Storage-Mode-Dashboards reparieren
   → read_file → Bulk-Replace → write_file (komprimiert!)

6. HA-Neustart
   → Pflicht nach Storage-Dashboard-Writes

7. Spook-Issues verifizieren
   → sensor.active_issues sollte 0 sein
   → Spook-Scan ist asynchron: 1-2 Minuten nach Neustart warten
```

---

## Helper Replacements

When replacing a template sensor with a built-in helper (`min_max`, `threshold`, `derivative`):

**New entity ID (Step 1):**
The helper creates a new entity with a different entity_id. The old template sensor's entity_id stops existing. Update every consumer of the old entity_id to reference the new one.

**Test equivalence (Step 5):**
Verify the new helper produces the same values as the old template sensor. Check units, precision, and unavailable-state handling.

---

## Trigger Restructuring

When converting `device_id` triggers to `entity_id` triggers, or replacing `wait_template` with `wait_for_trigger`:

**Behavioral equivalence (Step 1):**
`wait_for_trigger` waits for a state *change*; `wait_template` polls for *current state*. These differ when the target state is already true at wait start: `wait_for_trigger` blocks indefinitely, `wait_template` returns immediately.

**Automation callers (Step 2):**
Search for scripts or other automations that call the automation you are restructuring via `automation.trigger` or `automation.turn_on`. Renaming or splitting an automation changes its entity_id and breaks these callers.

---

## Config-Entry-Groups

When renaming entities that are members of a HA **group** created via the UI (Config-Entry-based group, platform: `group`):

**`ha_rename_entity` does NOT update group members automatically.**

Group member entity IDs are stored in `options.entities` of the group's Config Entry — not in the entity registry. A registry rename leaves the group referencing the old (now non-existent) entity ID, silently breaking it.

**Detection (Step 2):**
Check all Config-Entry-based groups for membership:

```bash
# Via ha_get_integration(domain="group") — inspect options.entities for old entity IDs
ha_get_integration(domain="group", include_options=True)
```

**Fix (Step 4):**
After the registry rename, update group membership via the Options Flow:

```bash
POST /api/config/config_entries/options/flow
{"handler": "<group_config_entry_id>"}

# Then PATCH the flow step with new member entity IDs:
POST /api/config/config_entries/options/flow/<flow_id>
{"entities": ["new.entity_id_1", "new.entity_id_2"]}
```

**Verify (Step 5):**
Re-fetch the group config entry and confirm `options.entities` contains only new entity IDs.