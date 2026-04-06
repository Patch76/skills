## Helper Configuration Examples

This reference provides practical examples for common helper operations.

**Preferred approach:** Use `ha_set_config_entry_helper` (UI/API) to create and update helpers. This avoids manual file edits and domain reloads. YAML-based configuration is a fallback for helpers not supported by the Config Flow.

### input_select

Use `input_select` for dropdown selection with a fixed list of options.

Key notes: options list must be non-empty; initial value must match one of the options, etc.

| Parameter | Type | Notes |
|---|---|---|
| name | string | Required. Display name shown in the UI etc. |
| options | list | Required. At least one option must be provided |
| initial | string | Optional. Must match one of the options |
| icon | string | Optional. MDI icon identifier |

**Preferred:** Create via `ha_set_config_entry_helper` — no reload required.

**YAML fallback** (only if Config Flow is unavailable):

```yaml
# Example: creating an input_select helper via YAML
input_select:
  scene_mode:
    name: Scene Mode
    options:
      - Normal
      - Film
      - Night
    initial: Normal
```

After writing YAML, call `input_select.reload` to apply changes.

### input_number

Use `input_number` for numeric values with optional min/max bounds.

Key notes: min and max are required; step defaults to 1; mode is either box or slider.

**Preferred:** Create and update via `ha_set_config_entry_helper`. Config Entry-managed helpers can also be updated via the `input_number/update` WebSocket message type.

**Note:** `input_number/update` only applies to helpers managed via Config Entries (UI/API). It cannot update helpers defined in YAML. To update a YAML-defined helper, edit the YAML file and reload the domain.

| Parameter | Type | Notes |
|---|---|---|
| name | string | Required |
| min | float | Required. Lower bound |
| max | float | Required. Upper bound |
| step | float | Optional. Defaults to 1 |
| mode | string | Optional. box or slider, etc. |

### counter

Counters track integer values and support increment, decrement, and reset operations.

Key notes: minimum and maximum are optional; restore defaults to true.

```yaml
counter:
  motion_count:
    name: Motion Counter
    initial: 0
    step: 1
    restore: true
```
